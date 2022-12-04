---
title: 6.s018 lab5 cow(Copy-on-write fork) 踩坑指南
date: 2022-12-4 13:00
---

如题，不讲废话，直接开始。2020 年的 lab，[lab 链接](https://pdos.csail.mit.edu/6.828/2020/labs/cow.html)

#### 复述

实现调用 `fork()` 时的写时复制，即 copy-on-write。

#### 思路

根据提示：

- 修改 `uvmcopy()` 函数。该函数仅会被 `fork()` 调用，原本用于将**父进程**的 pagetable 中所含的所有 page 复制到**子进程**的 pagetable 中，类似 deepcopy；修改后的 `uvmcopy()` 函数使用浅拷贝，子进程的 pagetable 结构与父进程完全一致。**但同时**，我们需要把父进程和子进**两者**的 pagetable 中的所有 pte 中，`PTE_W` 有效的 pte 的这一位清空，具体表现为使用 `*pte &= ~PTE_W`。同时，我们需要使用 pte 中 `RSW` 中的其中一位，用于标记这个 pte 实现了 cow，这里我选择了使用第 8 位作为标记，定义为：`#define PTE_COWPG (1L << 8)` 对 pte 做 `*pte |= PTE_COW` 操作。

- 修改 `usertrap()` 函数，为其添加 scause 处理入口。由 riscv 文档，我们仅需处理 15 号 code，该 code 代表 Store/AMO page fault，而 13 号代表的 Load page fault 不在我们实现 cow 的处理范围内。在这个 scause 处理分支里，我们需要通过调用 `r_stval()` 得知导致 page fault 的**虚拟**地址，并**一定**要判断这个地址是否为合法的 cow 地址，若否，将进程标记为 killed 并返回，否则，复制 stval 地址中一个 page 的内容至新跑分配的空间，计算出正确的 pte 目录并重写。

- 这里是一步非常容易出错，即关于每一个 page 的引用计数 (Reference Count)。基本思路是维护一个关于每个 page 的 array，每个 page 对应其在 array 中的索引为该 page 的物理地址除以 PGSIZE(4096)，即 `pa / PGSIZE`，我们还能粗略计算出这个表的长度为 `PHYSTOP >> PGSHIFT`。对于：

  + 每次 `kalloc()` 调用，分配新的 page，我们对该 page 所对应的 rc 加 1

  + 每次 `kfree()` 调用，将该 page 对应的 rc 减 1，若此时 rc 为 0，则真正释放这段内存，否则什么也不做。
  这里，我们有如下定义：`#define RC_SZ PHYSTOP >> PGSHIFT`
  + `uvmcopy()` 中，对每个添加了 `PTE_COWPG` 的 pte 条目对应的**物理地址**的 rc 加 1
  + `usertrap()` 中，在处理 cow 的分支中，对导致 page fault 的**虚拟地址**所对应的**物理地址**的 rc 减 1

而任何一步对 rc 的读写，需要进行**加锁**处理，否则会导致条件竞争，这种情况下，为 xv6 添加 `CPUS=1` 参数能勉强通过测试，但实际测试中无法得到正确结果。

- 修改 `copyout()` 函数，这一步与在 scause 中所做的修改如出一辙，不再复述。

#### 代码修改

- `defs.h`：添加几个导出的函数

``` c
  // kalloc.c
+ void            acquire_kmem(void);
+ void            release_kmem(void);
+ void            add_rc(uint64);
+ void            sub_rc(uint64);
+ int             get_rc(uint64);
+ void            set_rc(uint64, int);
  void*           kalloc(void);
  void            kfree(void *);
  void            kinit(void);

  // ...

  // vm.c
+ pte_t *         walk(pagetable_t, uint64, int);
```

- `riscv.h`：添加 `PTE_COWPG` 定义

``` c
#define PTE_COWPG (1L << 8)
```

- `kalloc.c`：添加一些 rc 的函数，修改 `kalloc()` 和 `kfree()`。需要注意操作 rc 时的加锁时机。这里的代码优化空间很大，我在这里的实现是 `kmem.freelist` 和 `kmem.rc` 使用同一把锁，可以再研究一下锁的颗粒度，或者为 rc 单独细分出一个锁。

``` c
+ #define RC_SZ PHYSTOP >> PGSHIFT
  struct {
    struct spinlock lock;
    struct run *freelist;
+   uint8 rc[RC_SZ];
  } kmem;

+ void acquire_kmem() {
+   acquire(&kmem.lock);
+ }

+ void release_kmem() {
+   release(&kmem.lock);
+ }

+ void add_rc(uint64 a) {
+   ++kmem.rc[a >> PGSHIFT];
+ }

+ void sub_rc(uint64 a) {
+   --kmem.rc[a >> PGSHIFT];
+ }

+ int get_rc(uint64 a) {
+   return kmem.rc[a >> PGSHIFT];
+ }

+ void set_rc(uint64 a, int rc) {
+   kmem.rc[a >> PGSHIFT] = rc;
+ }
```

**注意**下面对 `freerange()` 函数的修改：该函数在 xv6 启动时调用，并对每个 page 调用一次 `kfree()` ，由于 `kfree()` 会将所对应 page 的 rc 减 1，我们需要在 `kfree()` 调用之前将所对应的 page 的 rc 置为 1，否则会导致系统初始化后这些 page 的 rc 为 -1，引起未知的错误。这里不用为 rc 加锁的原因是 vx6 启动时只会使用一个核，不会产生竞争问题。
``` c
  void
  freerange(void *pa_start, void *pa_end)
  {
    char *p;
    p = (char*)PGROUNDUP((uint64)pa_start);
-   for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
+   for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) {
+     kmem.rc[((uint64)p) >> PGSHIFT] = 1;
    kfree(p);
+   }
  }
```
下面是 `kalloc()` 和 `kfree()` 的修改，注意锁的操控时机：
``` c
  void
  kfree(void *pa)
  {
    struct run *r;

    if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
      panic("kfree");

+   acquire(&kmem.lock);
+   int rc = get_rc((uint64) pa);
+   if (rc == 0)
+     panic("kfree: bad rc");
+   sub_rc((uint64) pa);
+   if (get_rc((uint64) pa) != 0) {
+     release(&kmem.lock);
+     return;
+   }
    // Fill with junk to catch dangling refs.
    memset(pa, 1, PGSIZE);

    r = (struct run*)pa;
-   acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  }
  
  void *
  kalloc(void)
  {
    struct run *r;

    acquire(&kmem.lock);
    r = kmem.freelist;
-   if(r)
+   if(r) {
+     kmem.freelist = r->next;
+     if (get_rc((uint64) r) != 0)
+       panic("kalloc: bad rc");
+     add_rc((uint64) r);
+   }
    release(&kmem.lock);

    if(r)
      memset((char*)r, 5, PGSIZE); // fill with junk
    return (void*)r;
}
```
- `vm.c:uvmcopy`：删除原来的分配内存的代码，取而代之的是仅对 pte 的修改。注意：对 rc 的操作需要加锁。

``` c
+ #include "spinlock.h"
  int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
  {
    pte_t *pte;
    uint64 pa, i;
    uint flags;
-   char *mem;

    for(i = 0; i < sz; i += PGSIZE){
      if((pte = walk(old, i, 0)) == 0)
        panic("uvmcopy: pte should exist");
      if((*pte & PTE_V) == 0)
        panic("uvmcopy: page not present");
      pa = PTE2PA(*pte);
      flags = PTE_FLAGS(*pte);
-     if((mem = kalloc()) == 0)
-       goto err;
-     memmove(mem, (char*)pa, PGSIZE);
+     if (flags & PTE_W) {
+       flags &= ~PTE_W;
+       flags |= PTE_COWPG;
+       *pte = PA2PTE(pa) | flags;
+     }
-     if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
+     if(mappages(new, i, PGSIZE, pa, flags) != 0){
-       kfree(mem);
        goto err;
      }
+     acquire_kmem();  // NOTE!!!
+     add_rc(pa);
+     release_kmem();  // NOTE!!!
    }
    return 0;

   err:
    uvmunmap(new, 0, i / PGSIZE, 1);
    return -1;
  }
```

- `trap.c:usertrap`：在判断 scause 的分支中插入一段判断 15 号 code 的代码即可，这里我在处理 `PTE_COWPG` 的 page 时做了一个优化：当产生 page fault 的地址的 rc 为 1 时不再做重新分配内存，复制，释放原内存的操作，而是直接修改原来 pte 的 flag 使其拥有 `PTE_W`，并移除掉 `PTE_COWPG` 标记。老样子，还是得注意锁的使用。

``` c
// ...
else if (cause == 15) {
    uint64 stval = r_stval();
    if (stval > p->sz || stval >= MAXVA) {
      p->killed = 1;
      goto out;
    }
    stval = PGROUNDDOWN(stval);
    pte_t *pte = walk(p->pagetable, stval, 0);
    if(pte == 0 || (*pte & PTE_V) == 0) {
      p->killed = 1;
      goto out;
    }
    uint64 pa = PTE2PA(*pte);
    uint flags = PTE_FLAGS(*pte);
    if (pa == 0 || (flags & PTE_COWPG) == 0) {
      p->killed = 1;
      goto out;
    }
    flags &= ~PTE_COWPG;
    flags |= PTE_W;
    acquire_kmem();
    int rc = get_rc(pa);
    if (rc == 0) {
      panic("usertrap: bad rc");
    } else if (rc == 1) {
      *pte |= PTE_W;
      *pte &= ~PTE_COWPG;
      release_kmem();
    } else {
      release_kmem();
      uint64 mem = (uint64) kalloc();
      if (mem == 0) {
        p->killed = 1;
        goto out;
      }
      memmove((char*)mem, (char*)pa, PGSIZE);
      // NOTE: cannot use sub_rc(pa)
      kfree((void *) pa);
      *pte = PA2PTE(mem) | flags;
    }
  }
// ...
```

代码中有一段不能使用 `sub_rc()` 的注释，原因如下：在 `CPUS=1` 的参数下使用是没有问题的，但若是在多核模式中运行，在中间一段没有持有锁的时期，其他进程也许会修改 rc 的数量，此时仅仅调用 `sub_rc()` 可能导致 rc 为 0 却没有被释放的情况。

- `vm.c:copyout`：和 `usertrap()` 的修改如出一辙，不多解释：

``` c
  int
  copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
  {
    uint64 n, va0, pa0;
  
    while(len > 0){
      va0 = PGROUNDDOWN(dstva);
      pa0 = walkaddr(pagetable, va0);
      if(pa0 == 0)
        return -1;

+     if (va0 >= MAXVA)
+       return -1;
+     va0 = PGROUNDDOWN(va0);
+     pte_t *pte = walk(pagetable, va0, 0);
+     if (*pte & PTE_COWPG) {
+       uint64 pa = PTE2PA(*pte);
+       uint16 flag = PTE_FLAGS(*pte);
+       flag &= ~PTE_COWPG;
+       flag |= PTE_W;
+       acquire_kmem();
+       int rc = get_rc(pa);
+       if (rc == 1) {
+         *pte = PA2PTE(pa) | flag;
+         release_kmem();
+       } else {
+         release_kmem();
+         uint64 mem = (uint64) kalloc();
+         if (mem == 0) {
+           return -1;
+         }
+         memmove((char *) mem, (char *) pa, PGSIZE);
+         acquire_kmem();
+         sub_rc(pa);
+         release_kmem();
+         *pte = PA2PTE(mem) | flag;
+         pa0 = mem;
+       }
+     }
  
      n = PGSIZE - (dstva - va0);
      if(n > len)
        n = len;
      memmove((void *)(pa0 + (dstva - va0)), src, n);
  
      len -= n;
      src += n;
      dstva = va0 + PGSIZE;
    }
    return 0;
  }
```

#### 总结

这个 lab 总体思路和实现不是很难，但却让我究竟了几个小时，主要就是卡在了引用计数的同步问题上，这个一定要多加注意。可以通过先添加 `CPUS=1` 参数看看测试能否通过，用这种方式判断是否问题出在同步上。

测试通过结果没有什么意义，就懒得贴上来了（
