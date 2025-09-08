---
title: CS144 Checkpoint0
description: '实现get_URL请求Web数据，和TCP层次结构中的ByteStream。'
pubDatetime: 2025-09-05 17:51:12
tags: ['CS144']
comment: true
---

整个 cs144 的实验结构层次图如下：
```css
应用层程序
   │
[ TCPSocket ]   ← 提供 connect/read/write 接口
   │
[ TCPConnection ] ← 整体状态机，协调发送方和接收方
   ├─ [ TCPSender ]   ← 分片、发送、重传
   └─ [ TCPReceiver ] ← 重排、确认、窗口
        │
   [ Reassembler ]   ← 拼接乱序片段
        │
   [ ByteStream ]    ← 有限容量的字节缓冲
```

实验的顺序为层次图从低到高，本实验中需要实现 `ByteStream`。

这里其实有个小疑问：既然 `Socket` 继承于 `FileDescritpion`，其中已经实现了文件的读写，为什么还要在底层实验 `ByteStream`，而不是直接在 `Socket` 中封装对应 API 和状态？
其实是因为 CS144 为了教学目的，将这些功能拎出来进行封装，使得层次更加清晰；而在真实的操作系统中，以上功能是封装在一起的，也不需要增加一个 `ByteStream`

在 Linux 内核中的层次结构如下：
```perl
┌───────────────────────────────┐
│        应用层程序 (用户态)      │
│  read/write, send/recv, etc.  │
└───────────────┬───────────────┘
                │
        系统调用接口 (syscall)
                │
┌───────────────▼────────────────┐
│     Socket 内核对象 (黑盒)       │  ← Linux 内核实现
│  struct socket / tcp_sock       │
│                                 │
│  - API 封装 (send/recv)         │
│  - TCP 状态机 (ESTABLISHED...)  │
│  - 序号空间 (SND.NXT/RCV.NXT)   │
│  - 定时器、RTT 估算             │
│  - 发送缓冲区 sndbuf            │
│  - 接收缓冲区 rcvbuf            │
│  - 乱序重组、ACK、重传          │
└───────────────┬────────────────┘
                │
                ▼
        TCP/IP 协议栈 (内核实现)
                │
                ▼
         网络接口/驱动/硬件

```


## webget

首先来看文件 `file_descriptor.hh`，其中使用 `FDWrapper` 来保存fd及其状态信息，而 `FileDescriptor` 提供了对外的操作接口并通过 `shared_ptr` 来管理 `FDWrapper`，多个 `FileDescriptor` 可共享同一个fd(`FDWrapper`)，内部增加其引用计数。

在 socket.hh 中，`Socket` 继承了 `FileDescriptor`，证明 `Socket` 本身就是一个文件描述符 fd，使用 `FileDescriptor` 管理fd声明周期，并在 `Socket` 中封装了 socket 相关的操作。
该文件中声明了许多类：

1. `DatagramSocket`：一个抽象层，封装了面向数据报（与之相反的是面向连接的 TCP）的 socket 操作，因为最后几个 class 都是面向数据报的，所以在这里添加一个抽象层用于继承
2. `UDPSocket`：UPD socket，直接继承的 `DatagramSocket`
3. `TCPSocket`：TCP socket，继承于 `Socket`，并提供一些面向连接的 API，如 `listen()` 和 `accept()`
4. 一些继承于 `DatagramSocket` 的 socket

`webget.cc` 中 `get_URL()` 实现：

```cpp
// 该文件的目的是使用TCP套接字连接到Web服务器并获取一个URL。

void get_URL(const string& host, const string& path)

{
  // cerr << "Function called: get_URL(" << host << ", " << path << ")\n";
  // cerr << "Warning: get_URL() has not been implemented yet.\n";

  // 1. 与web建立连接
  // 这里并没有调用bind()来绑定本地地址，因为客户端的内核会自动进行隐式绑定
  TCPSocket client;
  client.connect(Address(host, "http"));

  // 2. 组装请求报文
  string msg;
  msg += "GET " + path + " HTTP/1.1\r\n";
  msg += "Host: " + host + "\r\n";
  msg += "Connection: close\r\n"; // 非持续连接
  msg += "\r\n";                  // 报文要以\r\n结尾

  // 3. 发送请求报文
  client.write(msg);

  // 4. 循环获取响应报文，直到找到EOF
  string resp;
  while (!client.eof()) {
    resp.clear();
    client.read(resp);
    cout << resp;
  }
}
```

## ByteStream

这个实现也很简单，就是设立缓冲区，实现 `ByteStream`。

在 `byte_stream.hh` 中添加维护字段：

```cpp
protected:
  // Please add any additional state to the ByteStream here, and not to the Writer and Reader interfaces.
  uint64_t capacity_;
  bool error_ {};
  std::string buffer_;
  int bytes_pushed_;
  int bytes_popped_;
  bool closed_;
```

byte_stream.cc 中实现：

```cpp
#include "byte_stream.hh"
#include <cstdint>
#include <string_view>

using namespace std;

ByteStream::ByteStream(uint64_t capacity)
  : capacity_(capacity), buffer_(), bytes_pushed_(0), bytes_popped_(0), closed_(false)
{}

void Writer::push(string data)
{
  if (closed_) {
    return;
  }
  
  uint64_t available = available_capacity();
  if (data.size() > available) {
    data.resize(available);
  }

  buffer_.append(data);
  bytes_pushed_ += data.size(); // 这里不能加available，因为data长度可能没有溢出
}

void Writer::close()
{
  closed_ = true;
}

bool Writer::is_closed() const
{
  return closed_;
}

  

uint64_t Writer::available_capacity() const
{
  return capacity_ - buffer_.size();
}

uint64_t Writer::bytes_pushed() const
{
  return bytes_pushed_;
}

string_view Reader::peek() const
{
  return string_view(buffer_);
}

void Reader::pop(uint64_t len)
{
  if (len > buffer_.size()) {
    set_error();
  }
  
  buffer_.erase(0, len);
  bytes_popped_ += len;
}

bool Reader::is_finished() const
{
  return closed_ && buffer_.size() == 0;
}
  
uint64_t Reader::bytes_buffered() const
{
  return buffer_.size();
}

uint64_t Reader::bytes_popped() const
{
  return bytes_popped_;
}
```
