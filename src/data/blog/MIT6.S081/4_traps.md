---
title: MIT6.S081 Lab traps
description: '实现系统调用和中断处理，分析 trapframe 和上下文切换。'
pubDatetime: 2025-07-25 18:52:32
tags: ['MIT6.S081']
comment: true
---



## 1 Backtrace

题目要求：编译器会在每个栈帧中放置一个帧指针，该指针保存着调用者帧指针的地址。您的 backtrace 应利用这些帧指针遍历堆栈，并打印每个栈帧中保存的返回地址。

将 `backtrace()` 原型添加到 `defs.h` 中。
在 `kernel/riscv.h` 中添加函数，以获取帧指针：
```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

在 `kernel/printf.c` 中添加 `backtrace` 函数：
```c
void
backtrace(void) {
  printf("backtrace:\n");
  uint64 fp = r_fp();
  uint64 top = PGROUNDUP(fp);

  while (fp < top) {
    uint64 ra = *(uint64*)(fp - 8);
    printf("%p\n", &ra);
    fp = *(uint64*)(fp - 16);
  }
}
```
- 由于是从低地址向高地址遍历栈帧，因此只需检查 `PGROUNDUP(fp)` 边界即可；
- 需要注意返回地址位于 `*(fp-8)` 处，帧指针位于 `*(fp-16)` 处。

在 `printf.c` 的 `panic()` 中添加 `backtrace()`：
```c
void
panic(char *s)
{
  pr.locking = 0;
  printf("panic: ");
  printf(s);
  printf("\n");
  backtrace();
  panicked = 1; // freeze uart output from other CPUs
  for(;;)
    ;
}
```


## 2 Alarm
功能概述：
1. `sigalarm(n, fn)` ：
    - 设置每隔 `n` 个 CPU 时间 ticks 调用一次 `fn` 函数
    - 当 `fn` 返回后，程序从被中断的地方继续执行
    - 如果调用 `sigalarm(0, 0)`，则停止警报调用
2. `sigreturn()` ：
    - 由警报处理函数调用，用于恢复被中断的上下文

首先需要理解整个系统的调用流程：
1. 在 `alarmtest` 中初始化对 `sigalarm(2, periodic)` 的调用，内核会在 `proc` 中记录这些参数；
2. 每个时钟中断(tick)发生时：
	- 硬件触发中断-> 执行 `usertrap`
	- 根据条件 `which_dev == 2` 判断时钟中断
	- 只有 `ticks` 计数器等于初始化设置的 `interval` 时才调用 `periodic` 处理函数
3. 由于我们将 `usertrap` 的下一步变成了执行 `periodic` 处理函数而不是 `usertrapret`，因此需要在 `periodic` 中调用 `sigreturn()` 函数，从而进入恢复阶段
4. 在 `sigreturn()` 中我们需要将保存的上下文恢复和重置一些状态

### 2.1 一般方案

初始设置：
- 在 `MAKEFILE` 的添加 `alarmtest.c`；
- 在 `user/user.h` 中添加函数声明：
```c
int sigalarm(int ticks, void (*handler)());
int sigreturn(void);
```
- 更新 `user/usys.pl`、`kernel/syscall.h` 和 `kernel/syscall.c`。

`proc` 添加变量：
```c
 int interval;             // 警报间隔
 void (*handler)();        // 处理函数指针，无返回值和参数传入
 int ticks;                // 距离上次警报的ticks数
 int in_handler;           // 是否在处理函数中
 struct trapframe *alarm_trapframe; // 保存原始的trapframe
```
- `in_handler` 防止处理程序被重复调用
- 这里使用 `alarm_trapframe` 来避免了冗长的手动保存寄存器，保持代码整洁且符合原有的 xv6 风格，但是缺点是增加了内存占用以及性能开销。


在 `proc.c` 中添加对 `alarm_trapframe` 的分配和释放：
```c
static struct proc*
allocproc(void)
{
	... // 其它代码
	
	// Allocate a trapframe page.
	if(((p->trapframe = (struct trapframe *)kalloc()) == 0) 
  || (p->alarm_trapframe = (struct trapframe *)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  
  ...
}
```


在 `usertrap` 中添加对时钟中断的处理：
```c
} else if ((which_dev = devintr()) != 0) {
    // ok
    if (which_dev == 2 && p->in_handler == 0) {
      p->ticks++;
      if ((p->ticks == p->interval) && (p->interval != 0)) {
        p->in_handler = 1; // 设置为在处理函数中
        p->ticks = 0;      // 重置ticks计数
        p->alarm_trapframe = memmove(p->alarm_trapframe, p->trapframe, sizeof(*(p->trapframe)));
        p->trapframe->epc = (uint64)p->handler;
      }
    }
  }
```
- 将 `handler` 写入 `p->trapframe->epc` ，使得从 `usertrap` 返回时开始执行 `handler`
- 将整个 `trapframe` 保存至 `alarm_trapframe`


在 `kernel/sysproc.c` 中添加 `sigalarm` 和 `sigreturn` 的实现：
```c
uint64 sys_sigalarm(void) {
  int ticks;
  uint64 handler;
  if (argint(0, &ticks) < 0 || argaddr(1, &handler) < 0)
    return -1;
  struct proc *p = myproc();
  if (ticks < 0)
    return -1;
  p->interval = ticks;              // 设置警报间隔
  p->handler = (void (*)())handler; // 设置警报处理函数
  return 0;
}

uint64 sys_sigretrun(void) {
  struct proc *p = myproc();
  memmove(p->trapframe, p->alarm_trapframe, sizeof(*p->alarm_trapframe));
  p->in_handler = 0; // 重置为不在处理函数中
  return 0;
}
```


### 2.2 优化

由于处理函数 `periodic` 的逻辑非常简答，不会修改其它的用户寄存器，因此不需要保存全部的用户寄存器，而是仅保存几个重要的寄存器。

`proc`：
```c
int interval;             // 警报间隔
void (*handler)();        // 处理函数指针，无返回值和参数传入
int ticks;                // 距离上次警报的ticks数
int in_handler;           // 是否在处理函数中
uint64 alarm_epc;         // 保存用户程序的epc
uint64 alarm_sp;          // 保存sp
uint64 alarm_ra;          // 返回地址
uint64 alarm_a0;          // 参数
```

`usertrap`：
```c
} else if ((which_dev = devintr()) != 0) {
    // ok
    if (which_dev == 2 && p->in_handler == 0) {
      p->ticks++;
      if ((p->ticks == p->interval) && (p->interval != 0)) {
        p->in_handler = 1; // 设置为在处理函数中
        p->ticks = 0;      // 重置ticks计数
        p->alarm_epc = p->trapframe->epc;
        p->alarm_sp = p->trapframe->sp;
        p->alarm_a0 = p->trapframe->a0;
        p->alarm_ra = p->trapframe->ra;
        p->trapframe->epc = (uint64)p->handler;
      }
    }
  }
```

`sysproc`：

```c
uint64 sys_sigalarm(void) {
  int ticks;
  uint64 handler;
  if (argint(0, &ticks) < 0 || argaddr(1, &handler) < 0)
    return -1;
  struct proc *p = myproc();
  if (ticks < 0)
    return -1;
  p->interval = ticks;              // 设置警报间隔
  p->handler = (void (*)())handler; // 设置警报处理函数
  return 0;
}

uint64 sys_sigretrun(void) {
  struct proc *p = myproc();
  p->trapframe->epc = p->alarm_epc;
  p->trapframe->sp = p->alarm_sp;
  p->trapframe->ra = p->alarm_ra;
  p->trapframe->a0 = p->alarm_a0;
  p->in_handler = 0; // 重置为不在处理函数中
  return 0;
}
```



## 3 参考

- [Xiao Fan](https://fanxiao.tech/posts/2021-03-02-mit-6s081-notes/#55-lab-4-traps)
