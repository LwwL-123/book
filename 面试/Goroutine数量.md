# Goroutine



## 1. 什么是Goroutine

**Goroutine是一个由Go运行时管理的轻量级线程，一般称其为”协程“**

```go
go f(a,b,c)
```

操作系统本身无法感知到Goroutine的存在，Goroutine的操作和切换归属于”用户态“

Goroutine 由特定的调度模式来控制，以 “多路复用” 的形式运行在操作系统为 Go 程序分配的几个系统线程上。

同时创建 Goroutine 的开销很小，初始只需要 2-4k 的栈空间。Goroutine 本身会根据实际使用情况进行自伸缩，非常轻量。



## 2.调度是什么

既然有了用户态的代表 Goroutine，操作系统又看不到他。必然需要有某个东西去管理他，才能更好的运作起来。

这指的就是 Go 语言中的调度，最常见、面试最爱问的 GMP 模型。



### 调度基础知识

Go scheduler 的主要功能是针对在处理器上运行的 OS 线程分发可运行的 Goroutine，而我们一提到调度器，就离不开三个经常被提到的缩写，分别是：

- G：Goroutine，实际上我们每次调用 `go func` 就是生成了一个 G。
- P：Processor，处理器，一般 P 的数量就是处理器的核数，可以通过 `GOMAXPROCS` 进行修改。
- M：Machine，系统线程。



### 调度流程

我们以 GMP 模型的工作流程图进行简单分析，官方图如下:

![image-20211004105430928](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325170554.png)

<img src="https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325170558.png" alt="image-20211004105500505" style="zoom:50%;" />

1. 当我们执行 `go func()` 时，实际上就是创建一个全新的 Goroutine，我们称它为 G。
2. 新创建的 G 会被放入 P 的本地队列（Local Queue）或全局队列（Global Queue）中，准备下一步的动作。需要注意的一点，这里的 P 指的是创建 G 的 P。
3. 唤醒或创建 M 以便执行 G。
4. 不断地进行事件循环
5. 寻找在可用状态下的 G 进行执行任务
6. 清除后，重新进入事件循环

在描述中有提到全局和本地这两类队列，其实在功能上来讲都是用于存放正在等待运行的 G，但是不同点在于，本地队列有数量限制，不允许超过 256 个。

并且在新建 G 时，会优先选择 P 的本地队列，如果本地队列满了，则将 P 的本地队列的一半的 G 移动到全局队列。

这可以理解为调度资源的共享和再平衡。



### 窃取行为

我们可以看到图上有 steal 行为，这是用来做什么的呢，我们都知道当你创建新的 G 或者 G 变成可运行状态时，它会被推送加入到当前 P 的本地队列中。

其实当 P 执行 G 完毕后，它也会 “干活”，它会将其从本地队列中弹出 G，同时会检查当前本地队列是否为空，如果为空会随机的从其他 P 的本地队列中尝试窃取一半可运行的 G 到自己的名下。

官方图如下：

![image-20211009141808955](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325170603.png)

在这个例子中，P2 在本地队列中找不到可以运行的 G，它会执行 `work-stealing` 调度算法，随机选择其它的处理器 P1，并从 P1 的本地队列中窃取了三个 G 到它自己的本地队列中去。

至此，P1、P2 都拥有了可运行的 G，P1 多余的 G 也不会被浪费，调度资源将会更加平均的在多个处理器中流转。



## 有没有什么限制

接下来我们回到主题，思考 “goroutine 太多了，会不会有什么影响”。

在了解 GMP 的基础知识后，我们要知道**在协程的运行过程中，真正干活的 GPM 又分别被什么约束**？

煎鱼带大家分别从 GMP 来逐步分析。

### M 的限制

第一，要知道**在协程的执行中，真正干活的是 GPM 中的哪一个**？

那势必是 M（系统线程） 了，因为 G 是用户态上的东西，最终执行都是得映射，对应到 M 这一个系统线程上去运行。

那么 M 有没有限制呢？

答案是：有的。在 Go 语言中，**M 的默认数量限制是 10000**，如果超出则会报错：

```
GO: runtime: program exceeds 10000-thread limit
```

通常只有在 Goroutine 出现阻塞操作的情况下，才会遇到这种情况。这可能也预示着你的程序有问题。

若确切是需要那么多，还可以通过 `debug.SetMaxThreads` 方法进行设置。

### G 的限制

第二，那 G 呢，Goroutine 的创建数量是否有限制？

答案是：没有。但**理论上会受内存的影响**，假设一个 Goroutine 创建需要 4k（via @GoWKH）：

- 4k * 80,000 = 320,000k ≈ 0.3G内存
- 4k * 1,000,000 = 4,000,000k ≈ 4G内存

以此就可以相对计算出来一台单机在通俗情况下，所能够创建 Goroutine 的大概数量级别。

注：Goroutine 创建所需申请的 2-4k 是需要连续的内存块。

### P 的限制

第三，那 P 呢，P 的数量是否有限制，受什么影响？

答案是：有限制。**P 的数量受环境变量 `GOMAXPROCS` 的直接影响**。

环境变量 `GOMAXPROCS` 又是什么？在 Go 语言中，通过设置 `GOMAXPROCS`，用户可以调整调度中 P（Processor）的数量。

另一个重点在于，与 P 相关联的的 M（系统线程），是需要绑定 P 才能进行具体的任务执行的，因此 P 的多少会影响到 Go 程序的运行表现。

P 的数量基本是受本机的核数影响，没必要太过度纠结他。

那 P 的数量是否会影响 Goroutine 的数量创建呢？

答案是：不影响。且 Goroutine 多了少了，P 也该干嘛干嘛，不会带来灾难性问题。



# 总结

- M：有限制，默认数量限制是 10000，可调整。
- G：没限制，但受内存影响。
- P：受本机的核数影响，可大可小，不影响 G 的数量创建。

系统的瓶颈并不是GPM调度机制，也不是内存限制，而是协程起来之后对CPU的消耗

