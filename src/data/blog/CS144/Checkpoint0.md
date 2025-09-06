---
title: CS144 Checkpoint0: networking warmup
description: ''
pubDatetime: 2025-09-05 17:51:12
tags: ['CS144']
comment: true
---

## webget

首先是 `file_descriptor.hh` 中的结构，使用 `FDWrapper` 来保存fd及其状态信息，而 `FileDescriptor` 提供了对外的操作并通过 `shared_ptr` 来管理 `FDWrapper`，多个 `FileDescriptor` 可共享同一个fd(`FDWrapper`)，内部增加其引用计数。

在 `socket.hh` 中
