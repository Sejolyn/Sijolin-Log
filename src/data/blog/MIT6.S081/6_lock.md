---
title: MIT6.S081 Lab lock
description: ‘通过更细致的数据结构和锁划分来减少锁的争用，从而提升执行效率'
pubDatetime: 2025-08-15 04:19:20
tags: ['MIT6.S081']
comment: true

---



## Memory allocator

实验要求：
- 为每个 CPU 维护一个空闲链表，每个链表配备自己的锁
- 如果某个 CPU 的空闲链表为空，另一个 CPU 的链表仍有空闲内存，则该 CPU “窃取”其他 CPU 的空闲页



首先是为每个 CPU 维护一个空闲链表，并配备锁：

```c
struct {
	struct spinlock lock;
	struct run *freelist;
} kmem[NCPU];

void kinit() {
	for (int i = 0; i < NCPU; ++i) {
		initlock(&kmem[i].lock, "kmem");
	}
	freerange(end, (void *)PHYSTOP);
}
```
- 这里将每个锁都命名为 `keme` 也是没问题的



修改 `kfree()`，使其将空闲内存分配给当前 CPU 的空闲链表：

```c
void kfree(void *pa) {
	struct run *r;
	
	if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP)
		panic("kfree");
	
	// Fill with junk to catch dangling refs.
	memset(pa, 1, PGSIZE);
	
	r = (struct run *)pa;
	
	push_off();
	int icpu = cpuid();
	pop_off();
	
	acquire(&kmem[icpu].lock);
	r->next = kmem[icpu].freelist;
	kmem[icpu].freelist = r;
	release(&kmem[icpu].lock);
}
```
- 调用 `cpuid()` 时需要禁用中断，这样才能保证其结果准确



修改 `kalloc()`，使其能够在链表为空时“窃取”别的 CPU 的内存：

```c
void *kalloc(void) {
	struct run *r;
	
	push_off();
	int icpu = cpuid();
	pop_off();  
	
	acquire(&kmem[icpu].lock);
	r = kmem[icpu].freelist;
	if (r)
		kmem[icpu].freelist = r->next;
	if (!r) {
		for (int i = 0; i < NCPU; ++i) {
			if (i == icpu) // 当前cpu
				continue;
			acquire(&kmem[i].lock);
			r = kmem[i].freelist;
			if (r) {
				kmem[i].freelist = r->next;
				release(&kmem[i].lock);
				break;
			}
			release(&kmem[i].lock);
		}
	}
	release(&kmem[icpu].lock);
	  
	if (r)
		memset((char *)r, 5, PGSIZE); // fill with junk
	return (void *)r;
}
```
- `!r` 代表当前 CPU 链表已空，需要窃取内存
- 遍历其他 CPU 获取空闲内存



## Buffer cache

实验要求：
- 用哈希表代替 LRU 双向链表，为每个桶分配锁，从而减少对整体锁的争用
- 使用 `ticks` 来寻找 LRU `buf`



首先修改 `struct buf`：

```c
struct buf {
	int valid; // has data been read from disk?
	int disk; // does disk "own" buf?
	uint dev;
	uint blockno;
	struct sleeplock lock;
	uint refcnt;
	// struct buf *prev; // LRU cache list
	struct buf *next;
	uchar data[BSIZE];
	  
	uint timestamp;
};
```
- 不需要使用 `prev`
- 增加 `timestamp` 来表示其最近被使用的时间



声明变量和数据结构：

```c
extern uint ticks;
  
#define NBUCKET 13
#define NBUF (NBUCKET * 3)
  
struct {
	struct spinlock lock;
	struct buf buf[NBUF];
} bcache;
 
struct bucket {
	struct spinlock lock;
	struct buf head;
} hashtable[NBUCKET];

uint hash(uint blockno) { return blockno % NBUCKET; }
```
- 这里放弃了原来声明的 `NBUF`，这样改可以平均桶的分配
- 全局锁依然有存在的必要，比如保护 `buf`



接着在 `binit()` 中所有的锁以及初始化哈希表：

```c
void binit(void) {
	struct buf *b;
	  
	initlock(&bcache.lock, "bcache");
	
	for (b = bcache.buf; b < bcache.buf + NBUF; b++) {
		initsleeplock(&b->lock, "buffer");
	}
	
	b = bcache.buf;
	for (int i = 0; i < NBUCKET; i++) {
		initlock(&hashtable[i].lock, "bcache_bucket");
		for (int j = 0; j < NBUF / NBUCKET; j++) {
			b->blockno = i; // hash(b) should equal to i
			b->next = hashtable[i].head.next;
			hashtable[i].head.next = b;
			b++;
		}
	}
}
```
- 这里将所有 `buf` 平均分配到每个桶



然后是核心函数 `bget()`，需要在哈希表中找到目标 `buf`，如果没有缓存的话需要分配 LRU `buf`：

```c
static struct buf *bget(uint dev, uint blockno) {
	// printf("dev: %d blockno: %d Status: ", dev, blockno);
	struct buf *b;
	
	int idx = hash(blockno);
	struct bucket *bucket = hashtable + idx;
	acquire(&bucket->lock);
	
	// Is the block already cached?
	for (b = bucket->head.next; b != 0; b = b->next) {
		if (b->dev == dev && b->blockno == blockno) {
			b->refcnt++;
			b->timestamp = ticks;
			release(&bucket->lock);
			acquiresleep(&b->lock);
			return b;
		}
	}
	  
	// Not cached.
	// Look for LRU buf in current bucket
	uint min_time = __UINT32_MAX__;
	struct buf *replace_buf = 0;
	for (b = bucket->head.next; b != 0; b = b->next) {
		if (b->refcnt == 0 && b->timestamp < min_time) {
			replace_buf = b;
			min_time = b->timestamp;
		}
	}
	if (replace_buf) {
		goto find;
	}
	
	// Try to find in other bucket.
	acquire(&bcache.lock);
	refind:
	for (b = bcache.buf; b < bcache.buf + NBUF; b++) {
		if (b->refcnt == 0 && b->timestamp < min_time) {
			replace_buf = b;
			min_time = b->timestamp;
		}
	}
	if (replace_buf) {
		// remove from old bucket
		int ridx = hash(replace_buf->blockno);
		acquire(&hashtable[ridx].lock);
		if (replace_buf->refcnt != 1) // be used in another bucket's local find between finded and acquire
		{
			release(&hashtable[ridx].lock);
			goto refind;
		}
		struct buf *pre = &hashtable[ridx].head;
		struct buf *p = hashtable[ridx].head.next;
		while (p != replace_buf) {
			pre = pre->next;
			p = p->next;
		}
		pre->next = p->next;
		release(&hashtable[ridx].lock);
		// add to current bucket
		replace_buf->next = hashtable[idx].head.next;
		hashtable[idx].head.next = replace_buf;
		release(&bcache.lock);
		goto find;
	} else {
		panic("bget: no buffers");
	}
	
	find:
	replace_buf->dev = dev;
	replace_buf->blockno = blockno;
	replace_buf->valid = 0;
	replace_buf->refcnt = 1;
	release(&bucket->lock);
	acquiresleep(&replace_buf->lock);
	return replace_buf;
}
```



这里便可以看出对哈希表和 `buf` 链表分别上锁的好处：可以直接遍历 `buf` 链表，只需要维护一个锁。如果遍历哈希表，那我会出现同时持有两个桶的锁的情况，存在两个导致死锁的风险：

1. 如果进程又遍历到当前桶，会重复获取该桶的锁
2. 如果两个进程互相获取对方所持有的锁，那么也会造成死锁。这样的话就需要固定获取锁的顺序，如先获取桶号小的锁，再获取大的



接下来是 `brelse()`，减少计数，如果引用为零的话表示空闲，更新其时间戳：

```c
void brelse(struct buf *b) {
	if (!holdingsleep(&b->lock))
		panic("brelse");
	
	releasesleep(&b->lock);
	  
	int idx = hash(b->blockno);
	
	acquire(&hashtable[idx].lock);
	b->refcnt--;
	if (b->refcnt == 0) {
		// no one is waiting for it.
		b->timestamp = ticks;
	}
	
	release(&hashtable[idx].lock);
}
```



剩余的 `bpin()` / `bunpin()` 只需更新锁的获取就行：

```c
void bpin(struct buf *b) {
	int idx = hash(b->blockno);
	acquire(&hashtable[idx].lock);
	b->refcnt++;
	release(&hashtable[idx].lock);
}

void bunpin(struct buf *b) {
	int idx = hash( b->blockno);
	acquire(&hashtable[idx].lock);
	b->refcnt--;
	release(&hashtable[idx].lock);
}
```



测试结果：

```
$ bcachetest
start test0
test0 results:
--- lock kmem/bcache stats
lock: kmem: #test-and-set 0 #acquire() 32928
lock: kmem: #test-and-set 0 #acquire() 129
lock: kmem: #test-and-set 0 #acquire() 22
lock: bcache_bucket: #test-and-set 0 #acquire() 6176
lock: bcache_bucket: #test-and-set 0 #acquire() 6186
lock: bcache_bucket: #test-and-set 0 #acquire() 6324
lock: bcache_bucket: #test-and-set 0 #acquire() 6320
lock: bcache_bucket: #test-and-set 0 #acquire() 6320
lock: bcache_bucket: #test-and-set 0 #acquire() 6310
lock: bcache_bucket: #test-and-set 0 #acquire() 4532
lock: bcache_bucket: #test-and-set 0 #acquire() 5300
lock: bcache_bucket: #test-and-set 0 #acquire() 2112
lock: bcache_bucket: #test-and-set 0 #acquire() 4118
lock: bcache_bucket: #test-and-set 0 #acquire() 2120
lock: bcache_bucket: #test-and-set 0 #acquire() 4122
lock: bcache_bucket: #test-and-set 0 #acquire() 4170
--- top 5 contended locks:
lock: virtio_disk: #test-and-set 1007951 #acquire() 1068
lock: proc: #test-and-set 56089 #acquire() 404689
lock: proc: #test-and-set 43046 #acquire() 384260
lock: proc: #test-and-set 33896 #acquire() 384248
lock: proc: #test-and-set 32820 #acquire() 384266
tot= 0
test0: OK
start test1
test1 OK
```



## 参考

刚开始做 Buffer cache 时，思路就是将哈希表集成在 `bcache` 中，并在 `bget()` 中遍历哈希表来获取 LRU `buf`。这样做不仅复杂度提高，还经常出现让人摸不得头脑的死锁和 bug，从中午写到半夜也没有通过全部测试。看了博主[星见遥](https://www.cnblogs.com/weijunji/p/xv6-study-12.html)的实现后感觉非常巧妙，邃借鉴并在此说明。





