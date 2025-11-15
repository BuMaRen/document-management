---
title: "Gmp"
date: 2025-11-15T23:32:28+08:00
description : "Go语言的GMP模型（摘自Go源码）"
tags: ["go", "goroutine", "thread", "processor", "scheduler"]
image : "/img/posts/2025/20251115/gmp.jpg"
---

# Go scheduler

Go 调度器的作用就是将已经准备好运行的 goroutine 分发给工作线程去执行。

涉及的几个主要的概念：  
* `G` - goroutine
* `M` - 工作线程
* `P` - processor（处理器），这是执行 Go 代码必要的资源

`M` 必须绑定 `P` 才能执行 Go 代码，但在 `M` 被阻塞或进入系统调用（如read、write、网络IO）的时候可以没有绑定的 `P`，更多详情可参考[设计文档](https://golang.org/s/go11sched)。

# 工作线程的停放和唤醒

这套机制需要在充分利用硬件的并行性能的同时停放多余 M 以防资源浪费。这存在两个难点。

## 调度器的状态是分布式的

每个 P 都有自己的运行队列，调度器无法感知各个 P 的队列中所有的 G 的数量。这样做的好处是可以减少锁的处理，实现高并发，缺点在于系统无法快速判断当前的 M 中是否有 G 进而停放对应的 M。

## 无法预测 goroutine 何时准备就绪

如果一个 M 刚被停放，马上就有新的 G 准备就绪了，这时候就要唤醒 M，这会引起上下文切换。因此调度器只能基于当前的信息做一个“足够好”的决策。

# 废弃的调度策略

有一些方案能工作，但是表现不佳。

## Centralize all scheduler state
    
集中统计 G/M/P 的状态。这个方案要求一把全局锁，所有的 M 都在竞争这把锁，会降低并发度。

## Direct goroutine handoff

当 G 准备就绪的时候，唤醒 M 并将 G 分发给它执行。这会导致以下几个问题：
1. Thread state thrashing（线程状态抖动）

1. 
