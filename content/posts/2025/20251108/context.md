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
*原文地址：https://go.dev/blog/context*

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

### 衍生的 Context

context 包提供了杂已有 `Context` 上衍生新的 `Context` 的方法。当父 `Context` 被取消的时候，所有衍生的 `Context` 也会被取消。所有的 `Context` 均衍生自 `Background`， `Background` 永远不会被取消：

```Golang
// Background 返回一个空的 Context。这个Context永远不会被取消, 没有截至时间, 也没有携带任何值。
// Background 通常用在 main, init, tests 和一些需要顶级 Context 的场合。
func Background() Context
```

`WithCancel` 和 `WithTimeout` 返回可以在父 `Context` 之前取消的衍生 `Context`。每个请求的 `Context` 一般在请求结束后就被取消了。

```Golang
// WithCancel 返回 parent 的副本
// 这个副本的 Done 返回的 channel 在 cancel 被调用或者 parent 被取消的时候关闭
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// A CancelFunc cancels a Context.
type CancelFunc func()

// WithTimeout 返回 parent 的副本，与 WithCancel 的差别在于该副本在超时的时候 Done 返回的 channel 也会被关闭。
// 衍生 Context 的截止时间为 min(当前时间+timeout, parent的截止时间)
// 如果超时之前调用了 cancel, 资源会被回收
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithValue` 提供了将值关联到 `Context` 上的途径：

```Golang
// WithValue 返回 parent 的副本，副本中保存了 key-val 的键制对。
func WithValue(parent Context, key interface{}, val interface{}) Context
```

了解如何使用 context 包的最佳方法是通过一个实际示例。

## 示例：Google 网页搜索

这个示例是一个 HTTP 服务器，它通过将 `golang` 转发到 [Google Web Search API](https://developers.google.com/custom-search) 并呈现结果来处理类似 `/search?q=golang&timeout=1s` 的 URL。

代码分为三部分：
* [server](https://go.dev/blog/context/server/server.go) 包含了 `main` 和 `/search` 的处理逻辑。
* [userip](https://go.dev/blog/context/userip/userip.go) 包含了从 `Context` 红解析相关参数的逻辑。
* [google](https://go.dev/blog/context/google/google.go) 提供了查询 Google 的 `Search` 函数。

### server

server 处理类似 `/search?q=golang` 的请求，并提供 Google 搜索 golang 的前几个结果。`handleSearch` 被注册用于处理 `/search` 的请求。`handleSearch` 创建一个名为 ctx 的初始 `Context`，并安排在处理完成返回时取消。如果请求包含超时的参数，则超时后 `Context` 将自动取消：

```Golang
func handleSearch(w http.ResponseWriter, req *http.Request) {
    // ctx 是这个处理流程的 Context。 
    // cancel 用于关闭 ctx.Done 返回的 channel。
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    timeout, err := time.ParseDuration(req.FormValue("timeout"))
    if err == nil {
        // 创建一个超时自动 cancel 的 Context。
        ctx, cancel = context.WithTimeout(context.Background(), timeout)
    } else {
        ctx, cancel = context.WithCancel(context.Background())
    }
    defer cancel() // 流程结束后取消根 Context。
```

通过 userip 包提取请求中包含的客户端的 IP 地址。由于后续的流程也会用到，所以将改变量添加到 `Context` 中：

```Golang
    query := req.FormValue("q")
    if query == "" {
        http.Error(w, "no query", http.StatusBadRequest)
        return
    }

    // 将 userip 添加到 Context 中
    userIP, err := userip.FromRequest(req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    ctx = userip.NewContext(ctx, userIP)
```

调用 `google.Search` 进行查询：

```Golang
    start := time.Now()
    results, err := google.Search(ctx, query)
    elapsed := time.Since(start)
```

搜索成功时会呈现结果：

```Golang
    if err := resultsTemplate.Execute(w, struct {
        Results          google.Results
        Timeout, Elapsed time.Duration
    }{
        Results: results,
        Timeout: timeout,
        Elapsed: elapsed,
    }); err != nil {
        log.Print(err)
        return
    }
```

### userip

userip 包提供了从请求中 IP 地址并将其与 `Context` 关联的函数。`Context` 提供 k-v 映射，其中 key 和 value 均为 `interface{}` 类型。 key 必须支持相等性，并且 value 必须能够安全地供多个 `goroutine` 同时使用。像 userip 这样的包隐藏了这种 k-v 的细节，并提供了对特定 `Context` 值的强类型访问。为避免 key 冲突，userip 定义了一个未导出的类型 `key`，并使用该类型的值作为 `Context` 的 key：

```Golang
// key 类型不导出，防止冲突
type key int

// userIPkey 是 userIP 地址的在 Context 中的 key。
// 它的值为零是任意的。如果此 package 定义了其他的 Context 的 key，则它们将具有不同的整数值。
const userIPKey key = 0
```

`FromRequest` 从 `http.Request` 中提取 userIP:

```Golang
func FromRequest(req *http.Request) (net.IP, error) {
    ip, _, err := net.SplitHostPort(req.RemoteAddr)
    if err != nil {
        return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
    }
```

`NewContext` 返回一个新的 `Context`，其中包含提供的 userIP：

```Golang
func NewContext(ctx context.Context, userIP net.IP) context.Context {
    return context.WithValue(ctx, userIPKey, userIP)
}
```

FromContext 从 `Context` 中提取 userIP:

```Golang
func FromContext(ctx context.Context) (net.IP, bool) {
    // 如果 ctx 没有 k-v，则 ctx.Value 返回 nil;
    // net.IP 类型断言对 nil 返回 ok=false.
    userIP, ok := ctx.Value(userIPKey).(net.IP)
    return userIP, ok
}
```

### google

`google.Search` 函数向 [Google Web Search API](https://developers.google.com/custom-search) 发出 HTTP 请求，并解析 JSON 编码的结果。它接受一个 `Context` 参数 ctx，如果在请求进行中时 `ctx.Done` 关闭，则立即返回:

```Golang
func Search(ctx context.Context, query string) (Results, error) {
    // 准备 Google Web Search API 请求。
    req, err := http.NewRequest("GET", "https://ajax.googleapis.com/ajax/services/search/web?v=1.0", nil)
    if err != nil {
        return nil, err
    }
    q := req.URL.Query()
    q.Set("q", query)

    // 如果 ctx 包含 user IP，则将其转发到 server。
    // Google API 使用 user IP 来区分服务器发起的请求和最终用户的请求。
    if userIP, ok := userip.FromContext(ctx); ok {
        q.Set("userip", userIP.String())
    }
    req.URL.RawQuery = q.Encode()
```

`Search` 使用辅助函数 `httpDo` 来发出 HTTP 请求。在请求或响应处理过程中如果 `ctx.Done` 关闭则取消请求。`Search` 将一个闭包传递给 `httpDo` 来处理 HTTP 响应：

```Golang
    var results Results
    err = httpDo(ctx, req, func(resp *http.Response, err error) error {
        if err != nil {
            return err
        }
        defer resp.Body.Close()

        // 解析 json 搜索结果.
        // https://developers.google.com/web-search/docs/#fonje
        var data struct {
            ResponseData struct {
                Results []struct {
                    TitleNoFormatting string
                    URL               string
                }
            }
        }
        if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
            return err
        }
        for _, res := range data.ResponseData.Results {
            results = append(results, Result{Title: res.TitleNoFormatting, URL: res.URL})
        }
        return nil
    })
    // httpDo 会等待我们提供的闭包返回，因此可以安全地在此处读取结果。
    return results, err
```

`httpDo` 函数会在一个新的 `goroutine` 中运行 HTTP 请求并处理其响应。如果 `ctx.Done` 在 `goroutine` 退出之前关闭，则会取消该请求：

```Golang
func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
    // Run the HTTP request in a goroutine and pass the response to f.
    c := make(chan error, 1)
    req = req.WithContext(ctx)
    go func() { c <- f(http.DefaultClient.Do(req)) }()
    select {
    case <-ctx.Done():
        <-c // Wait for f to return.
        return ctx.Err()
    case err := <-c:
        return err
    }
}
```

## Context 的适配

许多服务器框架都提供了用于承载请求范围值的包和类型。我们可以定义 `Context` 接口的新实现，使现有代码可以在新框架下运行。

例如，`Gorilla` 的 `github.com/gorilla/context` 包允许处理程序通过提供从 `HTTP 请求`到`键值对`的映射，将数据与传入的请求关联起来。在 `gorilla.go` 中，我们提供了一个 `Context` 实现，其 `Value` 方法的返回与 `Gorilla` 包中特定 HTTP 请求相关联。

其他的包也提供了类似于 `Context` 的取消方法。比如 [Tomb](https://godoc.org/gopkg.in/tomb.v2) 提供了 `Kill` 函数通过关闭一个 `Dying` 的 channel 来表示取消。`Tomb` 还提供了一些方法等待 `goroutine` 的退出，比如说 `sync.WaitGroup`。在 [tomb.go](https://go.dev/blog/context/tomb/tomb.go) 中提供了一个 `Context` 实现，当其父 `Context` 被取消或提供的 `Tomb` 被销毁时，该 `Context` 实现也会被取消。

## 结束语

Google 要求 Go 程序员将 `Context` 作为入参和返回值的第一位。这使得不同团队开发的 Go 代码能够很好地配合。它提供了对超时和取消操作的简单控制，并确保安全凭证等关键值能够正确地在 Go 程序中传递。

基于 ``Context`` 构建服务的框架应该提供相应的 `Context` 实现以便于哪些依赖 `Context` 的包使用。Client 从调用者的代码接收 `Context` 对象。通过为请求范围的数据和取消操作建立通用接口，`Context` 使包开发者能够更轻松地共享代码，从而创建可扩展的服务。
