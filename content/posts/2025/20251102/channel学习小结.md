---
title: "Channel学习小结"
date: 2025-11-02T14:33:47+08:00
tags: ["go", "channel"]
Description: "Channel学习小结"
image: "posts/2025/20251102/Channel学习小结封面.png"
---

# Channel学习小结

channel可以通过`<-`运算符接受和发送值，默认情况下channel会阻塞直到对端就绪。即对端没有在读的情况下，写操作会阻塞，没有写的情况下读操作会阻塞，这样的特性允许channel用于goroutine间的同步。

## tips

* `close`关闭已经关闭的channel会`panic`
* 写：写一个已经关闭的channel会`panic`
* 读：读一个channel会返回channel的可读状态和读取的值，当可读状态为false的情况下，读取的值不可信，读一个已经close的channel不会`panic`
* 无缓冲的channel(`make`的大小为0)中，在一个写的goroutine阻塞的时候close这个channel，那个goroutine会`panic`
* `for ... range ...`语句遍历channel的时候会反复等待读取channel，直到channel被关闭。空channel被关闭时，并不会用一个不可信的值进入for循环而会break
