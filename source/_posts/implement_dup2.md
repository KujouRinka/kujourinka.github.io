---
title: 实现 dup2() 函数
date: 2022-02-24 21:58
---


来自 APUE 上的第 3.3 题，原文如下：

> 编写一个与 3.12 节中 [dup2](https://man7.org/linux/man-pages/man2/dup.2.html) 功能相同的函数，要求不调用 fcntl 函数，并且要有正确的出错处理。

函数原型是 `int dup2(int fd, int fd2)` ，作用是复制文件描述符 `fd`  ，使 `fd2` 为新的描述符的值，即 `fd` 与 `fd2` 共享同一个文件表项。若 `fd2` 已经打开，则先将其关闭。

很有意思的一个题。由于要求不调用 `fcntl` ，则基本思路为使用 `dup` 函数实现之。



#### 实现基本逻辑 `myDup2`

我们令自己实现的函数原型为 `int myDup2(int fd, int fd2)` ，先考虑该函数的基本逻辑。

我们让所有的 `errno` 处理在该函数进行处理。

首先，我们需要要求传入函数的两个文件描述符必须有效，于是有：

``` c
int myDup2(int fd, int fd2) {
    int max_fd = getdtablesize();    // 获取文件描述符表的大小
    if (fd < 0 || fd >= max_fd || fd2 < 0 || fd2 >= max_fd || !isOpen(fd)) {
        errno = EBADF;
        return -1;
    }
    // ...
```

其中 `getdtablesize()` 用于获取文件描述符表的的大小，限制了系统最大的文件描述符。

`isOpen` 函数用于判断给定文件描述符是否为已打开的文件描述符，后续实现。

若给定的文件描述符 `fd` 和 `fd2` 无效，则设置 `errno` 并返回 `-1` 。

接下来，由 `dup2` 的定义，若 `fd` 等于 `fd2` ，则 `dup2` 返回 `fd2` ，而不关闭它；以及如果 `fd2` 已经打开，则先将其关闭：

``` c
    // ...
    if (fd == fd2)
        return fd;
    if (isOpen(fd2))
        close(fd2);
    // ...
```

接下来为 `myDup2` 的核心逻辑。根据「 `dup` 返回的新文件描述符一定是当前可用文件描述符的最小数值」，基本思路为不断使用 `dup(fd)` 直至其返回值与 `fd2` 相等，随后将之前使用 `dup` 打开的文件描述符关闭即可。

考虑到需要记录由 `dup(fd)` 创建的文件描述符用于删除，加之 `C` 需要手动实现类似动态数组之类的数据结构，我们~~偷懒~~使用递归替代手动存储的工作。我们将这个操作使用 `int recursiveDup(int fd, int fd2)` 完成，我们定义返回值为成功复制后的 `fd2` ，若失败，则返回 `-1` ：

``` c
    // ...
    if ((fd2 = recursiveDup(fd, fd2)) == -1)
        errno = EBADF;
    return fd2;
}
```

调用 `recursiveDup` 后期望得到 `fd2` 的值，若失败，则设置 `errno` 。



#### 实现核心递归函数 `recursiveDup`

首先：

``` c
int recursiveDup(int fd, int fd2) {
    int nf = dup(fd);
    if (nf == -1)
        return -1;
    if (nf == fd2)
        return nf;
    int rf = recursiveDup(fd, fd2);
    // ...
```

使用 `dup` 复制文件描述符，若失败，返回 `-1` ，这里并不处理 `errno` 因为我们将所有的 `errno` 处理放在 `myDup` 中。若得到期望的文件描述符的值，返回之。若否，继续递归调用 `recursiveDup` 直至其产生。

之后：

``` c
    // ...
    close(nf);
    return rf;
}
```

关闭不合要求的 `nf` 返回结果（不论是否为 `-1`）。



#### 收尾，实现 `isOpen`

实现如下：

``` c
int isOpen(int fd) {
    int old_errno = errno;
    int test_f = dup(fd);
    errno = old_errno;
    if (test_f == -1) {
        return 0;
    }
    close(test_f);
    return 1;
}
```

我们利用 `dup` 函数只能作用于已打开文件描述符的机制进行判断。注意，我们需要对本来的 `errno` 进行处理，由于我们的 `isOpen` 仅进行判断，不应对系统本来的 `errno` 有影响，`dup` 后需要及时恢复。



#### 完整代码

``` c
int isOpen(int fd) {
    int old_errno = errno;
    int test_f = dup(fd);
    errno = old_errno;
    if (test_f == -1) {
        return 0;
    }
    close(test_f);
    return 1;
}

int recursiveDup(int fd, int fd2) {
    int nf = dup(fd);
    if (nf == -1)
        return -1;
    if (nf == fd2)
        return nf;
    int rf = recursiveDup(fd, fd2);
    close(nf);
    return rf;
}

int myDup2(int fd, int fd2) {
    int max_fd = getdtablesize();
    if (fd < 0 || fd >= max_fd || fd2 < 0 || fd2 >= max_fd || !isOpen(fd)) {
        errno = EBADF;
        return -1;
    }
    if (fd == fd2)
        return fd;
    if (isOpen(fd2))
        close(fd2);
    if ((fd2 = recursiveDup(fd, fd2)) == -1)
        errno = EBADF;
    return fd2;
}
```

个人感觉很有意思的一个题，不断反复利用 `dup` 函数的性质得到结果。
