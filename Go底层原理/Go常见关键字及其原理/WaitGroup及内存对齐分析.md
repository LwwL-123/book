# WaitGroup及内存对齐分析

>WaitGroup内存对齐网络讲解都十分不清晰不能让人信服，本文内存对齐参考多篇文章总结而来。如有问题请指出



WaitGroup是Golang应用开发过程中经常使用的并发控制技术。

WaitGroup，可理解为Wait-Goroutine-Group，即等待一组goroutine结束。比如某个goroutine需要等待其他几个goroutine全部完成，那么使用WaitGroup可以轻松实现。

下面程序展示了一个goroutine等待另外两个goroutine结束的例子：

```go
package main

import (
    "fmt"
    "time"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    wg.Add(2) //设置计数器，数值即为goroutine的个数
    go func() {
        //Do some work
        time.Sleep(1*time.Second)

        fmt.Println("Goroutine 1 finished!")
        wg.Done() //goroutine执行结束后将计数器减1
    }()

    go func() {
        //Do some work
        time.Sleep(2*time.Second)

        fmt.Println("Goroutine 2 finished!")
        wg.Done() //goroutine执行结束后将计数器减1
    }()

    wg.Wait() //主goroutine阻塞等待计数器变为0
    fmt.Printf("All Goroutine finished!")
}
```

简单的说，上面程序中wg内部维护了一个计数器：

1. 启动goroutine前将计数器通过Add(2)将计数器设置为待启动的goroutine个数。
2. 启动goroutine后，使用Wait()方法阻塞自己，等待计数器变为0。
3. 每个goroutine执行结束通过Done()方法将计数器减1。
4. 计数器变为0后，阻塞的goroutine被唤醒。

其实WaitGroup也可以实现一组goroutine等待另一组goroutine，这有点像玩杂技，很容出错，如果不了解其实现原理更是如此。实际上，WaitGroup的实现源码非常简单。



# 2. 基础知识

## 2.1 信号量

信号量是Unix系统提供的一种保护共享资源的机制，用于防止多个线程同时访问某个资源。

可简单理解为信号量为一个数值：

- 当信号量>0时，表示资源可用，获取信号量时系统自动将信号量减1；
- 当信号量==0时，表示资源暂不可用，获取信号量时，当前线程会进入睡眠，当信号量为正时被唤醒；

由于WaitGroup实现中也使用了信号量，在此做个简单介绍。



## 解析

```go
type noCopy struct{}

type WaitGroup struct {
    // 避免复制使用的一个技巧，可以告诉vet工具违反了复制使用的规则
    noCopy noCopy
    // 一个复合值，用来表示waiter数、计数值、信号量
    state1 [3]uint32
}
// 获取state的地址和信号量的地址
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        // 如果地址是64bit对齐的，数组前两个元素做state，后一个元素做信号量
        return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
    } else {
        // 如果地址是32bit对齐的，数组后两个元素用来做state，它可以用来做64bit的原子操作，第一个元素32bit用来做信号量
        return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
    }
}
```



state1是个长度为3的数组，其中包含了state和一个信号量，而state实际上是两个计数器：

- counter： 当前还未执行结束的goroutine计数器
- waiter count: 等待goroutine-group结束的goroutine数量，即有多少个等候者
- semaphore: 信号量

考虑到字节是否对齐，三者出现的位置不同，为简单起见，依照字节已对齐情况下，三者在内存中的位置如下所示：

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173154.png)

# 3. WaitGroup中state方法的内存对齐

## 3.1 数据结构

在讲之前需要注意的是noCopy是一个空的结构体，大小为0，不需要做内存对齐，所以大家在看的时候可以忽略这个字段。

在WaitGroup里面，使用了uint32的数组来构造state1字段，然后根据系统的位数的不同构造不同的返回值，下面我面先来说说怎么通过sate1这个字段构建waiter数、计数值、信号量的。

首先`unsafe.Pointer`来获取state1的地址值然后转换成uintptr类型的，然后判断一下这个地址值是否能被8整除，这里通过地址 mod 8的方式来判断地址是否是64位对齐。



因为有内存对齐的存在，在64位架构里面WaitGroup结构体state1起始的位置肯定是64位对齐的，所以在64位架构上用state1前两个元素并成uint64来表示state，state1最后一个元素表示semap；

![image-20220113111712028](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173157.png)

那么64位架构上面获取state1的时候能不能第一个元素表示semap，后两个元素拼成64位返回呢？

答案自然是不可以，因为uint32的对齐保证是4bytes，64位架构中一次性处理事务的一个固定长度是8bytes，如果用state1的后两个元素表示一个64位字的字段的话CPU需要读取内存两次，不能保证原子性。



但是在32位架构里面，一个字长是4bytes，要操作64位的数据分布在**两个数据块**中，需要两次操作才能完成访问。如果两次操作中间有可能别其他操作修改，不能保证原子性。

同理32位架构想要原子性的操作8bytes，需要由调用方保证其数据地址是64位对齐的，否则原子访问会有异常，我们在这里https://golang.org/pkg/sync/atomic/#pkg-note-BUG可以看到描述：

> On ARM, x86-32, and 32-bit MIPS, it is the caller’s responsibility to arrange for 64-bit alignment of 64-bit words accessed atomically. The first word in a variable or in an allocated struct, array, or slice can be relied upon to be 64-bit aligned.



所以为了保证64位字对齐，只能让变量或开辟的结构体、数组和切片值中的第一个64位字可以被认为是64位字对齐。但是在使用WaitGroup的时候会有嵌套的情况，不能保证总是让WaitGroup存在于结构体的第一个字段上，所以我们需要增加填充使它能对齐64位字。

在32位架构中，WaitGroup在初始化的时候，分配内存地址的时候是随机的，所以WaitGroup结构体state1起始的位置不一定是64位对齐，可能会是：`uintptr(unsafe.Pointer(&wg.state1))%8 = 4`，如果出现这样的情况，那么就需要用state1的第一个元素做padding，用state1的后两个元素合并成uint64来表示statep。

![image-20220113112143629](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173200.png)



### 内存对齐总结

首先go语言需要兼容32位和64位平台，但是在32位平台上对64字节的uint操作可能不是原子的，比如在读取一个字长度的时候，另外一个字的数据很有可能已经发生改变了(在32位操作系统上,字长是4，而uint64长度为8)， 所以在实际计数的时候，其实sync.WaitGroup也就使用了4个字节来进行

在cpu内有一个cache line的缓存，这个缓存通常是8个字节的长度，在intel的cpu中，会保证针对一个cache line的操作是原子，如果只有8个字节很有可能会出现上面的这种情况，即垮了两个cache line, 这样不论是在原子操作还是性能上可能都会有问题

![image.png](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173204.png)

也就是说，如果在64位的系统中，如果semap放在前面的话，由于读取步长是8bytes，会出现以下情况

| semap | statep.0 | statep.1 | XXXX |
| ----- | -------- | -------- | ---- |

64位架构中一次性处理事务的一个固定长度是8bytes，如果用state1的后两个元素表示一个64位字的字段的话CPU需要读取内存两次，不能保证原子性



而在32位系统中，读取步长是4byte，也就是可能出现 uintptr(unsafe.Pointer(&wg.state1))%8 == 4的情况，所以这时，我们要先把semap放入，后让cpu读取statep，这样就能保证CPU原子性的读取statep了

| XXXX | semap | statep.0 | statep.1 |
| ---- | ----- | -------- | -------- |



## 3.2 Add(delta int)

Add()做了两件事，一是把delta值累加到counter中，因为delta可以为负值，也就是说counter有可能变成0或负值，所以第二件事就是当counter值变为0时，跟据waiter数值释放等量的信号量，把等待的goroutine全部唤醒，如果counter变为负值，则panic.

```go
func (wg *WaitGroup) Add(delta int) {
    // 获取状态值
    statep, semap := wg.state()
    ...
    // 高32bit是计数值v，所以把delta左移32，增加到计数上
    state := atomic.AddUint64(statep, uint64(delta)<<32)
    // 获取计数器的值
    v := int32(state >> 32)
    // 获取waiter的值
    w := uint32(state)
    ...
    // 任务计数器不能为负数
    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    // wait不等于0说明已经执行了Wait，此时不容许Add
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // 计数器的值大于或者没有waiter在等待,直接返回
    if v > 0 || w == 0 {
        return
    } 
    if *statep != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // 此时，counter一定等于0，而waiter一定大于0
    // 先把counter置为0，再释放waiter个数的信号量
    *statep = 0
    for ; w != 0; w-- {
        //释放信号量，执行一次释放一个，唤醒一个等待者
        runtime_Semrelease(semap, false, 0)
    }
}
```

1. add方法首先会调用state方法获取statep、semap的值。statep是一个uint64类型的值，高32位用来记录add方法传入的delta值之和；低32位用来表示调用wait方法等待的goroutine的数量，也就是waiter的数量。如下：

<img src="https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173209.svg" alt="Group 15" style="zoom:200%;" />

1. add方法会调用`atomic.AddUint64`方法将传入的delta左移32位，也就是将counter加上delta的值；
2. 因为计数器counter可能为负数，所以int32来获取计数器的值，waiter不可能为负数，所以使用uint32来获取；
3. 接下来就是一系列的校验，v不能小于零表示任务计数器不能为负数，否则会panic；w不等于，并且v的值等于delta表示wait方法先于add方法执行，此时也会panic，因为waitgroup不允许调用了Wait方法后还调用add方法；
4. v大于零或者w等于零直接返回，说明这个时候不需要释放waiter，所以直接返回；
5. `*statep != state`到了这个校验这里，状态只能是waiter大于零并且counter为零。当waiter大于零的时候是不允许再调用add方法，counter为零的时候也不能调用wait方法，所以这里使用state的值和内存的地址值进行比较，查看是否调用了add或者wait导致state变动，如果有就是非法调用会引起panic；
6. 最后将statep值重置为零，然后释放所有的waiter；