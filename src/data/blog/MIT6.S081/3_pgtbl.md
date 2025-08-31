---
title: MIT6.S081 Lab pgtbl
description: '探索 Xv6 页表机制，修改页表映射并打印信息。'
pubDatetime: 2025-07-25 18:48:14
tags: ['MIT6.S081']
comment: true
---

## 1 Speed up system calls

需要在创建进程是在 USYSCALL 处映射一个只读页面，在改位置存储一个 `struct usyscall`，并初始化为当前进程的 PID。
```c
#define USYSCALL (TRAPFRAME - PGSIZE)

struct usyscall {
  int pid;  // Process ID
};
```

首先是需要在 `proc.c` 的 `struct proc` 中添加 `usyscall` 变量：
```c
struct usyscall *usyscall;   // Usyscall
```

在 `allocproc()` 中为其分配物理内存，并初始化数据：
```c
// Allocate a usyscall page.
  if ((p->usyscall = (struct usyscall *)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  p->usyscall->pid = p->pid;
```


在 `proc_pagetable()` 中调用 `mappages()` 插入映射：
```c
// map the usyscall at USYSCALL
  if (mappages(pagetable, USYSCALL, PGSIZE, 
              (uint64)(p->usyscall), PTE_R | PTE_U) < 0) {
    uvmunmap(pagetable, USYSCALL, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }
```
- 注意这里需要添加 `PTE_U`，使得用户能够访问

在 `freeproc` 中释放物理页：
```c
if (p->usyscall)
    kfree((void*)p->usyscall);
  p->usyscall = 0;
```

在 `proc_freepagetable` 中解除页表映射：
```c
void proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0);
  uvmfree(pagetable, sz);
}
```


## 2 Print a page table

定义一个 `vmprint()` 的函数，接受一个 `pagetable_t` 参数，并以指定格式打印该页表。

在 `exec.c` 添加
```c
if (p->pid == 1)
   vmprint(p->pagetable);
```

在 `kernel/def.h` 中添加 `vmprint()` 的原型：
```c
void            vmprint(pagetable_t);
```

这里使用递归来实现，但是需要根据是否是最底层页表来判断是否继续向下递归，这里就需要一层判断：
如果 PTE 没有 R/W/X 权限，说明它是一个中间页表项（指向下一层页表）；反之则是最底层页表（指向实际的物理页）。

在 `vm.c` 中实现 `vmprint()` 函数：
```c
void vmprint_helper(pagetable_t pagetable, int level) {
  for (int i = 0; i < 512; ++i) {
    pte_t pte = pagetable[i];
    if ((pte & PTE_V) && (pte & (PTE_R | PTE_W | PTE_X)) == 0) { // 中间页表项
      uint64 child = PTE2PA(pte); // 下一级页表的物理地址
      for (int j = 0; j <= level; ++j) { // 根据当前level打印缩进
        printf("..");
        if (j + 1 <= level)
          printf(" ");
      }
      printf("%d: pte %p pa %p\n", i, pte, child);
      vmprint_helper((pagetable_t)child, level + 1); // 递归处理下一级页表
    } else if (pte & PTE_V) { // 指向实际的物理页（最底层页表）
      uint64 child = PTE2PA(pte);
      printf(".. .. ..%d: pte %p pa %p\n", i, pte, child);
    }
  }
}

void vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  vmprint_helper(pagetable, 0);
}
```

`PTE2PA` 在 `riscv.h` 中定义，用于物理地址（PA）和页表项（PTE）之间的转换。
```c
// shift a physical address to the right place for a PTE.
#define PA2PTE(pa) ((((uint64)pa) >> 12) << 10)
#define PTE2PA(pte) (((pte) >> 10) << 12)
```

## 3 Detecting which pages have been accessed

首先需要在 kernel/riscv. h 中定义 PTE_A。根据 RISC-V 手册，PTE_A 是第 6 位：
![[Pasted image 20250722062258.png]] 

因此代码为
```c
#define PTE_A (1L << 6) // 访问位
```

其次是在 kernel/sysproc. c 中实现 `sys_pgaccess()`：
```c
uint64 sys_pageccess(void) {
  uint64 start; // 起始虚拟地址
  int len;   // 页面数量
  uint64 mask;// 位掩码缓冲区地址

  // 获取参数
  if (argaddr(0, &start) < 0 || argint(1, &len) < 0 
    || argaddr(2, &mask) < 0)
    return -1;
  
  // 对应掩码缓冲区长度
  if (len < 1 || len > 64)
    return -1;

  struct proc *p = myproc();
  pagetable_t pagetable = p->pagetable;

  // 在内核中创建临时缓冲区存储结果
  uint64 abits = 0;

  for (int i = 0; i < len; ++i) {
    uint64 va = start + i * PGSIZE;
    pte_t *pte = walk(pagetable, va, 0); // 获取对应的PTE
    
    if (pte == 0) continue; // PTE不存在

    if (*pte & PTE_A) {
      abits |= (1 << i);
      *pte &= ~PTE_A; // 清除
    }
  }
  
  // 复制到用户空间
  if (copyout(pagetable, mask, (char *)&abits, sizeof(abits)) < 0)
    return -1;
}
```
