---
title: "Waitgroup"
date: 2025-11-03T23:15:44+08:00
description : "WaitGroup小结"
tags: ["go", "WaitGroup"]
image : "/img/posts/2025/20251103/waitgroup.png"
---

# WaitGroup

WaitGroup可用于同步处理。通过Add添加计数，Done释放计数，Wait等待计数减少到0。一般用于等待各个goroutine结束。

```go
func worker(wg *sync.WaitGroup) {
	defer wg.Done()
    // do something
}

func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	go worker(&wg)
	wg.Wait()
}
```

## tips

go的源码中提到：
* 往计数为0的WaitGroup中Add正数的操作必须发生在Wait操作之前
* 往计数非0的WaitGroup中Add正数或负数可以发生在任何时候
* WaitGroup计数为负数的时候会panic
* 如果WaitGroup被重用来等待其他的事件，那么Add要在之前所有的Wait完成之后执行

第一条也指出了Add操作要放在goroutine外面。此外，传递WaitGroup的时候需使用引用，不能复制。
