# 1. context

Golang context是Golang应用开发常用的并发控制技术，它与WaitGroup最大的不同点是context对于派生goroutine有更强的控制力，它可以控制多级的goroutine。

context翻译成中文是"上下文"，即它可以控制一组呈树状结构的goroutine，每个goroutine拥有相同的上下文。

典型的使用场景如下图所示：