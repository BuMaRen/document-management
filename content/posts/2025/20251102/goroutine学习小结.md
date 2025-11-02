---
title: "goroutine学习小结"
date: 2025-11-02T00:08:02+08:00
tags: ["go", "goroutine"]
Description: "goroutine学习小结"
image: "/img/posts/2025/20251102/goroutine.png"
---

# goroutine学习小结

区别于进程、线程，goroutine更像是协程，但相较协程更为轻量。

当程序中使用`go func()`之后，runtime会新建一个结构代表goroutine，为其分配初始栈，完成之后放到待运行队列中。当调度器有资源时，队列中的这个结构会被选中并绑定到一个线程上，由线程执行函数体。

上述操作均发生在用户态空间，不涉及内核态的上下文切换。

当这个goroutine函数执行完毕或者panic未被恢复导致退出，goroutine就会进入“终止”状态，对应的资源就可以被回收了。
