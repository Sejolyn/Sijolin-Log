---
title: MIT6.S081 Lab thread
description: '实现线程切换，使用多线程加速程序运行，并实现一个屏障(barrier)。'
pubDatetime: 2025-08-09 14:25:15
tags: ['MIT6.S081']
comment: true
---

## 1 Uthread: switching between threads

目的是实现线程的创建和切换。

在 `uthread_switch.S`中保存和恢复上下文（仿照 `swtch.S`）：
```asm
thread_switch:

/* YOUR CODE HERE */
	sd ra, 0(a0)
	sd sp, 8(a0)
	sd s0, 16(a0)
	sd s1, 24(a0)
	sd s2, 32(a0)
	sd s3, 40(a0)
	sd s4, 48(a0)
	sd s5, 56(a0)	
	sd s6, 64(a0)	
	sd s7, 72(a0)
	sd s8, 80(a0)
	sd s9, 88(a0)
	sd s10, 96(a0)
	sd s11, 104(a0)
	
	ld ra, 0(a1)
	ld sp, 8(a1)
	ld s0, 16(a1)
	ld s1, 24(a1)
	ld s2, 32(a1)
	ld s3, 40(a1)
	ld s4, 48(a1)
	ld s5, 56(a1)
	ld s6, 64(a1)
	ld s7, 72(a1)
	ld s8, 80(a1)
	ld s9, 88(a1)
	ld s10, 96(a1)
	ld s11, 104(a1)
	
	ret /* return to ra */
```



在 `uthread.c` 的 `struct thread` 中添加 `struct context`：

```c
struct thread {
	char stack[STACK_SIZE]; /* the thread's stack */
	int state; /* FREE, RUNNING, RUNNABLE */
	
	struct context context;
};
```



在 `thread_create()` 中初始化线程的栈和返回地址：

```c
void thread_create(void (*func)()) {
	struct thread *t;
	
	for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
		if (t->state == FREE)
			break;
	}
	
	t->state = RUNNABLE;
	// YOUR CODE HERE
	t->context.ra = (uint64) func;
	t->context.sp = (uint64) t->stack + STACK_SIZE;
}
```

这里实际上是在该进程的内存空间（xv6 中一个进程中运行一个线程）中**显式声明了一块内存区域作为该线程的栈**。在 `thread_create()` 中进行初始化后，之后运行 `thread_switch()` 会恢复上下文，从而达到在指定栈运行线程函数的效果。



在 `thread_schedule` 中调用 `thread_switch()`：

```c
thread_switch((uint64) &t->context, (uint64) &next_thread->context);
```





## 2 Using threads

目的是使用锁来解决 `put` 中存在的竞争条件。

如果两个线程同时 `put()` 同一个桶，那么可能会出现：
- 线程 A 检查 `table[i]`，发现 `key` 不存在，准备插入
- 线程 B 检查 `table[i]`，发现 `key` 不存在，也准备插入
- 线程 A 执行 `insert()` ，新节点被插入到桶链表头部
- 线程 B 执行 `insert()`，如果此时线程 A 还未更新链表头，那么线程 A 的节点将被线程 B 覆盖
这导致对应的 key 并没有被插入，被 `get()` 归为 `missing`



那么为什么不可能是 `put()` 和 `get()` 发生竞争条件？
因为在 `main()` 中 `put()` 和 `get()` 的执行是**严格分离的两个阶段**，只有在执行完 `put()` 后才会执行 `get()`，因此这两个函数不会发生竞争条件。



通过对 `put()` 加锁来实现其原子性：

```c
pthread_mutex_t locks[NBUCKET]; // 为每个桶增加锁

// 在 main() 中初始化锁
for (int i = 0; i < NBUCKET; ++i) {
	pthread_mutex_init(&locks[i], NULL);
}

// 在 put() 中使用锁
static void put(int key, int value) {
	int i = key % NBUCKET;
	
	// is the key already present?
	struct entry* e = 0;
	pthread_mutex_lock(&locks[i]);
	for (e = table[i]; e != 0; e = e->next) {
		if (e->key == key)
			break;
	}
	if (e) {
		// update the existing key.
		e->value = value;
	} else {
		// the new is new.
		insert(key, value, &table[i], table[i]);
	}
	pthread_mutex_unlock(&locks[i]);
}
```
- 检查和插入之间同样存在竞争，因此临界区必须将其全部覆盖



由于 `main()` 中的顺序执行，因此不需要为 `get()` 加锁。






## 3 Barrier

目的是实现 `barrier()`，用于同步所有线程。

感觉目的很明确，逻辑也比前两题简单：
```c
static void barrier() {
	// YOUR CODE HERE
	//
	// Block until all threads have called barrier() and
	// then increment bstate.round.
	//
	pthread_mutex_lock(&bstate.barrier_mutex);
	bstate.nthread++;
	if (bstate.nthread == nthread) {
		bstate.round++;
		bstate.nthread = 0;
		pthread_cond_broadcast(&bstate.barrier_cond);
	} else {
		pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
	}
	pthread_mutex_unlock(&bstate.barrier_mutex);
}
```