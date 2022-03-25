#  Go 中栈操作

> 本篇文章转载于luozhiyun的博客：[https://www.luozhiyun.com](https://www.luozhiyun.com/)

## 知识点

### LInux 进程在内存布局

![Linux_stack](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325172213.png)

多任务操作系统中的每个进程都在自己的内存沙盒中运行。在32位模式下，它总是4GB内存地址空间，内存分配是分配虚拟内存给进程，当进程真正访问某一虚拟内存地址时，操作系统通过触发缺页中断，在物理内存上分配一段相应的空间再与之建立映射关系，这样进程访问的虚拟内存地址，会被自动转换变成有效物理内存地址，便可以进行数据的存储与访问了。



**Kernel space**：操作系统内核地址空间；

**Stack**：栈空间，是用户存放程序临时创建的局部变量，栈的增长方向是从高位地址到地位地址向下进行增长。在现代主流机器架构上（例如`x86`）中，栈都是向下生长的。然而，也有一些处理器（例如`B5000`）栈是向上生长的，还有一些架构（例如`System Z`）允许自定义栈的生长方向，甚至还有一些处理器（例如`SPARC`）是循环栈的处理方式；

**Heap**：堆空间，堆是用于存放进程运行中被动态分配的内存段，它的大小并不固定，可动态扩张或缩减；

**BBS segment**：BSS段，存放的是全局或者静态数据，但是存放的是全局/静态未初始化数据；

**Data segment：**数据段，通常是指用来存放程序中已初始化的全局变量的一块内存区域；

**Text segment**：代码段，指用来存放程序执行代码的一块内存区域。这部分区域的大小在程序运行前就已经确定，并且内存区域属于只读。



### 栈的相关概念

> 调用栈`call stack`，简称栈，是一种栈数据结构，用于存储有关计算机程序的活动 subroutines 信息。在计算机编程中，subroutines 是执行特定任务的一系列程序指令，打包为一个单元。
>
> 栈帧`stack frame`又常被称为帧`frame`是在调用栈中储存的函数之间的调用关系，每一帧对应了函数调用以及它的参数数据。

有了函数调用自然就要有调用者 caller 和被调用者 callee ，如在 函数 A 里 调用 函数 B，A 是 caller，B 是 callee。

调用者与被调用者的栈帧结构如下图所示：

![Stack layout](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325172217.png)

Go 语言的汇编代码中栈寄存器解释的非常模糊，我们大概只要知道两个寄存器 BP 和 SP 的作用就可以了：

BP：**基准指针寄存器**，维护当前栈帧的基准地址，以便用来索引变量和参数，就像一个锚点一样，在其它架构中它等价于帧指针`FP`，只是在x86架构下，变量和参数都可以通过SP来索引；

SP：**栈指针寄存器**，总是指向栈顶；



### Goroutine 栈操作

![G Stack](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325172220.png)

在 Goroutine 中有一个 stack 数据结构，里面有两个属性 lo 与 hi，描述了实际的栈内存地址：

- stack.lo：栈空间的低地址；
- stack.hi：栈空间的高地址；



在 Goroutine 中会通过 stackguard0 来判断是否要进行栈增长：

- stackguard0：`stack.lo + StackGuard`, 用于stack overlow的检测；
- StackGuard：保护区大小，常量Linux上为 928 字节；
- StackSmall：常量大小为 128 字节，用于小函数调用的优化；
- StackBig：常量大小为 4096 字节；

根据被调用函数栈帧的大小来判断是否需要扩容：

1. 当栈帧大小（FramSzie）小于等于 StackSmall（128）时，如果 SP **小于** stackguard0 那么就执行栈扩容；
2. 当栈帧大小（FramSzie）大于 StackSmall（128）时，就会根据公式 `SP - FramSzie + StackSmall` 和 stackguard0 比较，如果**小于** stackguard0 则执行扩容；
3. 当栈帧大小（FramSzie）大于StackBig（4096）时，首先会检查 stackguard0 是否已转变成 StackPreempt 状态了；然后根据公式 `SP-stackguard0+StackGuard <= framesize + (StackGuard-StackSmall)`判断，如果是 true 则执行扩容；

需要注意的是，由于栈是由高地址向低地址增长的，所以对比的时候，都是小于才执行扩容，这里需要大家品品。

当执行栈扩容时，会在内存空间中分配更大的栈内存空间，然后将旧栈中的所有内容复制到新栈中，并修改指向旧栈对应变量的指针重新指向新栈，最后销毁并回收旧栈的内存空间，从而实现栈的动态扩容。