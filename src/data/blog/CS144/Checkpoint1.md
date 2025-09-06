---
title: CS144 Checkpoint1
description: '实现一个重组器，将分组拼接成一个字节流。'
pubDatetime: 2025-09-07 00:06:45
tags: ['CS144']
comment: true
---


在 TCP 里，发送方会把数据切成一段一段的分组丢到网络上，接收方再想办法拼回原来的字节流。问题是网络环境并不老实，分组可能乱序、可能丢、甚至可能重复。所以我们要写一个小组件 —— **重组器 (Reassembler)**，来处理这些情况。

有以下几个要点：
- 每个分组里只有开头字节的索引，剩下的自己算
- 只能把**前面所有字节都齐了**的分组写进字节流，否则就得先存着
- 超出字节流容量范围的东西丢掉
- 如果碰到最后一段（EOF），就记下来，等写完的时候关掉流

这里使用一个 `map<uint64_t, string>` 来存还没拼进去的分组，key 就是分组起始索引，value 是数据。  
另外有两个变量：
- `unassembled_`：统计有多少字节还卡在缓存里。
- `eof_index_`：如果最后一段来了，记一下它的位置。

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
  std::map<uint64_t, std::string> segs_; // 已接收但未进入字节流中的数据
  uint64_t unassembled_;                   // segs中的字节数
  std::optional<uint64_t> eof_index_;    // eof字符串索引
};
```

`reassembler.cc`：
```cpp
// 接收一个子字符串，first_index为起始index，data为数据
// 这里的index是相对于整个数据报
void Reassembler::insert(uint64_t first_index, string data, bool is_last_substring)
{
  const uint64_t next_index = output_.writer().bytes_pushed(); // 下一个要写入的索引
  // 维护一个窗口，即data在stream中可插入的范围（裁去重叠部分和溢出部分）
  // 因为是下一个要写入的索引，所以右端为开区间 [win_left, win_right)
  // 之后对于段的右端都是开区间
  const uint64_t win_left = next_index;
  const uint64_t win_right = next_index + output_.writer().available_capacity();

  // 记录eof指针位置
  if (is_last_substring) {
    const uint64_t logical_eof = first_index + static_cast<uint64_t>(data.size());
    if (!eof_index_.has_value()) {
      eof_index_ = logical_eof;
    }
  }

  // 确定data在窗口中插入的位置
  // [start, end)
  uint64_t start = max(first_index, win_left);
  const uint64_t end = min<uint64_t>(first_index + static_cast<uint64_t>(data.size()), win_right);
  if (start >= end) {
    /*
     * 三种情况：
     * 1. 子串在窗口左边
     * 2. 子串在窗口右边（超出窗口）
     * 3. 子串为空
     * 这些情况没有字节可以接收，之所以要单独讨论是因为其可能为is_last_substring，触发收尾
     * 对于超出窗口的部分，应该直接丢弃而不是缓存在seqs中。理由如下：
     * 这里的窗口是根据capacity计算的，而capacity本身就包括了已push的字节和已缓存但未push的字节
     */
    if (eof_index_.has_value() && output_.writer().bytes_pushed() == *eof_index_) {
      output_.writer().close();
    }
    return;
  }

  // 裁剪字符串, 这里应该是对于data的相对索引，而不是整个数据报的索引 [start - first_index, end - start)
  std::string clipped = data.substr(static_cast<size_t>(start - first_index), static_cast<size_t>(end - start));

  // 将字符串统一先缓存到seqs中，这里依旧需要处理重叠的片段
  // 这里比较麻烦，因为使用map维护缓存，其key是字符串头部的index，并不连续，只能以段为单位遍历
  uint64_t pos = start;
  auto it = segs_.lower_bound(start);
  // 可能出现被前面一段覆盖的情况，比如[5, 9)覆盖7，但是此时it是后面一段
  if (it != segs_.begin()) {
    auto prev_it = std::prev(it);
    if (prev_it->first + prev_it->second.size() > start) {
      it = prev_it;
    }
  }
  
  // 迭代器的end()并不是最后一对元素，而是一个空指针
  while (pos < end && it != segs_.end()) {
    uint64_t L = it->first;
    uint64_t R = it->first + it->second.size();

    // clipped左端未被覆盖分为两种情况，一种是在左边，一种是在右边
    // 1. 在左边 R <= pos ：直接跳过，不用考虑
    // 比如 7 : [5, 7)
    if (R <= pos) {
      it++;
      continue;
    }

    // 2. 在右边 L > pos：考虑 clipped 的右端点是否被覆盖
    // 比如 7 : [8, 10)
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
      if (pos == end)
        break;
    }

    // 3. L <= pos < R，左端被覆盖
    // 比如 6 : [5, 7)，取 [6, 7)
    pos = R;
    ++it;
  }

  // 有可能clipped超出了segs现在的范围，导致其还剩余一个后置gap
  // 比如 clipped: [6, 10), segs最后一段: [7, 9), 导致 [9, 10)需要添加在最后
  if (pos < end) {
    const size_t off = static_cast<size_t>(pos - start);
    const size_t len = static_cast<size_t>(end - pos);
    std::string piece = clipped.substr(off, len);
    if (!piece.empty()) {
      segs_.emplace(pos, std::move(piece));
      unassembled_ += len;
    }
  }

  // 开始push符合条件的缓存
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

  // 最后只差eof时(bytes_pushed_ == *eof_index_)，可以开始关闭端口
  if (eof_index_.has_value() && output_.writer().bytes_pushed() == *eof_index_) {
    output_.writer().close();
  }
}

// How many bytes are stored in the Reassembler itself?
// This function is for testing only; don't add extra state to support it.
uint64_t Reassembler::count_bytes_pending() const
{
  // debug("unimplemented count_bytes_pending() called");
  return unassembled_;
}
```