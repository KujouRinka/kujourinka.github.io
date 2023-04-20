---
title: 简记一次 rCore 中 sys_sbrk() 与 lazy allocation 的实现
date: 2023-04-20 17:00
---

#### sbrk() 的作用
`sbrk()` 系统是做什么的，可以参考[这里](https://man7.org/linux/man-pages/man2/brk.2.html)，简要来说，就是调整用户程序的堆（heap）顶位置，表现为对堆大小的增大或缩小。

#### rCore 的地址空间布局与调整
这里的 `sbrk()` 实现仅与用户虚拟地址空间有关。参考[这里](http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter4/5kernel-app-spaces.html#term-vm-app-addr-space)。

对于 rCore 原始布局，在低地址部分，从低地址往高地址依次为：`.text`，`.rodata`，`.data`，`.bss`，`guard page`，`user stack`，随后才是 `heap`：

![application address space](https://raw.githubusercontent.com/rcore-os/rCore-Tutorial-Book-v3/main/source/chapter4/app-as-full.png)

但我对这个布局不太满意，于是通过修改 `mm::memory_set::MemorySet::from_elf()`，将空间布局修改为如下：
```
/// High 256GB
/// +-------------------+
/// |  Trampoline Code  |
/// +-------------------+  <- TRAMPOLINE
/// |    TrapContext    |
/// +-------------------+  <- TRAP_CONTEXT, user_stack_top
/// |    User Stack     |
/// +-------------------+  <- user_stack_bottom
/// |       ...         |
///
/// Low 256GB
/// |       ...         |
/// +-------------------+
/// |    Heap Memory    |
/// +-------------------+  <- heap_bottom
/// |      .bss         |
/// +-------------------+
/// |      .data        |
/// +-------------------+
/// |      .rodata      |
/// +-------------------+
/// |      .text        |
/// +-------------------+  <- BASE_ADDRESS (0x10000 va)
```

heap 紧挨着 bss 段，而 user stack 调整到高地址处。这么做的原因是当物理地址理论足够的情况下，可以方便 heap 和 user stack 的拓展，同时由于二者位置相差大，不太可能重叠，而原实现相较于此，user stack 的大小限制为 8K（即两个 PAGE SIZE），由由于其位置的特殊性，上有 heap 空间，下有 guard page，kernel 段，难以拓展。这个修改很简单，我们的重点不在这里。

#### sbrk()

与其他的系统调用实现一样，先添加系统调用接口，再实现具体的逻辑。

``` rust
pub fn sys_sbrk(size: i32) -> isize {
  if let Some(old_brk) = change_program_brk(size) {
    return old_brk as isize
  } else {
    -1
  }
}
```

顺着调用链，我们来到 `task::task::TaskControlBlock::change_brk()`，这是实际的 sbrk 的实现。主要的逻辑为根据判断传入的参数的正负判断堆空间的缩减，并更新相应的堆顶指针：

``` rust
// ...
let old_brk = self.program_brk;
let new_brk = self.program_brk as isize + size as isize;
let result = if size < 0 {
  // shrink heap
  self.memory_set.shrink_to(...)
} else {
  // extend heap
  self.memory_set.append_to(...)
};
if result {
  self.program_brk = new_brk as usize;
  Some(old_brk)
} else {
  None
}
```

缩小和扩大 heap 空间分别对应 `shrink_to` 和 `append_to` 两个函数，而二者的逻辑较为相似，这里以 `append_to` 为例：

``` rust
pub fn append_to(&mut self, page_table: &mut PageTable, new_end: VirtPageNum) {
  for vpn in VPNRange::new(self.vpn_range.get_end(), new_end) {
    self.map_one(page_table, vpn);
  }
  self.vpn_range = VPNRange::new(self.vpn_range.get_start(), new_end);
}
```

即朴实地为每一页在 PageTable 里创建一个映射，并没有什么花里胡哨的东西。

#### lazy alloc

lazy alloc 可以概括为：只标记，需要时分配。

具体来说：

- 当调用 `sbrk()` 申请拓展 heap 空间，内核只增加堆顶指针（在这里是 `TaskControlBlock::program_brk`）的值，不做实际的内存分配，添加映射。用户程序访问这段内存会引起 PageFault，我们在处理该 Trap 的时候通过 `stval` 寄存器判断该地址是否位于 `heap_bottom` 和 `program_brk` 之间，若是，分配实际内存，添加页表映射；
- 当调用 `sbrk()` 申请缩小 heap 空间，需要减小 `program_brk` 的值，同时收回原堆顶和现堆顶之间的 page。

##### 拓展 PageTable

由于原实现中 `PageTable::map()` 和 `PageTable::unmap()` 在接口方面实现较为有限，我将其进行了拓展，将

``` rust
pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags, mut frame: Option<FrameTracker>);
pub fn unmap(&mut self, vpn: VirtPageNum, dealloc: bool);
```

修改为：

``` rust
pub fn map(&mut self, args: MapArgs) {
  let MapArgs { vpn, ppn, flags, mut frame } = args;
  // ...
}
pub fn unmap(&mut self, args: UnmapArgs) {
  let vpn = args.vpn;
  let pte = match self.find_pte(vpn) {
    Some(pte) if pte.is_valid() => pte,
    _ => if args.panic {
      panic!("vpn {:?} should mapped but not", vpn);
    } else {
      return;
    }
  };
  // ...
}
```

目前，`MapArgs` 和 `UnmapArgs` 中字段足够使用：

``` rust
pub struct MapArgs {
  vpn: VirtPageNum,
  ppn: PhysPageNum,
  flags: PTEFlags,
  frame: Option<FrameTracker>,
}

pub struct UnmapArgs {
  vpn: VirtPageNum,
  dealloc: bool,
  panic: bool,
}
```

这两个内以设计模式中的 Builder 设计，使用一系列的 `with_...` 成员函数调整参数。

为了便于控制后续课程中 lazy_alloc 的使用与否，为 lazy alloc 新增一个 feature 并使用条件编译：

``` toml
# Cargo.toml
# ...
[features]
default = ["sbrk_lazy_alloc"]
sbrk_lazy_alloc = []
```

##### Trap

`trap_handler` 中为 lazy alloc 添加的入口：

``` rust
Trap::Exception(Exception::StoreFault)
| Trap::Exception(Exception::StorePageFault)
| Trap::Exception(Exception::LoadFault)
| Trap::Exception(Exception::LoadPageFault) => {
  let tcb = get_current_tcb_ref();
  let ok = if stval >= tcb.heap_bottom && stval < tcb.program_brk {
    // lazy allocation for sbrk()
    #[cfg(feature = "sbrk_lazy_alloc")] {
      lazy_alloc_page(stval.into())
    }
    #[cfg(not(feature = "sbrk_lazy_alloc"))] {
      false
    }
// ...
```

当判断 `stval` 中的地址为 heap 中的有效地址时，使用 `lazy_alloc_page()` 分配一页内存，实现较为简单，这里不过分说明。

##### change_brk()

我们的重点是 `change_brk()` 的修改：

``` rust
pub fn change_brk(&mut self, size: i32) -> Option<usize> {
  let old_brk = self.program_brk;
  let new_brk = self.program_brk as isize + size as isize;
  if new_brk < self.heap_bottom as isize {
    return None;
  }
  let ok;
  cfg_if! {
    if #[cfg(feature = "sbrk_lazy_alloc")] {
      ok = true;
      if size < 0 {
        self.memory_set
          .remove_framed_area(
            VirtAddr::from(new_brk as usize),
            VirtAddr::from(self.program_brk),
          );
      }
    } else {
      // ...
    }
  }
  if ok {
    self.program_brk = new_brk as usize;
    Some(old_brk)
  } else {
    None
  }
}
```

我们注意到，当 `size > 0` 我们仅仅增加了 `program_brk` 的值就返回，其他什么事也不做，而当该值小于 0，只增加了一点微不足道的操作：释放减少的那段空间。

至于为什么我拓展了 PageTable 的参数，下面以 `remove_framed_area` 为例：

``` rust
pub fn remove_framed_area(
  &mut self,
  start_va: VirtAddr,
  end_va: VirtAddr,
) {
  for vpn in VPNRange::new(start_va.ceil(), end_va.floor()) {
    self.page_table.unmap(
      UnmapArgs::builder(vpn)
        .with_dealloc(true)
        .with_panic(false),
    )
  }
}
```

可以看到我们通过链式构造一步一步对 `UnmapArgs` 做出了要求，我们要求释放物理地址，并且保证不会 panic。这里我们为什么要保证不会 panic？由于原实现中的 `unmap()`，设计默认认为传给它的参数一定经过映射，当无法找到最后一级页表时，会认为是内核设计出现了 bug，于是直接通过 `unwarp()` 进行了 panic。

而这里我们释放 heap 空间，由于 lazy alloc 的存在，我们并不能事先知道哪些 page 被映射了而哪些没有，我们又不想付出额外的存储代价，于是选择对这段释放的空间的每一 page 进行遍历，其中有很大概率会遇见由于从未读写而未进行分配的 page，这样直接作为参数传给原来的 `unmap()` 必然会导致内核 panic，于是抽象出了 `unmap()` 的参数，为其添加 `panic` 字段，用于提示 `unmap()` 是否能够 panic。

最终的实现见 [#876012b](https://github.com/KujouRinka/rCore/tree/876012b64189dd9c65e72743d4dbec9e72f22884)