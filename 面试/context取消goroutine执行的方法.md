# context取消goroutine执行的方法

`Go`语言里每一个并发的执行单元叫做`goroutine`，当一个用`Go`语言编写的程序启动时，其`main`函数在一个单独的`goroutine`中运行。`main`函数返回时，所有的`goroutine`都会被直接打断，程序退出。除此之外如果想通过编程的方法让一个`goroutine`中断其他`goroutine`的执行，只能是通过在多个`goroutine`间通过`context`上下文对象同步取消信号的方式来实现。



## 为什么需要取消功能

简单来说，我们需要取消功能来防止系统做一些不必要的工作。

考虑以下常见的场景：一个`HTTP`服务器查询数据库并将查询到的数据作为响应返回给客户端：

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325170946.jpg)

如果一切正常，时序图将如下所示：

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325170950.png)

但是，如果客户端在中途取消了请求会发生什么？这种情况可以发生在，比如用户在请求中途关闭了浏览器。如果不支持取消功能，`HTTP`服务器和数据库会继续工作，由于客户端已经关闭所以他们工作的成果也就被浪费了。这种情况的时序图如下所示：

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325170955.png)

理想情况下，如果我们知道某个处理过程（在此示例中为HTTP请求）已停止，则希望该过程的所有下游组件都停止运行：

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325170958.jpg)

## 使用context实现取消功能

现在我们知道了应用程序为什么需要取消功能，接下来我们开始探究在`Go`中如何实现它。因为“取消事件”与正在执行的操作高度相关，因此很自然地会将它与上下文捆绑在一起。

取消功能需要从两方面实现才能完成：

- 监听取消事件
- 发出取消事件

### 监听取消事件

`Go`语言`context`标准库的`Context`类型提供了一个`Done()`方法，该方法返回一个类型为`<-chan struct{}`的`channel`。每次`context`收到取消事件后这个`channel`都会接收到一个`struct{}`类型的值。所以在`Go`语言里监听取消事件就是等待接收`<-ctx.Done()`。

举例来说，假设一个`HTTP`服务器需要花费两秒钟来处理一个请求。如果在处理完成之前请求被取消，我们想让程序能立即中断不再继续执行下去：

```go
func main() {
    // 创建一个监听8000端口的服务器
    http.ListenAndServe(":8000", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        // 输出到STDOUT展示处理已经开始
        fmt.Fprint(os.Stdout, "processing request\n")
   		  // 通过select监听多个channel
        select {
        case <-time.After(2 * time.Second):
        // 如果两秒后接受到了一个消息后，意味请求已经处理完成
        // 我们写入"request processed"作为响应
            w.Write([]byte("request processed"))
        case <-ctx.Done():
        // 如果处理完成前取消了，在STDERR中记录请求被取消的消息
            fmt.Fprint(os.Stderr, "request cancelled\n")
        }
    }))
}
```

你可以通过运行服务器并在浏览器中打开`localhost:8000`进行测试。如果你在2秒钟前关闭浏览器，则应该在终端窗口上看到“request cancelled”字样。



### 发出取消事件

如果你有一个可以取消的操作，则必须通过`context`发出取消事件。可以通过`context`包的`WithCancel`函数返回的取消函数来完成此操作（`withCancel`还会返回一个支持取消功能的上下文对象）。该函数不接受参数也不返回任何内容，当需要取消上下文时会调用该函数，发出取消事件。

考虑有两个相互依赖的操作的情况。在这里，“依赖”是指如果其中一个失败，那么另一个就没有意义，而不是第二个操作依赖第一个操作的结果（那种情况下，两个操作不能并行）。在这种情况下，如果我们很早就知道其中一个操作失败，那么我们就会希望能取消所有相关的操作。

```go
var c = 1

func speakMemo(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("ctx.Done")
			return
		default:
			fmt.Printf("exec default func:%d\n",c)
			c++
		}
	}
}

func main() {
	rootContext := context.Background()
	ctx, cancelFunc := context.WithCancel(rootContext)

	go speakMemo(ctx)

	time.Sleep(time.Second)
	cancelFunc()
}
```



### 基于时间的取消

任何需要在请求的最大持续时间内维持SLA（服务水平协议）的应用程序，都应使用基于时间的取消。该API与前面的示例几乎相同，但有一些补充：

```go
// 这个上下文将会在3秒后被取消
// 如果需要在到期前就取消可以像前面的例子那样使用cancel函数
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)

// 上下文将在2009-11-10 23:00:00被取消
ctx, cancel := context.WithDeadline(ctx, time.Date(2009, time.November, 10, 23, 0, 0, 0, time.UTC))
```

```go
func timeout(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println("进程结束")
			return
		default:
			time.Sleep(time.Millisecond)
			fmt.Printf("我是子进程")
		}
	}
}

func main() {
	rootContext := context.Background()
	ctx,_ := context.WithTimeout(rootContext,time.Second)
	go timeout(ctx)
	time.Sleep(time.Second * 2)
}
```



## context使用上的一些陷阱

尽管`Go`中的上下文取消功能是一种多功能工具，但是在继续操作之前，你需要牢记一些注意事项。其中最重要的是，上下文只能被取消一次。如果您想在同一操作中传播多个错误，那么使用上下文取消可能不是最佳选择。使用取消上下文的场景是你实际上确实要取消某项操作，而不仅仅是通知下游进程发生了错误。 还需要记住的另一件事是，应该将相同的上下文实例传递给你可能要取消的所有函数和`goroutine`。

用`WithTimeout`或`WithCancel`包装一个已经支持取消功能的上下文将会造成多种可能会导致你的上下文被取消的情况，应该避免这种二次包装。