---
title: MIT6.S081 Lab mmap
description: '实现mmap和munmap功能'
pubDatetime: 2025-08-18 22:18:13
tags: ['MIT6.S081']
comment: true

---



参考博客：[Xiao Fan（樊潇）](https://fanxiao.tech/posts/2021-03-02-mit-6s081-notes/#144-lab-10-mmap)



实验目的：

- 实现一个功能稍微简略的 `mmap()`，`addr` 始终为零，即由内核决定映射文件的虚拟地址
- 实现 `munmap()`，移除指定地址范围内的内存映射。如果进程已修改该内存且将其映射 `MAP_SHARED`，则应现将修改内容写入文件



首先是将 `$U/_mmaptest\` 添加到 Makefile，然后添加 `mmap()` 和 `munmap()` 系统调用，这里不再赘述。



在 proc.h 中添加对 `struct vma` 的定义：

```c
struct vma {
  int valid;
  uint64 addr;
  int length;
  int prot;
  int flags;
  struct file *mapfile;
};

// Per-process state
struct proc {
  // ...
  struct vma vmas[NVMA];       // Process vmas
};
```
- 由于默认 offset 为零，因此这里不需要声明该字段



在 param.h 中添加 `NVMA` ：

```c
#define NVMA 16 // number of process vmas
```



在 sysfile.c 中添加 `sys_mmap()`：

```c
uint64 sys_mmap(void) {
  int length, prot, flags, fd;
  struct proc *p = myproc();
  struct file *mapfile;

  // get argument
  if (argint(1, &length) < 0 || argint(2, &prot) < 0 ||
      argint(3, &flags) < 0 || argfd(4, &fd, &mapfile) < 0)
    return -1;

  // check
  length = PGROUNDDOWN(length);
  if(MAXVA - length < p->sz)
    return -1;
  if (!mapfile->readable && (prot & PROT_READ))
    return -1;
  if (!mapfile->writable && (prot & PROT_WRITE) && (flags & MAP_SHARED))
    return -1;

  // find a free vma and contain it
  for (int i = 0; i < NVMA; ++i) {
    struct vma *curvma = &p->vmas[i];
    if (!curvma->valid) {
      curvma->valid = 1;
      curvma->addr = p->sz;
      p->sz += length;
      curvma->length = length;
      curvma->flags = flags;
      curvma->prot = prot;
      curvma->mapfile = mapfile;
      filedup(mapfile);
      return curvma->addr;
    }
  }

  // no free vma
  return -1;
}
```
- 这里固定 `addr` 为 0，因此不需要获取
- `pvma[i].addr = p->sz`：将新映射的内存区域放在堆的栈顶，紧接在现有地址空间之后
- `pvma[i].valid_len = pvma[i].len`：延迟加载，初始时未分配任何物理页，但是要将其表示为已占用



接下来在 `usertrap()` 中添加对页错误的处理，实现延迟加载：

```c
} else if (r_scause() == 13 || r_scause() == 15) { // Page Fault
    uint64 va = r_stval();
    struct proc *p = myproc();
    struct vma *vmas = p->vmas;

    // check va safe
    if (va > MAXVA || va >= p->sz)
      goto exception;

    // lazy allocation
    for (int i = 0; i < NVMA; ++i) {
      struct vma *curvma = &vmas[i];
      if (curvma->valid && va >= curvma->addr &&
          va < curvma->addr + curvma->length) {
        va = PGROUNDDOWN(va);
        uint64 pa = (uint64)kalloc();
        if (pa == 0)
          goto exception;
        memset((void *)pa, 0, PGSIZE);
        ilock(curvma->mapfile->ip);
        if (readi(curvma->mapfile->ip, 0, pa, va - curvma->addr, PGSIZE) < 0) {
          iunlock(curvma->mapfile->ip);
          break;
        }
        iunlock(curvma->mapfile->ip);
        int flag = (curvma->prot << 1) | PTE_V | PTE_U;
        if (mappages(p->pagetable, va, PGSIZE, pa, flag) < 0) {
          kfree((void*)pa);
          break;
        }
        break;
      }
    } 
  } else {
      exception:
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }

    if (p->killed)
      exit(-1);

    // give up the CPU if this is a timer interrupt.
    if (which_dev == 2)
      yield();

    usertrapret();
  }
```
- `int flag = (pvma[i].prot << 1) | PTE_U | PTE_V`：这里需要将 `prot` 转换为 PTE 的权限位
	- 在 PTE 中 `PTE_R (1L << 1)、PTE_W (1L << 2)、PTE_X (1L << 3)`
	- 在 `prot` 中 `PROT_READ 0x1、PROT_WRITE 0x2、PROT_EXEC 0x4` 
	- 也就是说 `prot` 向左移动一位正好匹配 PTE 的标志位



接下来实现 `munmap()`：

```c
uint64 sys_munmap(void) {
  uint64 addr;
  int length;
  struct proc *p = myproc();
  
  // get argument
  if (argaddr(0, &addr) < 0 || argint(1, &length) < 0)
    return -1;

  // look for vma
  struct vma *vma = 0;
  int found = 0;
  for (int i = 0; i < NVMA; ++i) {
    vma = &p->vmas[i];
    if (vma->valid && addr >= vma->addr && addr < vma->addr + vma->length) {
      found = 1;
      break;
    }
  }
  
  // not found
  if (!found)
    return -1;

  addr = PGROUNDDOWN(addr);
  length = PGROUNDDOWN(length);
  if (vma->flags & MAP_SHARED) {
    // if MAP_SHARED then write back first
    if (filewrite(vma->mapfile, addr, length) < 0)
      printf("munmap: filewrite < 0\n");
  }

  // unmapped
  uvmunmap(p->pagetable, addr, length / PGSIZE, 1);

  if (addr == vma->addr) {
    if (length == vma->length) {
      // unmapped whole vma
      fileclose(vma->mapfile);
      vma->valid = 0;
      p->sz -= length;
    } else {
      // unmapped from start to middle
      vma->addr += length;
      vma->length -= length;
    } 
  } else if (addr + length == vma->addr + vma->length) {
    // unmapped from middle to end
    vma->length -= length;
  } else {
    return -1;
  }

  return 0;
}
```
- 如果进程已修改该内存且将其映射 `MAP_SHARED` ，则应先将修改内容写入文件
- 分情况讨论取消映射的范围，要么从起始处开始，要么一直到末尾，而不会在中间打洞
- `uvmunmap()` 取消映射并释放先前分配的物理内存，最后一个参数为 `1`（为 `0` 则不释放物理内存）
- `p->sz -= length`：既然是取消映射了整个 vma，这里应当更新 `p->sz`



更新 `fork()` 和 `exit()`：

```c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

	// Copy vma from parent to child
  for (int i = 0; i < NVMA; i++) {
    if (p->vmas[i].valid) {
      memmove(&np->vmas[i], &p->vmas[i], sizeof(struct vma));
      filedup(np->vmas[i].mapfile);
    }
  }

  // ...
}

// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }

  // unmap all vma
  for (int i = 0; i < NVMA; ++i) {
    if (p->vmas[i].valid) {
      if (p->vmas[i].flags & MAP_SHARED) {
        filewrite(p->vmas[i].mapfile, p->vmas[i].addr, p->vmas[i].length);
      }
      fileclose(p->vmas[i].mapfile);
      uvmunmap(p->pagetable, p->vmas[i].addr, p->vmas[i].length / PGSIZE, 1);
      p->vmas[i].valid = 0;
    }
  }

  // ...
}
```
- 这里如果是 `MAP_SHARED` 则同样需要先写入
- 使用 `fileclose()` 减少引用
- 依旧使用 `uvmunmap()` 取消映射并释放物理内存



到这一步依然存在许多 bug，导致无法通过测试。



在 `munmap()` 中存在 `unmap from start to middle` 和 `unmap from middle to end` 的情况，这就导致在 `p->sz` 以内的内存并不一定都有映射，因此可能会造成 `uvmunmap()` 和 `uvmcopy()` 的 `panic`，需要作以下修改：

```c
// uvmcopy
if((*pte & PTE_V) == 0)
     // panic("uvmcopy: page not present");
     continue;
  
// uvmunmap  
// if((*pte & PTE_V) == 0)
//   panic("uvmunmap: not mapped");
```



另外 `kfree` 会试图解放 0 这个物理内存，参考博客的作者没有给出原因，我在上一个 bug 耗费了太多时间和精力，因此也没有弄明白，直接作修改吧：

```c
void
kfree(void *pa)
{
  struct run *r;
  if (pa == 0)
    return;
```



总结：

- 这个 lab 的逻辑并不是非常难（默认 `addr` 和 `offset` 为 0，简化了逻辑）
- 后边所说的这些 bug 实属可恶，只能使用 `printf` 慢慢找错误点
- 查看了许多博客的实现，但是好多都没有涉及上述 bug 的解决，不知道是哪里有差异或是我使用的 21 年版本新出现的bug





