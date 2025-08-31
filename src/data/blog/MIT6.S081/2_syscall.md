---
title: MIT6.S081 Lab syscall
description: '尝试在 Xv6 中添加系统调用，理解用户态与内核态的交互。'
pubDatetime: 2025-07-21 17:26:24
tags: ['MIT6.S081']
comment: true
---



系统调用流程概述

1. 用户程序调用用户空间包装函数（位于 `user.h`）
2. 包装函数通过汇编指令触发软中断
3. 内核终端处理程序根据系统调用号（位于 `syscall.h`）分派到正确的系统调用实现（位于 `syscall.c`）
4. 结果返回给用户程序

其中，`usys.pl` 为每个系统调用生成统一的汇编代码模板，处理系统调用号和参数传递，触发软中断进入内核态。

## 1 Sysetm call tracing

要求：
1. 新增 `trace` 系统调用
	- 接受一个整数参数 `mask`，二进制位用于指定要追踪的系统调用。
	- 例如：`trace(1 << SYS_fork)` 表示追踪 `fork` 系统调用
2. 修改内核以输出追踪信息
	- 当被追踪的系统调用即将返回时，内核需打印一行信息，包括：
		- 进程 ID
		- 系统调用名称
		- 返回值
	- 仅当系统调用编号在 `mask` 中对应的位被设置时，才输出信息。
3. 追踪的继承性
	- `trace` 调用后，当前进程及后续通过 `fork` 创建的子进程均启用追踪，但不得影响其它无关进程

首先是添加系统调用的声明：
```c
// user.h
int trace(int);

// usys.S
entry("trace");

// syscall.h
#define SYS_trace  22
```

其次是实现内核调用 `sys_trace`：
```c
// 需要在proc.h为struct proc添加新变量
struct proc {
	...
	int trace_mask; // 进程要追踪的掩码
}

// 在sysproc.c中添加sys_trace()函数
// trace the system call from user space
uint64
sys_trace(void) {
  int mask;

  if (argint(0, &mask) < 0) // 获取mask
    return -1;
  myproc()->trace_mask = mask;
  return 0;
}
```

在 `kernel/proc.c` 中修改 `fork()`，使得子进程能够继承父进程的跟踪掩码：
```c
int fork(void) {
	...
	np->trace_mask = p->trace_mask;
}
```

最后在 `kernel/syscall.c` 中修改 `syscall()` 函数，在系统调用执行完成后检查 `trace_mask`，若当前系统调用编号被设置，则打印追踪信息：
```c
void syscall(void)
{
  int num;
  struct proc *p = myproc();
  // 系统调用名称数组，用于索引
  char* syscall_name[22] = {"fork", "exit", "wait", "pipe", "read", 
  "kill", "exec", "fstat", "chdir", "dup", "getpid", "sbrk", "sleep", 
  "uptime", "open", "write", "mknod", "unlink", "link", "mkdir", "close", 
  "trace"};

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    if ((1 << num) & (p->trace_mask)) // 检查当前调用是否被追踪
      printf("%d: syscall %s -> %d\n", p->pid, syscall_name[num - 1], p->trapframe->a0);
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```


## 2 Sysinfo

### 2.1 系统调用声明

要求：新增 `sysinfo` 的系统调用，其参数为 `struct sysinfo*`（声明在 `kernel/sysinfo.h`），要求填充该结构体。

按照之前的步骤添加声明，唯一不同的是在 `user/user.h` 中需要声明 `struct sysinfo` 的存在：
```c
struct sysinfo;
...
int sysinfo(struct sysinfo*);
```

### 2.2 收集空闲内存量

在 `kernel/kalloc.c` 中存在以下声明：
```c
struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
```
由于物理内存中被分成了页进行管理，这里实际上是用链表来存储空闲的内存页。其中 `freelist` 为头节点，而 `struct run` 定义了一个链表节点结构，这里实现为单链表。
于是我们可以据此计算出空闲空间的大小：
```c
// get the free memory size of user space
uint64 freememSize(void) {
  struct run *r = kmem.freelist;
  uint64 i = 0; // 空闲页的数量
  while (r) {
    i++;
    r = r->next;
  }
  return i * PGSIZE;
}
```

### 2.3 收集进程数量
`struct sysinfo` 中的 `nproc` 设置为 `state` 不为 UNUSED 的进程数量。

在 `kernel/proc.c` 的头部声明了 `struct proc proc[NPROC]`，这相当于是进程数组，因此我们只需要遍历数组即可：`
```c
// get the num of procs that aren't UNUSED.
uint64 nproc_active(void) {
  int i = 0;
  uint64 n = 0;
  while (i < NPROC) {
    if (proc[i].state != UNUSED) 
      n++;
    i++;
  }
  return n;
}
```

### 2.4 实现 sysinfo

在 `kernel/sysproc.c` 中添加 `sys_sysinfo` 函数：
```c
// collects information about the running system.
uint64
sys_sysinfo(void) {
  uint64 st; // 指向 struct sysinfo 的指针
  struct sysinfo sf;

  if (argaddr(0, &st) < 0) // 获取用户空间的目标虚拟地址
    return -1;
  sf.freemem = freememSize();
  sf.nproc = nproc_active();
  if (copyout(myproc()->pagetable, st, (char *)&sf, sizeof(sf)) < 0)
    return -1;
  return 0;
}
```

之所以需要用到 `copyout` ，是因为我们传递给 sysinfo 函数的参数是一个用户空间的指针，而我们的 sys_sysinfo 函数是内核函数，其运行在内核空间，在其内填充的 `struct sysinfo` 也位于内核空间，用户空间的指针无法直接访问。而 `copyout` 函数的作用就是**将内核空间的数据复制到用户空间**。

`copyout` 函数的用法（可以参考 `kernel/sysfile.c` 的 `sys_fstat()` 和 `kernel/file.c` 的 `filestat()`）：
```c
int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len);
```
- `pagetable`：目标进程的页表
- `dstva`：用户空间的目标虚拟地址
- `src`：内核空间的元数据地址
- `len`：要复制的字节数
成功返回 0，失败返回 -1（如用户地址非法或不可写）。