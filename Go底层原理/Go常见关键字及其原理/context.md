# 1. context

`Context` 是 Golang 中非常有趣的设计，它与 Go 语言中的并发编程有着比较密切的关系，在其他语言中我们很难见到类似 `Context` 的东西，它不仅能够用来设置截止日期、同步『信号』还能用来传递请求相关的值。

context翻译成中文是"上下文"，即它可以控制一组呈树状结构的goroutine，每个goroutine拥有相同的上下文。

## 概述

Go 语言中的每一个请求的都是通过一个单独的 Goroutine 进行处理的，HTTP/RPC 请求的处理器往往都会启动新的 Goroutine 访问数据库和 RPC 服务，我们可能会创建多个 Goroutine 来处理一次请求，而 `Context` 的主要作用就是在不同的 Goroutine 之间同步请求特定的数据、取消信号以及处理请求的截止日期。

![img](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220118190756.png)

每一个 `Context` 都会从最顶层的 Goroutine 一层一层传递到最下层，这也是 Golang 中上下文最常见的使用方式，如果没有 `Context`，当上层执行的操作出现错误时，下层其实不会收到错误而是会继续执行下去。

<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220118191053.png" alt="image-20220118191053067" style="zoom:50%;" />

当最上层的 Goroutine 因为某些原因执行失败时，下两层的 Goroutine 由于没有接收到这个信号所以会继续工作；但是当我们正确地使用 `Context` 时，就可以在下层及时停掉无用的工作减少额外资源的消耗：

<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220118191137.png" alt="image-20220118191137684" style="zoom:50%;" />

这其实就是 Golang 中上下文的最大作用，在不同 Goroutine 之间对信号进行同步避免对计算资源的浪费，与此同时 `Context` 还能携带以请求为作用域的键值对信息。



### 接口

`Context` 其实是 Go 语言 `context` 包对外暴露的接口，该接口定义了四个需要实现的方法，其中包括：

1. `Deadline` 方法需要返回当前 `Context` 被取消的时间，也就是完成工作的截止日期；

2. `Done` 方法需要返回一个 Channel，这个 Channel 会在当前工作完成或者上下文被取消之后关闭，多次调用 `Done` 方法会返回同一个 Channel；

3. `Err` 方法会返回当前 `Context` 结束的原因，它只会在 `Done` 返回的 Channel 被关闭时才会返回非空的值；

4. 1. 如果当前 `Context` 被取消就会返回 `Canceled` 错误；
   2. 如果当前 `Context` 超时就会返回 `DeadlineExceeded` 错误；

5. `Value` 方法会从 `Context` 中返回键对应的值，对于同一个上下文来说，多次调用 `Value` 并传入相同的 `Key` 会返回相同的结果，这个功能可以用来传递请求特定的数据；

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

`context` 包中提供的 `Background`、`TODO`、`WithDeadline` 等方法就会返回实现该接口的私有结构体的，我们会在后面的小节中详细介绍它们的工作原理。



### 示例

我们可以通过一个例子简单了解一下 `Context` 是如何对信号进行同步的，在这段代码中我们创建了一个过期时间为 `1s` 的上下文，并将上下文传入 `handle` 方法，该方法会使用 `500ms` 的时间处理该『请求』：

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    go handle(ctx, 500*time.Millisecond)

    select {
    case <-ctx.Done():
        fmt.Println("main", ctx.Err())
    }
}

func handle(ctx context.Context, duration time.Duration) {
    select {
    case <-ctx.Done():
        fmt.Println("handle", ctx.Err())

    case <-time.After(duration):
        fmt.Println("process request with", duration)
    }
}
```

所以我们有足够的时间处理该『请求』，而运行上述代码时会打印出如下所示的内容：

```
$ go run context.go
process request with 500ms
main context deadline exceeded
```

『请求』被 Goroutine 正常处理没有进入超时的 `select` 分支，但是在 `main` 函数中的 `select` 却会等待 `Context` 的超时最终打印出 `main context deadline exceeded`，如果我们将处理『请求』的时间改成 `1500ms`，当前处理的过程就会因为 `Context` 到截止日期而被中止：

```
$ go run context.go
main context deadline exceeded
handle context deadline exceeded
```

两个函数都会因为 `ctx.Done()` 返回的管道被关闭而中止，也就是上下文超时。

相信这两个例子能够帮助各位读者了解 `Context` 的使用方法以及基本的工作原理 — 多个 Goroutine 同时订阅 `ctx.Done()` 管道中的消息，一旦接收到取消信号就停止当前正在执行的工作并提前返回。