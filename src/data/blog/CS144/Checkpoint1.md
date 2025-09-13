---
title: CS144 Checkpoint1
description: '实现TCP层次结构中的Reassembler（重组器），用于将分组拼接成字节流。'
pubDatetime: 2025-09-07 00:06:45
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

在 Checkpoint1 中，我们需要实现一个 **TCP 重组器（Reassembler）**。这个模块的主要任务，就是把可能乱序到达的分段（segment）拼接成一个连续的字节流，最终交给 `ByteStream`。

简单来说，TCP 的世界里：
- 数据传输是 **字节流** 的概念；
- 但是底层传输的时候会切割成一个个 **分段**；
- 由于网络的特性，分段可能会乱序到达、丢失、甚至重传；
- 所以接收方必须 **缓存未到位的分段**，并在合适的时候写入字节流。


## 类的设计

`Reassembler` 内部的几个重要成员：
```cpp
ByteStream output_;                     // 真正的字节流 
std::map<uint64_t, std::string> segs_;  // 缓存未组装的分段 
uint64_t unassembled_;                  // segs_ 里累计的字节数 
std::optional<uint64_t> eof_index_;     // FIN 报文对应的 EOF 位置
```

这里的核心就是一个 `map`：
- key 是分段的起始索引（first_index），
- value 是分段的字符串内容。

利用 `map` 的有序性，可以方便地处理乱序和重叠。


## insert 的主逻辑

`insert` 方法接收三个参数：
- `first_index`：子串的起始位置；
- `data`：子串内容；
- `is_last_substring`：是否是 TCP FIN 报文。

代码整体分为几个阶段：

### 1. 确定接收窗口

TCP 缓冲区是有限的，所以要限制只接收窗口范围内的数据：

```cpp
const uint64_t next_index = output_.writer().bytes_pushed();
const uint64_t win_left = next_index;
const uint64_t win_right = next_index + output_.writer().available_capacity();
```

然后根据窗口对 `data` 做裁剪：
- 窗口左边的丢掉（已经被写过的字节）；
- 窗口右边的丢掉（超过缓存能力的部分）。


### 2. 记录 EOF

TCP 里的 FIN 报文表示“数据结束”，这里用 eof 模拟。由于可能重传，`eof_index_` 只需要记录一次。
```cpp
if (is_last_substring) {
    const uint64_t logical_eof = first_index + data.size();
    if (!eof_index_.has_value()) {
        eof_index_ = logical_eof;
    }
}
```


### 3. 处理分段重叠

这是实现的核心难点。
- 首先找到第一个可能与新区间重叠的旧分段：
```cpp
auto it = segs_.lower_bound(start);
if (it != segs_.begin()) {
    auto prev_it = std::prev(it);
    if (prev_it->first + prev_it->second.size() > start) {
        it = prev_it;
    }
}
```

- 然后从左到右遍历，处理和已有分段的覆盖、缺口情况：
    - 如果旧分段完全在左边，跳过；
    - 如果有 gap，把缺口部分切出来放进缓存；
    - 如果被覆盖了，就移动到下一个。
```cpp
uint64_t pos = start;
while (pos < end && it != segs_.end()) {
    uint64_t L = it->first;
    uint64_t R = it->first + static_cast<uint64_t>(it->second.size());
  
    // 判断每个分段与当前分段的重叠情况
    // 1. R <= pos ：直接跳过，不用考虑
    if (R <= pos) {
      it++;
      continue;
    }

    // 2. L > pos：考虑 clipped 的右端是否被覆盖
    if (L > pos) {
      const uint64_t gap_end = std::min<uint64_t>(end, L);
      // 从 clipped 中裁剪出 gap
      const size_t off = static_cast<size_t>(pos - start);
      const size_t len = static_cast<size_t>(gap_end - pos);
      std::string gap = clipped.substr(off, len);
  
      // 入库
      if (!gap.empty()) {
        segs_.emplace(pos, std::move(gap));
        unassembled_ += len;
      }
      pos = gap_end;
      if (pos == end) // 整个分段被处理完成
        break;
    }

    // 3. L <= pos < R，左端被覆盖
    pos = R;
    ++it;
  }
```

这样可以避免重复存储字节。


### 4. 处理尾部缺口

可能存在新分段超出已有缓存范围的情况，需要补上尾部的 gap。
```cpp
if (pos < end) {
    std::string last_gap = clipped.substr(pos - start, end - pos);
    segs_.emplace(pos, std::move(last_gap));
    unassembled_ += end - pos;
}
```


### 5. push 到 ByteStream

最后一步，就是把已经连续的部分从缓存里推送到字节流：
```cpp
while (true) {
    const uint64_t next = output_.writer().bytes_pushed();
    auto hit = segs_.find(next);
    if (hit == segs_.end()) break;

    output_.writer().push(hit->second);
    unassembled_ -= hit->second.size();
    segs_.erase(hit);
}
```

这样，`ByteStream` 就始终保持尽可能完整的前缀字节流。


### 6. 收尾关闭

当所有字节都被写入，且 `bytes_pushed == eof_index_`，就可以关闭流：
```cpp
if (eof_index_.has_value() && output_.writer().bytes_pushed() == *eof_index_) {
    output_.writer().close();
}
```


# 整体代码

`reassembler.hh`：
```cpp
class Reassembler
{
public:
  // Construct Reassembler to write into given ByteStream.
  // 维护一个字节流
  explicit Reassembler(ByteStream&& output)
    : output_(std::move(output)), segs_(), unassembled_(0), eof_index_(std::nullopt)
  {}
  
  // ...

private:
  ByteStream output_;
  std::map<uint64_t, std::string> segs_; // 未进入字节流（已接收但不连续）
  uint64_t unassembled_;                 // segs中的字节数
  std::optional<uint64_t> eof_index_;    // eof字符串索引
};
```

`reassembler.cc`：
```cpp
// 接受一个子字符串，其 first_index 代表该子串头部字节在整个字节流中的序号（这里规定序号从 0 开始）
// data 单纯代表数据，不包含头部
// is_last_substring 模拟的是 TCP FIN 报文
void Reassembler::insert(uint64_t first_index, string data, bool is_last_substring)
{
  const uint64_t next_index = output_.writer().bytes_pushed(); // 下一个要写入的索引
  // 划定接收窗口，即缓存中未被占用的部分 [win_left, win_right)
  const uint64_t win_left = next_index;
  const uint64_t win_right = next_index + output_.writer().available_capacity();
  
  // 记录eof指针位置
  // FIN报文只有一个，但是由于网络重传，其可能会被多次发送，因此这里只需记录一次
  if (is_last_substring) {
    const uint64_t logical_eof = first_index + static_cast<uint64_t>(data.size());
    if (!eof_index_.has_value()) {
      eof_index_ = logical_eof; // 只记录一次
    }
  }

  // 确定data在窗口中的位置，溢出窗口的部分直接丢弃
  // [start, end)
  uint64_t start = max<uint64_t>(first_index, win_left);
  const uint64_t end = min<uint64_t>(first_index + static_cast<uint64_t>(data.size()), win_right);
  if (start >= end) {
    /*
     * 三种情况：
     * 1. 子串在窗口左边（冗余序列）
     * 2. 子串在窗口右边（溢出序列）
     * 3. 子串为空
     * 这些情况没有字节可以接收，之所以要单独讨论是因为其可能为is_last_substring，触发收尾
     */
    if (eof_index_.has_value() && output_.writer().bytes_pushed() == *eof_index_) {
      output_.writer().close();
    }
    return;
  }

  // 裁剪字符串 [start - first_index, end - start)
  std::string clipped = data.substr(static_cast<uint64_t>(start - first_index), static_cast<uint64_t>(end - start));

  // 若存在重叠，则获取第一个与clipped重叠的分段
  // 若不存在，则默认获取后面一个分段
  auto it = segs_.lower_bound(start); // key>=start
  // 有可能被前面分段覆盖
  if (it != segs_.begin()) {
    auto prev_it = std::prev(it);
    if (prev_it->first + static_cast<uint64_t>(prev_it->second.size()) > start) {
      it = prev_it;
    }
  }

  // 可能与多个分段存在重叠，因此需要从最早的那个开始遍历
  uint64_t pos = start;
  while (pos < end && it != segs_.end()) {
    uint64_t L = it->first;
    uint64_t R = it->first + static_cast<uint64_t>(it->second.size());

    // 判断每个分段与当前分段的重叠情况
    // 1. R <= pos ：直接跳过，不用考虑
    if (R <= pos) {
      it++;
      continue;
    }

    // 2. L > pos：考虑 clipped 的右端是否被覆盖
    if (L > pos) {
      const uint64_t gap_end = std::min<uint64_t>(end, L);
      // 从 clipped 中裁剪出 gap
      const size_t off = static_cast<size_t>(pos - start);
      const size_t len = static_cast<size_t>(gap_end - pos);
      std::string gap = clipped.substr(off, len);

      // 入库
      if (!gap.empty()) {
        segs_.emplace(pos, std::move(gap));
        unassembled_ += len;
      }
      pos = gap_end;
      if (pos == end) // 整个分段被处理完成
        break;
    }

    // 3. L <= pos < R，左端被覆盖
    pos = R;
    ++it;
  }

  // 有可能clipped超出了segs现在的范围，导致其还剩余一个后置gap
  // 比如 clipped: [6, 10), segs最后一段: [7, 9), 导致 [9, 10)需要添加在最后
  if (pos < end) {
    const size_t off = static_cast<size_t>(pos - start);
    const size_t len = static_cast<size_t>(end - pos);
    std::string last_gap = clipped.substr(off, len);
    if (!last_gap.empty()) {
      segs_.emplace(pos, std::move(last_gap));
      unassembled_ += len;
    }
  }

  // 开始push连续的分段
  while (true) {
    // bytes_pushed_会一直更新，因此每次都要重新获取
    const uint64_t next = output_.writer().bytes_pushed();
    auto hit = segs_.find(next);
    if (hit == segs_.end())
      break;

    output_.writer().push(hit->second);
    unassembled_ -= hit->second.size();
    segs_.erase(hit);
  }

  // 最后只差eof时(bytes_pushed_ == *eof_index_)，可以开始关闭接收端口
  if (eof_index_.has_value() && output_.writer().bytes_pushed() == *eof_index_) {
    output_.writer().close();
  }
}
```
