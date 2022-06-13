---
title: 将 linux 0.11 进程切换由基于 tss 改写为基于内核栈
date: 2022-06-13 19:00
---

来自哈工大李治军老师的操作系统网课 [link](https://www.icourse163.org/learn/HIT-1002531008) 实验四。

具体是 linux 0.11 使用 CPU 提供的 TSS 进行进程的切换，虽然使用这个机制能很方便地进行进程切换，但是效率不太高，于是要求我们在课程的理解上将其改写为基于内核栈（kernel stack）的进程切换。

#### 写在前面的废话

我的理解能力有限，外加周围没有人能指导，摸爬滚打查阅了数天的资料，理解了其中大致的原理，故才有了下面这篇文章。我认为只是看看网文，大致了解个思路然后心里想着“哦好像是这样”，然后草草了然进入下一课是不行的。你得了解核心代码每一行做了什么，为什么这样做，反复问自己。

#### 前置知识

下面写一些前置知识，了解的可以当做复习，不了解的必须了解啊，有助于下文的理解。

##### 中断做了什么

李老师在[第五课](https://www.bilibili.com/video/BV1d4411v7u7?p=5)中借用系统调用的实现顺便讲了讲中断是怎么做的，简单来说，就是用 `int` 指令发出中断信号，系统根据 `int` 后的操作数选择提前注册好的中断处理函数进行中断处理。举个例，`int 0x80` 指令发出后会陷入到系统调用，执行 `system_call` （定义于 `kernel/system_call.s`）中的指令。为什么这条指令执行后会直接执行 `system_call`？linux 在初始化时调用了 `set_system_gate(0x80, &system_call)` 将 `system_call` 注册为 `0x80` 号中断的处理函数。**后文提到的中断默认以系统调用作为例子。**

执行 `int` 指令后由用户栈陷入内核栈，**请注意，`int` 调用后并不是马上执行 `system_call` 中的代码**。在 `int` 调用后，`system_call` 执行前，需要把用户态对应的一些重要寄存器保存，用以中断返回后的现场恢复。具体的，我们需要依次保存 `ss`（用户数据栈底），`sp`（用户数据栈顶），`eflags`（标志寄存器），`cs`，`ip` 这五个寄存器的值到内核栈。随后进入到 `syscall`，此时内核栈是这样的：

![kernel stack0](https://raw.githubusercontent.com/KujouRinka/flowchart/master/20220613/kernel_stack0.svg)

随后执行 `system_call` 中的代码。而要从内核栈返回到用户栈，需要使用 `iret` 指令，此时必须保证内核栈中保存有用于恢复用户态的信息，如上图。

##### TSS

TSS 的结构由 `include/linux/sched.h` 中的 `struct tss_struct` 定义。在 linux 0.11 代码中，TSS 用于切换进程时，将当前 CPU 中寄存器的所有值保存到当前进程的 TSS，随后使用即将切换的进程的 TSS 中的信息来填写 CPU 中的寄存器，这样就完成了进程的切换。其中较重要的寄存器是 `ss`，`sp`，`cs`，`ip`。注意，这四个寄存器，不管是现在 CPU 中的值，还是即将切换的进程的 TSS 中所保存的这四个寄存器的值，记录的都是相应的**内核态**的状态。

##### switch_to 函数

定义于 `include/linux/sched.h`。[这篇博文](https://blog.csdn.net/smallmuou/article/details/6837087)解释地非常好，重点理解其中的 `ljmp` 指令，顺便复习一下 GDT 表的结构。

##### 其他的

时刻记住 `call` 相当于 `push ip`，`ret` 相当于 `pop ip`。

#### 流程分析

首先应该明确，修改进程切换的方法同时需要修改 `fork()` 系统调用，因为 linux 启动时需要通过 `fork()` 启动 shell。

首先来看 `system_call` 中的部分代码，不重要的省去：

``` assembly
system_call:
	cmpl $nr_system_calls-1,%eax
	ja bad_sys_call
	push %ds
	push %es
	push %fs
	pushl %edx
	pushl %ecx
	pushl %ebx
	...
	call sys_call_table(,%eax,4)
```

入栈一些寄存器，使用 `call` 调用真正的系统调用函数是，内核栈如下图：

![kernel stack1](https://raw.githubusercontent.com/KujouRinka/flowchart/master/20220613/kernel_stack1.svg)

系统调用执行完成后：

``` assembly
	pushl %eax
	movl current,%eax
	cmpl $0,state(%eax)		# state
	jne reschedule
	cmpl $0,counter(%eax)		# counter
	je reschedule
```

将 `eax` 入内核栈，其中两个 `cmpl` 分别判断进程的状态（运行？挂起？...）和时间片，根据判断结果决定是否跳转到 `reschedule` 处，`reschedule` 定义：

``` assembly
reschedule:
	pushl $ret_from_sys_call
	jmp schedule
```

将 `ret_from_sys_call` 的地址入栈，调用 `schedule` 函数进行进程切换。其中，对于 `ret_from_sys_call`，其核心为：

``` assembly
ret_from_sys_call:
	...
	popl %eax
	popl %ebx
	popl %ecx
	popl %edx
	pop %fs
	pop %es
	pop %ds
	iret
```

将内核栈中的内容 pop 直到仅剩 `ss`，`sp`，`eflags`，`cs`，`ip`，这些正好是恢复到用户态需要的寄存器信息。随后调用 `iret` 回到用户态。

#### schedule 和 switch_to

`schedule` 是一个 C 函数，跳转到 `schedule` 函数时，内核栈中的内容：

![kernel stack3](https://raw.githubusercontent.com/KujouRinka/flowchart/master/20220613/kernel_stack3.svg)

此时 `schedule`，中的调度算法计算出下一个切换的进程，并调用 `switch_to` 切换至这个进程。调用 `switch_to` 前，内核栈中的内容是不变的。

`switch_to` 的讲解见[这篇博文](https://blog.csdn.net/smallmuou/article/details/6837087)，写得非常好。

#### fork() 函数

`fork()`  原始的函数的讲解见李老师的[视频](https://www.bilibili.com/video/BV1d4411v7u7?p=12)后半段，解释地非常清楚。

#### 改写注意事项

改写时一定要清楚程序运行到改写处时，此时寄存器中的值和内核栈中的值。

#### 改写 switch_to() 函数

将原先的 `switch_to` 宏定义删除，在 `system_call.s` 添加 `switch_to` 的**汇编**实现。

我们定义 `long switch_to(long pnext, long ldt)` 为函数签名，其中 `pnext` 为指向一个 `task_struct` 的指针，即为 PCB，`ldt` 为其对应的 LDT 描述符地址。因此在 `schedule()` 中调用 `switch(pnext, _LDT(next))` 并进入 `switch_to` 函数体后，`8(%ebp)` 为 `pnext`，`12(%ebp)`为`_LDT(next)`，这是由 C 函数调用时将参数和返回值逆序压入栈中所决定的。下面贴上网络上给出的 `switch_to` 完整具体实现，并选择一些解释：

``` assembly
switch_to:
	pushl %ebp
	movl %esp,%ebp
	pushl %ecx
	pushl %ebx
	pushl %eax
	movl 8(%ebp),%ebx
	cmpl %ebx,current
	je 1f
	movl %ebx,%eax
	xchgl %eax,current
	movl tss,%ecx
	addl $4096,%ebx
	movl %ebx,4(%ecx)
	movl %esp,12(%eax)
	movl 8(%ebp),%ebx
	movl 12(%ebx),%esp
	movl 12(%ebp),%ecx	
	lldt %cx
	movl $0x17,%ecx
	mov %cx,%fs
	cmpl %eax,last_task_used_math
	jne 1f
	clts
1:
	popl %eax
	popl %ebx
	popl %ecx
	popl %ebp
	ret
```

从第 1 行到第 7 行所作的任务是处理 C 调用的帧栈；保存一些寄存器的值，因为这些寄存器稍后将使用，第 7 行 `movl 8(%ebp),%ebx` 执行完毕后，内核栈的状态：

![kernel stack4](https://raw.githubusercontent.com/KujouRinka/flowchart/master/20220613/kernel_stack4.svg)

``` assembly
movl 8(%ebp),%ebx
```

将 `pnext`，即新进程的 PCB 赋给 `ebx`

``` assembly
cmpl %ebx,current
je 1f
```

比较新进程的 PBC 与当前进程的 PCB，若相同，调到标签 1 处，即什么也不做返回函数。

``` assembly
movl %ebx,%eax
xchgl %eax,current
```

执行完这两句，`ebx` 和 `current` 值为新的进程，`eax` 为当前进程。

```assembly
movl tss,%ecx
addl $4096,%ebx
movl %ebx,4(%ecx)
```

此处的 tss 为我们自己定义的全局变量，作为所有进程公共的 TSS 表，我们在 `kernel/sched.h` 中加上 `struct tss_struct *global_tss = &(init_task.task.tss);` 一句，定义全局 TSS。

对于 `addl $4096,%ebx`，由于 `ebx` 中此前的值为新的进程的 PCB，对于 linux 0.11 其一页内存大小为 4k。对于一个进程，其 PCB 存储在一页内存的低地址，其内核栈在该页内存的高地址，结构如图：

![pcb](https://raw.githubusercontent.com/KujouRinka/flowchart/master/20220613/pcb.svg)

随后 `movl %ebx,4(%ecx)`，将内核栈栈顶赋值给 TSS 的 `sp0` 字段，该字段在 `tss_struct` 结构的第 4 个偏移位置，至于为什么要设置 `sp0`，引用一段来自[这里](https://github.com/Wangzhike/HIT-Linux-0.11/blob/master/exp4_process/%E5%9C%A8Linux-0.11%E4%B8%AD%E5%AE%9E%E7%8E%B0%E5%9F%BA%E4%BA%8E%E5%86%85%E6%A0%B8%E6%A0%88%E5%88%87%E6%8D%A2%E7%9A%84%E8%BF%9B%E7%A8%8B%E5%88%87%E6%8D%A2.md)的解释：

> 虽然所有进程共用一个 tss，但不同进程的内核栈是不同的，所以在每次进程切换时，需要更新 tss 中 esp0 的值，让它指向新的进程的内核栈，并且要指向新的进程的内核栈的栈底，即要保证此时的内核栈是个空栈，帧指针和栈指针都指向内核栈的栈底。 这是因为新进程每次中断进入内核时，其内核栈应该是一个空栈。

接下来三句：

``` assembly
movl %esp,12(%eax)
movl 8(%ebp),%ebx
movl 12(%ebx),%esp
```

`eax` 为当前进程 PCB。

这里我们需要先修改 `include/linux/sched.h` 中 `task_struct` ，即 PCB 的定义。新增一个 `long kernel_stack` 字段于结构体第四个字段，因此表现为相对于结构体首地址偏移 12 个单位。

这三句的作用是，将当前 `esp` ，即当前内核栈栈顶指针存储到 PCB 中；使 `ebx` 指向新进程 PCB，将新进程 PCB 的 `esp` 即内核栈栈顶指针赋值给 `esp` 寄存器。**此时已经完成内核栈由当前进程到新进程的切换**。

接下来两句：

``` assembly
movl 12(%ebp),%ecx
lldt %cx
```

切换 LDT。

``` assembly
movl $0x17,%ecx
mov %cx,%fs
cmpl %eax,last_task_used_math
jne 1f
clts
```

为固定指令，后续课程会讲到，目前不做要求。具体解释引用[蓝桥云课](https://www.lanqiao.cn/courses/115/learning)中一段解释，目前我还不了解：

> 这两句代码的含义是重新取一下段寄存器 fs 的值，这两句话必须要加、也必须要出现在切换完 LDT 之后，这是因为在实践项目 2 中曾经看到过 fs 的作用——通过 fs 访问进程的用户态内存，LDT 切换完成就意味着切换了分配给进程的用户态内存地址空间，所以前一个 fs 指向的是上一个进程的用户态内存，而现在需要执行下一个进程的用户态内存，所以就需要用这两条指令来重取 fs。
>
> 不过，细心的读者可能会发现：fs 是一个选择子，即 fs 是一个指向描述符表项的指针，这个描述符才是指向实际的用户态内存的指针，所以上一个进程和下一个进程的 fs 实际上都是 0x17，真正找到不同的用户态内存是因为两个进程查的 LDT 表不一样，所以这样重置一下 `fs=0x17` 有用吗，有什么用？要回答这个问题就需要对段寄存器有更深刻的认识，实际上段寄存器包含两个部分：显式部分和隐式部分，如下图给出实例所示，就是那个著名的 `jmpi 0, 8`，虽然我们的指令是让 `cs=8`，但在执行这条指令时，会在段表（GDT）中找到 8 对应的那个描述符表项，取出基地址和段限长，除了完成和 eip 的累加算出 PC 以外，还会将取出的基地址和段限长放在 cs 的隐藏部分，即图中的基地址 0 和段限长 7FF。为什么要这样做？下次执行 `jmp 100` 时，由于 cs 没有改过，仍然是 8，所以可以不再去查 GDT 表，而是直接用其隐藏部分中的基地址 0 和 100 累加直接得到 PC，增加了执行指令的效率。现在想必明白了为什么重新设置 fs=0x17 了吧？而且为什么要出现在切换完 LDT 之后？

最后，标签 1 后的代码为函数返回所做的工作，最后返回到系统调用 `iret`，退出中断。

#### 改写 fork() 函数

调用 `fork()` 函数时，函数调用链大致为 `fork(C)->syscall(asm)->sys_fork(asm)->copy_process(C)`。

进入 `copy_process` 前，内核栈内容如下图：

![kernel stack5](https://raw.githubusercontent.com/KujouRinka/flowchart/master/20220613/kernel_stack5.svg)

对应函数签名：

```c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
      long ebx,long ecx,long edx,
      long fs,long es,long ds,
      long eip,long cs,long eflags,long esp,long ss)
```

在 `copy_process` 中使用 `p = get_free_page()` 获得一页新的内存，该内存即为新进程的内核栈和 PCB 所使用的的空间，`p` 指向 PCB 首地址，我们添加下面的代码设置子进程的内核栈：

``` C
long *kernel_stack_top = (long *)(PAGE_SIZE + (long)p);
// prepare info for user stack
*(--kernel_stack_top) = ss & 0xffff;
*(--kernel_stack_top) = esp;
*(--kernel_stack_top) = eflags;
*(--kernel_stack_top) = cs & 0xffff;
*(--kernel_stack_top) = eip;
*(--kernel_stack_top) = ds & 0xffff;
*(--kernel_stack_top) = es & 0xffff;
*(--kernel_stack_top) = fs & 0xffff;
*(--kernel_stack_top) = gs & 0xffff;
*(--kernel_stack_top) = esi;
*(--kernel_stack_top) = edi;
*(--kernel_stack_top) = edx;
*(--kernel_stack_top) = (long)first_ret_from_fork;
*(--kernel_stack_top) = ebp;
*(--kernel_stack_top) = ecx;
*(--kernel_stack_top) = ebx;
*(--kernel_stack_top) = 0;	// fork return 0 in child process
p->kernel_stack = (long)kernel_stack_top;
```

第一条指令将 `kernel_stack_top` 设置为内核栈栈顶，后面依次填写数据，最后一条指令记录子进程的内核栈栈顶，也就是 `sp` 指针位置。

对于子进程，`fork()` 返回 0，因此将内核栈栈顶，也就是对应的 `eax` 的位置设置为 0。同时可以删掉 `copy_process ` 中一些不必要的代码，不删除不影响结果。

#### 改写后的运行逻辑

现在假设我们将所有修改应用，我们现在分析 `fork()` 后子进程的行为。

当 `switch_to` 切换到子进程并运行到其中的标签 1 处，依次对 `eax`，`ebx`，`ecx`，`ebp` 进行出栈并调用 `ret`，调用 `ret` 时内核栈栈顶为 `first_ret_from_fork()` 函数，考虑这个函数应该做什么。既然子进程已经切换完毕，我们则需要从内核态返回到用户态执行用户代码，因此该函数需要承担从内核态返回的任务，调用 `first_ret_from_fork()` 时，此时对应的**子进程**内核栈的内容如下：

![kernel stack6](https://raw.githubusercontent.com/KujouRinka/flowchart/master/20220613/kernel_stack6.svg)

很自然写出 `first_ret_from_fork()` 的定义：

``` assembly
first_ret_from_fork:
	popl %edx
	popl %edi
	popl %esi
	pop %gs
	pop %fs
	pop %es
	pop %ds
	iret
```

#### 写在后面的废话

上面记录了一些我认为较难理解的细节，我的完整的修改过程记录在了[这个 commit](https://github.com/KujouRinka/linux-0.11/commit/264072e0d97567eada96cb3b47d4d3d5193e3da0) 里。个人认为这个更像是对于线程的修改，因为父子进程共享了一个用户数据栈，实际写代码验证也正是如此，修改 `fork()` 后子进程中的变量会对父进程中的变量造成影响，反之亦然。
