---
title: "Context"
date: 2025-11-08T22:11:15+08:00
description : "《Go Concurrency Patterns: Context》自译"
tags: ["go", "Context"]
image : "/img/posts/2025/20251108/Context.png"
---

# Go语言并发模式：Context

*Sameer Ajmani*  
*29 July 2014*

## 介绍

在 Go servers 中，requests 一般在单独的 goroutine 中处理。Request 的 Handlers 通常会创建额外的 goroutine 以访问 RPC 或数据库之类的后端服务。处理请求的 goroutine 集合通常需要访问请求特定的值，例如最终用户的身份、授权令牌和请求的截止时间。这些正在处理的 goroutine 需要在请求被取消或者超时的时候尽快退出以便于系统回收他们占用的资源。

Google 开发了 context 包用于将 request 范围内的值、signal 等传输给所有涉及的 goroutine。这些功能打包在公共包 context 中。本文介绍如何使用并提供一些示例。

## Context

context 包的核心就是 `Context` 这个类型：

```Golang
// Context 包含了跨越 API 边界的截止时间，取消的信号和请求范围内的值(伴随请求生命周期的值)
// Context 中的方法是并发安全的
type Context interface {
    // Done 返回一个 channel，这个 channel 在 Context 被取消(比如调用了 cancel )或超时后关闭。
    Done() <-chan struct{}

    // Done 返回的 channel 被关闭后，Err可以返回关闭的原因。
    Err() error

    // Deadline 返回 Context 的截至时间（如果有）。
    Deadline() (deadline time.Time, ok bool)

    // Value 返回 key 对应存储的值，如果没有则返回 nil。
    Value(key interface{}) interface{}
}
```
（更详细的内容请参阅[godoc](https://pkg.go.dev/context)）

`Done` 方法返回的 channel 充当当前 Context 下正在执行的函数的取消信号：当这个 channel 被关闭时，当前的函数应该停止工作并返回。`Err` 方法会返回关闭的原因。更多详细内容参考[Pipelines and Cancellation](https://go.dev/blog/pipelines)。

`Context` 没有 `Cancel` 方法的原因：发出取消信号的函数和接收取消信号的函数通常不是用一个。尤其当主线程创建 goroutine 处理子任务的时候，主线程的 `Context` 不能被子任务取消。相对地，context 包提供了`WithCancel` 函数用于取消一个 `Context`。出于同样的原因，`Done` 返回的 channel 时只读的。

函数可以通过 `Deadline` 来确认 `Context` 的剩余有效时间，如果时间太小则不应该启动新的任务。同样地，代码在设置 I/O 操作的超时时间时也应该考虑 `Deadline`。

`Value` 允许 `Context` 携带请求范围的数据。该数据必须多个 goroutine 间的并发安全。

## 衍生的 Context

context 包提供了杂已有 `Context` 上衍生新的 `Context` 的方法。当父 `Context` 被取消的时候，所有衍生的 `Context` 也会被取消。所有的 `Context` 均衍生自 `Background`， `Background` 永远不会被取消：

```Golang
// Background 返回一个空的 Context。这个Context永远不会被取消, 没有截至时间, 也没有携带任何值。
// Background 通常用在 main, init, tests 和一些需要顶级 Context 的场合。
func Background() Context
```