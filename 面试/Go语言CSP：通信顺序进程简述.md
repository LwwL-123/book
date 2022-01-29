# Go语言CSP：通信顺序进程简述

Go实现了两种并发形式，第一种是大家普遍认知的多线程共享内存，其实就是 Java或 C++ 等语言中的多线程开发；另外一种是Go语言特有的，也是Go语言推荐的 CSP（communicating sequential processes）并发模型。

CSP 并发模型是上个世纪七十年代提出的，用于描述两个独立的并发实体通过共享 channel（管道）进行通信的并发模型。

Go语言就是借用 CSP 并发模型的一些概念为之实现并发的，但是Go语言并没有完全实现了 CSP 并发模型的所有理论，仅仅是实现了 process 和 channel 这两个概念。

process 就是Go语言中的 goroutine，每个 goroutine 之间是通过 channel 通讯来实现数据共享。



这里我们要明确的是“并发不是并行”。并发更关注的是程序的设计层面，并发的程序完全是可以顺序执行的，只有在真正的多核 CPU 上才可能真正地同时运行；并行更关注的是程序的运行层面，并行一般是简单的大量重复，例如 GPU 中对图像处理都会有大量的并行运算。

为了更好地编写并发程序，从设计之初Go语言就注重如何在编程语言层级上设计一个简洁安全高效的抽象模型，让开发人员专注于分解问题和组合方案，而且不用被线程管理和信号互斥这些烦琐的操作分散精力。



在并发编程中，对共享资源的正确访问需要精确地控制，在目前的绝大多数语言中，都是通过加锁等线程同步方案来解决这一困难问题，而Go语言却另辟蹊径，它将共享的值通过通道传递（实际上多个独立执行的线程很少主动共享资源）。

并发编程的核心概念是同步通信，但是同步的方式却有多种。先以大家熟悉的互斥量 sync.Mutex 来实现同步通信，示例代码如下所示：

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var mu sync.Mutex

    go func() {
        fmt.Println("C语言中文网")
        mu.Lock()
    }()

    mu.Unlock()
}
		
```

由于 mu.Lock() 和 mu.Unlock() 并不在同一个 Goroutine 中，所以也就不满足**顺序一致性内存模型**。同时它们也没有其他的同步事件可以参考，也就是说这两件事是可以并发的。

因为可能是并发的事件，所以 main() 函数中的 mu.Unlock() 很有可能先发生，而这个时刻 mu 互斥对象还处于未加锁的状态，因而会导致运行时异常。

```go
package main

import (
    "fmt"
    "sync"
)
func main() {
    var mu sync.Mutex

    mu.Lock()
    go func() {
        fmt.Println("C语言中文网")
        mu.Unlock()
    }()

    mu.Lock()
}
```

修复的方式是在 main() 函数所在线程中执行两次 mu.Lock()，当第二次加锁时会因为锁已经被占用（不是递归锁）而阻塞，main() 函数的阻塞状态驱动后台线程继续向前执行。

当后台线程执行到 mu.Unlock() 时解锁，此时打印工作已经完成了，解锁会导致 main() 函数中的第二个 mu.Lock() 阻塞状态取消，此时后台线程和主线程再没有其他的同步事件参考，它们退出的事件将是并发的，在 main() 函数退出导致程序退出时，后台线程可能已经退出了，也可能没有退出。虽然无法确定两个线程退出的时间，但是打印工作是可以正确完成的。



使用 sync.Mutex 互斥锁同步是比较低级的做法，我们现在改用无缓存通道来实现同步：

```go
package main

import (
    "fmt"
)

func main() {
    done := make(chan int)

    go func() {
        fmt.Println("go")
        <-done
    }()

    done <- 1
} 
```

对于这种要等待 N 个线程完成后再进行下一步的同步操作有一个简单的做法，就是使用 sync.WaitGroup 来等待一组事件：

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    // 开N个后台打印线程
    for i := 0; i < 10; i++ {
        wg.Add(1)

        go func() {
            fmt.Println("go")
            wg.Done()
        }()
    }

    // 等待N个后台线程完成
    wg.Wait()
}
```

其中 wg.Add(1) 用于增加等待事件的个数，必须确保在后台线程启动之前执行（如果放到后台线程之中执行则不能保证被正常执行到）。当后台线程完成打印工作之后，调用 wg.Done() 表示完成一个事件，main() 函数的 wg.Wait() 是等待全部的事件完成。