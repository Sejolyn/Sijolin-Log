---
title: CS144 Checkpoint2
description: '实现序号转换以及接收端的receive/send函数。'
pubDatetime: 2025-09-16 11:23:11
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

在 Checkpoint2 中，我们需要实现一个 **TCP 接收器（TCPReceiver）**。该模块主要任务如下：
1. 接收发送方的报文（Message），并且使用之前实现的 Reassembler 将其中的数据段组装成 ByteStream
2. 向发送方回复报文，其中包含 ACK number（ackno）以及当前接收窗口的空闲空间大小（用于流量控制）

## wrap/unwrap

在 Reassembler 中每个字节的序号由 64 位表示，并且序号从 0 开始，称为**绝对序号（absolute seqno）**。但是在 TCP 首部中要尽可能的压缩空间，于是使用 32 位来表示，称为**序号（seqno）**。这新增了以下机制：
1. **循环（wrap）**：相比 64 位，32 位能表示的范围非常小，如果超过最大值则进行循环处理
2. **随机初始序号（ISN）**：为了防止旧报文干扰新连接，采取随机初始序号的方式
3. **标志位**：TCP 首部的 SYN 标志位表示“请求建立连接”，而 FIN 标志位表示“请求断开连接”

![[Pasted image 20250915163109.png]]
- `zero_point = 2^32 - 2` ，代表逻辑零点，对应 TCP 的 SYN（因为 seqno 使用随机 ISN）
- stream index 才是传入重组器的参数，因为我们之前的实现没有考虑标志位

已经提前声明了 `Wrap32`（wrapping_integers.hh） 来表示报文中的序号，其中使用 `uint32_t` 来存储数据。

这里需要实现绝对序号和序号之间的转换，以便后面将其发送给 Reassembler 进行拼接。

`wrap()`：Absolute seqno -> seqno
```cpp
// Absolute seqno -> seqno
Wrap32 Wrap32::wrap(uint64_t n, Wrap32 zero_point)
{
  constexpr uint64_t MOD = 1ull << 32;
  // (isn + absolute_seqno) % 2^32
  const uint64_t sum = n + static_cast<uint64_t>(zero_point.raw_value_);
  return Wrap32(static_cast<uint32_t>(sum % MOD));
}
```

`unwrap()`：序号 -> 绝对序号
```cpp
// seqno -> Absolute seqno
uint64_t Wrap32::unwrap(Wrap32 zero_point, uint64_t checkpoint) const
{
  constexpr uint64_t MOD = 1ull << 32;

  // 计算 32 位偏移量
  const uint32_t off = this->raw_value_ - zero_point.raw_value_;
  // 对齐高 32 位
  uint64_t candidate = (checkpoint & ~(MOD - 1)) + static_cast<uint64_t>(off);

  // 判断与中点的相对位置
  if (candidate + (MOD >> 1) <= checkpoint) {
    candidate += MOD; // 靠左 -> 下一圈
  } else if (candidate > checkpoint + (MOD >> 1)) {
    if (candidate >= MOD) candidate -= MOD; // 靠右 -> 上一圈（前提是不会下溢）
    // 否则就保持当前不变
  }

  return candidate;
}
```
- 由于是从大范围映射到小范围（64 位 -> 32 位），因此会存在映射冲突
- `checkpoint` 为已知的最后一个绝对序号（相当于 `bytes_pushed_`），因此最接近 `checkpoint` 才是正确的绝对序号
- 如果直接减去 `MOD` 有可能会发生下溢，反而距离更远；加一圈并不会造成上溢

![[unwrap.excalidraw 1.png]]

![[unwrap.excalidraw]]



## TCPreceiver

报文数据结构已经提前声明，分别是 `TCPSenderMessage` 和 `TCPReceiverMessage`。

### recive

首先如果收到 `RST` 报文，那么直接设置出错：
```cpp
if (message.RST) {
    reassembler_.output_.set_error();
    return;
  }
```

如果还未收到 `SYN` 报文，那么应该忽略所有报文，因为此时连接还未建立，无法接收数据：
```cpp
if (!has_syn_) {
    // 还未收到 SYN 报文
    if (!message.SYN) return; // 忽略所有非 SYN 报文
    has_syn_ = true;
    zero_point_ = message.seqno;
  }
```


接下来就是要计算 `checkpoint`，这样才能算出段首的绝对序号：
```cpp
  // checkpoint = 1(SYN) + bytes_pushed + (如果已经结束，再+1(FIN))
  const uint64_t bytes_pushed = reassembler_.output_.writer().bytes_pushed();
  const bool ended = reassembler_.output_.reader.is_finished();
  const uint64_t checkpoint = 1 + bytes_pushed + (ended ? 1 : 0);
  
  // payload 的 absolute seqno
  const uint64_t abs_seqno = message.seqno.unwrap(zero_point_, checkpoint);
```

知道了绝对序号后，我们要将其转换为子字符串也就是 `payload` 的字节流序号：
```cpp
  // payload 的 stream index
  const uint64_t stream_index = abs_seqno - 1 + (message.SYN ? 1 : 0);
```

![[seqno2index.excalidraw.png]]


最后便是将数据推给 Reassembler，即重组器：
```cpp
  // 推给 BReassembler
  const std::string data = message.payload;
  reassembler_.insert(stream_index, data, message.FIN);
```


### send

首先是设置 RST 标志位：
```cpp
  TCPReceiverMessage out;

  // 如果收到RST报文或底层字节流出错，要将其反映在发送消息中
  const bool stream_error = reader().has_error();
  out.RST = stream_error || rst_;
```

其次是计算确认号，与先前计算 `checkpoint` 逻辑一致：
```cpp
  // ackno
  if (syn_) {
    const uint64_t bytes_pushed = writer().bytes_pushed();
    const bool ended = writer().is_closed();
    uint64_t ack_abs_seqno = 1 + bytes_pushed + (ended ? 1 : 0);
    out.ackno = Wrap32::wrap(ack_abs_seqno, zero_point_);
  } else {
    out.ackno = nullopt;
  }
```

最后是计算窗口大小：
```cpp
  // window_size
  const size_t win = static_cast<uint64_t>(writer().available_capacity());
  out.window_size = static_cast<uint16_t>(min<size_t>(win, std::numeric_limits<uint16_t>::max()));
```