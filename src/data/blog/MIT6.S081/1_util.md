---
title: MIT6.S081 Lab util
description: '添加一些简单的系统调用，主要涉及进程和文件描述符的知识点。'
pubDatetime: 2025-07-17 17:52:52
tags: ['MIT6.S081']
comment: true
---



## sleep

暂停用户指定的时钟周期数。

首先是获取命令行参数（可以参考 *rm.c*），将其转化成 `int` 类型后再进行系统调用。

```c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
  // 参数少于两个，需要报错
  if (argc < 2) {
    fprintf(2, "Usage: sleep seconds\n");
    exit(1);
  }
  
  // 系统调用，但是需要将char *转化成int
  sleep(atoi(argv[1]));
  exit(0);
}
```



## pingpong

在两个进程之间传递一个字节。



关于管道的两个端口：

- p[0]：0为标准输出（把0想象成Output），因此此端为输出端口
- p[1]：1为标准输入（把1想象成Input），因此此端为输入端口



目的是通过管道实现进程之间的通信。创建两个管道，分别实现父对子通信和子对父通信，注意需要将用不到的管道关闭。

```c
#include "kernel/types.h"
#include "user/user.h"

#define READEND 0
#define WRITEEDN 1

int main() {
  int p1[2]; // 父对子
  int p2[2]; // 子对父
  char buf[1]; // 用于临时存放进程间通信的一个字节

  pipe(p1);
  pipe(p2);
  
  if (fork() == 0) {
    // child progress
    close(p1[WRITEEDN]);
    close(p2[READEND]);
    read(p1[READEND], buf, 1);
    printf("%d: received ping\n", getpid());
    write(p2[WRITEEDN], " ", 1);
    close(p1[READEND]);
    close(p2[WRITEEDN]);
  } else {
    // parent progress
    close(p1[READEND]);
    close(p2[WRITEEDN]);
    write(p1[WRITEEDN], " ", 1);
    read(p2[READEND], buf, 1);
    printf("%d: received pong\n", getpid());
    close(p1[WRITEEDN]);
    close(p2[READEND]);
  }
  exit(0);
}

```

- 子进程需要从p1[0]端读取数据，并从p2[1]端发送数据
- 父进程需要从p1[1]端输入数据，并从p2[0]端读取数据



其中需要注意的是两个进程中 `read` 和 `write` 的顺序：必须存在一个进程的 `write` 在 `read` 之前，否则会导致进程阻塞或死锁。原因如下：

- 如果两个进程都是 `read` 在前，那么由于谁都没有发送数据，双方都会卡在 `read` 这一步；
- 在其它情况下，即使进程之间的执行顺序无从得知，但无论如何都会有一个或两个进程写入了数据，最终可以读取到。



## primes

通过创建子进程和管道来筛选素数，就像一个筛网一样层层筛选：

![img](https://swtch.com/~rsc/thread/sieve.gif)

由于xv6的文件描述符和进程数量有限，所以进程的最大数量为35，同时还要及时关闭用不到的文件描述符。

采取递归的方式来实现：

```c
#include "kernel/types.h"
#include "user/user.h"

#define READEND 0
#define WRITEEND 1
#define MAXFEEDS 35

void child(int *pl); 

int main() {
  int p[2];
  pipe(p);

  if (fork() == 0) {
    child(p);
  } else {
    close(p[READEND]); // 关闭输出端口
    for (int i = 2; i <= MAXFEEDS; ++i) {
      write(p[WRITEEND], &i, sizeof(int));
    }
    close(p[WRITEEND]);
    wait((int *) 0);
  }
  exit(0);
}

#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Winfinite-recursion"

void child(int *pl) {
  int pr[2];
  int n;
  close(pl[WRITEEND]);
  
  // 输入端口关闭时，read返回0，这代表当前处于最后一层递归
  int ret = read(pl[READEND], &n, sizeof(int));
  if (ret == 0) {
    exit(0);
  }

  pipe(pr);

  if (fork() == 0) {
    child(pr);
  } else {
    close(pr[READEND]);
    printf("prime %d\n", n);
    int prime = n;
    while (read(pl[READEND], &n, sizeof(int)) != 0) {
      if (n % prime != 0) {
        write(pr[WRITEEND], &n, sizeof(int));
      }
    }
    close(pr[WRITEEND]);
    wait((int *) 0);
  }
  exit(0);
}

#pragma GCC diagnostic pop

```

- 第17行需要关闭用不到的输出端口，但是为什么不在一开始即 `if` 之前关？因为子进程会复制父进程的文件描述符，如果一开始就关那么子进程将无法读取数据；
- 由于递归的终止条件是**管道读取完毕**(`read`返回`0`)，但这是运行时行为，编译器无法预知，于是其会认为此程序无限递归从而会引发报错，所以需要在 `child`  函数的前后加上编译指示。



## find

在目录树中查找所有具有特定名称的文件，将其打印出来。此函数重点考查文件系统。



位于 *fs.h* 中的 `struct dirent`，用来描述目录条目：

```c
// 文件名的最大长度
#define DIRSIZ 14

struct dirent {
  ushort inum; // 文件的i节点号，用于唯一标识文件。inum=0为空闲条目
  char name[DIRSIZ]; // 文件名
};
```



位于 stat.h 中的 `struct stat`，用来描述文件元数据：

```c
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device

struct stat {
  int dev;     // File system's disk device
  uint ino;    // Inode number
  short type;  // Type of file
  short nlink; // Number of links to file
  uint64 size; // Size of file in bytes
};
```



依旧是通过递归实现，代码与 *ls.c* 存在大量重叠：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char *path, char *file);

int main(int argc, char *argv[]) {
  if (argc != 3) { // 只能为3个参数
    fprintf(2, "ERROR: need to pass only 2 arguments\n");
    exit(1);
  }

  find(argv[1], argv[2]);
  exit(0);
}

void find(char *path, char *file) {
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  // 打开现有文件（只读）
  if ((fd = open(path, 0)) < 0) {
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  // 将fd指定的文件元数据存储到st中
  if (fstat(fd, &st) < 0) {
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }
  
  switch (st.type) {
    case T_FILE: // 如果是文件，直接打印名称
      if (strcmp(path + strlen(path) - strlen(file), file) == 0) {
        printf("%s\n", path);
      }
      break;
    case T_DIR: // 如果是目录，需要继续递归寻找，同时维护当前路径名
      if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
        printf("find: path too long\n");
        break;
      } 
      strcpy(buf, path);
      p = buf + strlen(buf);
      *p++ = '/';
      while (read(fd, &de, sizeof(de)) == sizeof(de)) {
        if (de.inum == 0) // 空闲条目
          continue;
        memmove(p, de.name, DIRSIZ);
        p[DIRSIZ] = 0;
        if (strcmp(de.name, ".") != 0 && strcmp(de.name, "..") != 0) {
          // 子目录递归寻找
          find(buf, file);
        }
      }
      break;
  }
  close(fd); // 当前递归结束前记得要关闭文件描述符
}


```





## xargs

从标准输入读取行数据，并为每行数据执行指定命令，将该行内容作为命令参数传入。

例如命令 `echo hello too | xargs echo bye`，由于使用了管道符，因此 xargs 的标准输入为 `hello too`，而指定命令则是 `echo bye`，组合时候也就是 `echo bye hello too`。

但是需要注意这里的命令参数是以行为单位（上面的例子只有一行标准输入），因此执行 `echo "1\n2" | xargs -n 1 echo line` 实际上是执行 `echo line 1` 和 `echo line 2`（其中 `-n 1`意为只传入一个命令）。



```c++
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/param.h"

#define MAXLEN 100 // 参数的最大长度

int main(int argc, char *argv[]) {
  char *command = argv[1]; // 命令
  char bf;
  char paramv[MAXARG][MAXLEN]; // 存储行数据
  char *para[MAXARG]; 

  while (1) {
    int cnt = argc - 1;
    memset(paramv, 0, MAXARG * MAXLEN);
    // argv的第一个参数为程序本身，第二个才是命令
    for (int i = 1; i < argc - 1; ++i) {
      strcpy(paramv[i - 1], argv[i + 1]);
    }
    
    int ret;
    int cursor = 0;
    int flag = 0; // 标志位，为0时表示一个参数读取完毕

    while ((ret = read(0, &bf, 1)) > 0 && bf != '\n') {
      if (bf != ' ') {
        paramv[cnt][cursor++] = bf;
        flag = 1;
      } else if (bf == ' ' && flag == 1) {
        cnt++;
        cursor = 0;
        flag = 0;
      }
    }

    if (ret <= 0) { // 标准输入全部读取完
      break;
    }
    
    // 当前行参数已经读取完成
    for (int i = 0; i < MAXARG - 1; ++i) {
      para[i] = paramv[i];
    }
    para[MAXARG - 1] = 0;

    if (fork() == 0) {
      exec(command, para);
      exit(0);
    } else {
      wait((int *) 0);
    }
  }
  exit(0);
}


```

