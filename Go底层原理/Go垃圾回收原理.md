# Go垃圾回收原理

## 1.1. 定义

垃圾回收(GC)是在后台运行一个守护线程，它的作用是在监控各个对象的状态，识别并且丢弃不再使用的对象来释放和重用资源。

简单的说，垃圾回收的核心就是标记出哪些内存还在使用中(即被引用到)，哪些内存不再使用了（即未被引用），把未被引用的内存回收掉，以供后续内存分配时使用。

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223140201.png)

上图中，内存块1、2、4号位上的内存块已被分配(数字1代表已被分配，0 未分配)。变量a, b为一指针，指向内存的1、2号位。内存块的4号位曾经被使用过，但现在没有任何对象引用了，就需要被回收掉。

垃圾回收开始时从root对象开始扫描，把root对象引用的内存标记为"被引用"，考虑到内存块中存放的可能是指针，所以还需要递归的进行标记，全部标记完成后，只保留被标记的内存，未被标记的全部标识为未分配即完成了回收。



## 1.2. 垃圾回收触发时机

**内存分配量达到阀值触发GC**

每次内存分配时都会检查当前内存分配量是否已达到阀值，如果达到阀值则立即启动GC。

**阀值 = 上次GC内存分配量 \* 内存增长率**

内存增长率由环境变量GOGC控制，默认为**100，即每当内存扩大一倍时启动GC**。

**定期触发GC**

默认情况下，最长2分钟触发一次GC，这个间隔在`src/runtime/proc.go:forcegcperiod`变量中被声明：

```go
// forcegcperiod is the maximum time in nanoseconds between garbage
// collections. If we go this long without a garbage collection, one
// is forced to run.
//
// This is a variable for testing purposes. It normally doesn't change.
var forcegcperiod int64 = 2 * 60 * 1e9
Copy
```

#### 手动触发

程序代码中也可以使用`runtime.GC()`来手动触发GC。这主要用于GC性能测试和统计。



## 1.3. 垃圾回收算法

业界常见的垃圾回收算法有以下几种：

**引用计数：对每个对象维护一个引用计数，当引用该对象的对象被销毁时，引用计数减1，当引用计数器为0时回收该对象。**

- 优点：对象可以很快的被回收，不会出现内存耗尽或达到某个阀值时才回收。
- 缺点：不能很好的处理循环引用，而且实时维护引用计数，有也一定的代价。
- 代表语言：Python、PHP

**标记-清除：从根变量开始遍历所有引用的对象，引用的对象标记为"被引用"，没有被标记的进行回收。**

- 优点：解决了引用计数的缺点。
- 缺点：需要STW，即要暂时停掉程序运行。
- 代表语言：Golang(其采用三色标记法)

**分代收集：按照对象生命周期长短划分不同的代空间，生命周期长的放入老年代，而短的放入新生代，不同代有不能的回收算法和回收频率。**

- 优点：回收性能好
- 缺点：算法复杂
- 代表语言： JAVA

Golang中的垃圾回收主要应用三色标记法，GC过程和其他用户goroutine可并发运行，但需要一定时间的**STW(stop the world)**，STW的过程中，CPU不执行用户代码，全部用于垃圾回收，这个过程的影响很大，Golang进行了多次的迭代优化来解决这个问题，下面我们就来探讨一下这个问题。



## 1.4. Go V1.3之前的标记-清除(mark and sweep)算法

此算法主要有两个主要的步骤：

- 标记(Mark phase)——暂停程序业务逻辑, 找出不可达的对象，然后做上标记。
- 清除(Sweep phase)——回收标记好的对象

**第一步，暂停程序业务逻辑**

mark and sweep算法在执行的时候，需要程序暂停，即 STW(stop the world)。也就是说，这段时间程序会卡在哪儿。

<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223141854.png" alt="gc" style="zoom: 33%;" />

**第二步, 开始标记，程序找出它所有可达的对象，并做上标记。**

<img src="http://interview.wzcu.com/static/gc03.png" alt="gc" style="zoom: 40%;" />

**第三步, 标记完了之后，然后开始清除未标记的对象.**

<img src="http://interview.wzcu.com/static/gc04.png" alt="gc" style="zoom:33%;" />

**第四步, 停止暂停，让程序继续跑。然后循环重复这个过程，直到process程序生命周期结束。**



## 1.5. 标记-清扫(mark and sweep)的缺点

**STW，stop the world；让程序暂停，程序出现卡顿 (重要问题)。**

所以Go V1.3版本之前就是以上来实施的, 流程是

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223142455.png)

Go V1.3 做了简单的优化,将STW提前, 减少STW暂停的时间范围.![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223142505.png)

这里面最重要的问题就是：mark-and-sweep 算法会暂停整个程序 。

Go是如何面对并这个问题的呢？接下来Go V1.5版本 就用三色并发标记法来优化这个问题.



## 1.6. Go V1.5的三色并发标记法

三色标记法实际上就是通过三个阶段的标记来确定清除的对象都有哪些.

**第一步,就是只要是新创建的对象,默认的颜色都是标记为“白色”.**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223142603.png)

这里面需要注意的是, 所谓“程序”, 则是一些对象的根节点集合.



![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223145739.png)

**第二步, 每次GC回收开始, 然后从根节点开始遍历所有对象，把遍历到的对象从白色集合放入“灰色”集合。**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223150628.png)

**第三步, 遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223151034.png)

**第四步, 重复第三步, 直到灰色中无任何对象.**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223151111.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223151239.png)

**第五步: 回收所有的白色标记表的对象. 也就是回收垃圾.**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223152440.png)



##  1.7. 三色标记法的问题

三色并发标记法是一定要依赖STW的. 因为如果不暂停程序, 程序的逻辑改变对象引用关系, 这种动作如果在标记阶段做了修改，会影响标记结果的正确性。

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223152543.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223153200.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223153408.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223153602.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223153720.png)

可以看出，有两个问题, 在三色标记法中,是不希望被发生的

- 条件1: 一个白色对象被黑色对象引用**(白色被挂在黑色下)**
- 条件2: 灰色对象与它之间的可达关系的白色对象遭到破坏**(灰色同时丢了该白色)**

当以上两个条件同时满足时, 就会出现对象丢失现象!

当然, 如果上述中的白色对象3, 如果他还有很多下游对象的话, 也会一并都清理掉.

为了防止这种现象的发生，最简单的方式就是STW，直接禁止掉其他用户程序对对象引用关系的干扰，但是STW的过程有明显的资源浪费，对所有的用户程序都有很大影响

如何能在保证对象不丢失的情况下合理的尽可能的提高GC效率，减少STW时间呢？

答案就是, **那么我们只要使用一个机制,来破坏上面的两个条件就可以了**.



## 1.8. 屏障机制

我们让GC回收器,满足下面两种情况之一时,可保对象不丢失. 所以引出两种方式.

### 1.8.1. “强-弱” 三色不变式

#### 强三色不变式

不存在黑色对象引用到白色对象的指针。

<img src="http://interview.wzcu.com/static/gc19.png" alt="gc" style="zoom: 50%;" />



#### 弱三色不变式

<img src="http://interview.wzcu.com/static/gc20.png" alt="gc" style="zoom:50%;" />

为了遵循上述的两个方式,Golang团队初步得到了如下具体的两种屏障方式“插入屏障”, “删除屏障”.



### 1.8.2. 插入屏障

- 具体操作: 在A对象引用B对象的时候，B对象被标记为灰色。(将B挂在A下游，B必须被标记为灰色)
- 满足: **强三色不变式**. (不存在黑色对象引用白色对象的情况了， 因为白色会强制变成灰色)

我们知道,黑色对象的内存槽有两种位置, 栈和堆. 栈空间的特点是容量小,但是要求响应速度快,因为函数调用弹出频繁使用, 所以“插入屏障”机制,在栈空间的对象操作中不使用. 而仅仅使用在堆空间对象的操作中.

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223162301.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223162427.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223170110.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223170220.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223170436.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223170517.png)

但是如果栈不添加,当全部三色标记扫描之后,栈上有可能依然存在白色对象被引用的情况(如上图的对象9). 所以要对栈重新进行三色标记扫描, 但这次为了对象不丢失, 要对本次标记扫描启动STW暂停. 直到栈空间的三色标记结束.

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223170725.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223170833.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223170923.png)

最后将栈和堆空间 扫描剩余的全部 白色节点清除. 这次STW大约的时间在10~100ms间.





###  1.8.3. 删除屏障

- 具体操作: 被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。
- 满足: **弱三色不变式**. (保护灰色对象到白色对象的路径不会断)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223171102.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223171218.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223171539.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223172108.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223172535.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223172611.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223172710.png)

这种方式的回收精度低，一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮GC中被清理掉。



## 1.9. Go V1.8的混合写屏障机制

插入写屏障和删除写屏障的短板：

- **插入写屏障：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活；**
- **删除写屏障：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。**

Go V1.8版本引入了**混合写屏障机制（hybrid write barrier）**，避免了对栈re-scan的过程，极大的减少了STW的时间。结合了两者的优点。

**(1) 混合写屏障规则**

具体操作:

1. GC开始将栈上的对象全部扫描并标记为黑色(之后不再进行第二次重复扫描，无需STW)。
2. GC期间，任何在栈上创建的新对象，均为黑色。
3. 被删除的对象标记为灰色。
4. 被添加的对象标记为灰色。

满足: 变形的弱三色不变式.



**(2) 混合写屏障的具体场景分析**

GC开始：扫描栈区，将可达对象全部标记为黑

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223172947.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223173156.png)

**场景一： 对象被一个堆对象删除引用，成为栈对象的下游**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223173635.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223173716.png)



**场景二： 对象被一个栈对象删除引用，成为另一个栈对象的下游**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223173842.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223173945.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223174025.png)



**场景三：对象被一个堆对象删除引用，成为另一个堆对象的下游**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223174110.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223174209.png)

![](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223174216.png)



**场景四：对象从一个栈对象删除引用，成为另一个堆对象的下游**

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223174308.png)

![gc](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211223174354.png)

Golang中的混合写屏障满足弱三色不变式，结合了删除写屏障和插入写屏障的优点，只需要在开始时并发扫描各个goroutine的栈，使其变黑并一直保持，这个过程不需要STW，而标记结束后，因为栈在扫描后始终是黑色的，也无需再进行re-scan操作了，减少了STW的时间。



## 1.10. 总结

以上便是Golang的GC全部的标记-清除逻辑及场景演示全过程。

GoV1.3- 普通标记清除法，整体过程需要启动STW，时间在几百ms，效率极低。

GoV1.5- 三色标记法， 堆空间启动写屏障，栈空间不启动，全部扫描之后，需要重新扫描一次栈(需要STW)，时间在10ms左右，效率普通

GoV1.8-三色标记法，混合写屏障机制， 栈空间不启动，堆空间启动。整个过程几乎不需要STW，时间在0.5ms以内，效率较高。