# Go的互斥锁Mutex

互斥锁是并发程序中对共享资源进行访问控制的主要手段，对此Go语言提供了非常简单易用的Mutex，Mutex为一结构体类型，对外暴露两个方法Lock()和Unlock()分别用于加锁和解锁。

Mutex使用起来非常方便，但其内部实现却复杂得多，这包括Mutex的几种状态。另外，我们也想探究一下Mutex重复解锁引起panic的原因。



# 2. Mutex数据结构

## 2.1 Mutex结构体

源码包`src/sync/mutex.go:Mutex`定义了互斥锁的数据结构：

```go
type Mutex struct {
    state int32
    sema  uint32
}
```

- Mutex.state表示互斥锁的状态，比如是否被锁定等。
- Mutex.sema表示信号量，协程阻塞等待该信号量，解锁的协程释放信号量从而唤醒等待信号量的协程。



在Go的1.9版本中，为了解决等待中的 goroutine 可能会一直获取不到锁，增加了饥饿模式，让锁变得更公平，不公平的等待时间限制在 1 毫秒。



我们看到Mutex.state是32位的整型变量，内部实现时把该变量分成四份，用于记录Mutex的四种状态。

下图展示Mutex的内存布局：

![img](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220112170344.png)

- Locked: 表示该Mutex是否已被锁定，0：没有锁定 1：已被锁定。
- Woken: 表示是否有协程已被唤醒，0：没有协程唤醒 1：已有协程唤醒，正在加锁过程中。
- Starving：表示该Mutex是否处理饥饿状态， 0：没有饥饿 1：饥饿状态，说明有协程阻塞了超过1ms。
- Waiter: 表示阻塞等待锁的协程个数，协程解锁时根据此值来判断是否需要释放信号量。

协程之间抢锁实际上是抢给Locked赋值的权利，能给Locked域置1，就说明抢锁成功。抢不到的话就阻塞等待Mutex.sema信号量，一旦持有锁的协程解锁，等待的协程会依次被唤醒。

Woken和Starving主要用于控制协程间的抢锁过程，后面再进行了解。



## 2.2 Mutex方法

Mutext对外提供两个方法，实际上也只有这两个方法：

- Lock() : 加锁方法
- Unlock(): 解锁方法

下面我们分析一下加锁和解锁的过程，加锁分成功和失败两种情况，成功的话直接获取锁，失败后当前协程被阻塞，同样，解锁时跟据是否有阻塞协程也有两种处理。





# 3. 加解锁过程

## 3.1 简单加锁

假定当前只有一个协程在加锁，没有其他协程干扰，那么过程如下图所示：

![img](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220112171430.png)

加锁过程会去判断Locked标志位是否为0，如果是0则把Locked位置1，代表加锁成功。从上图可见，加锁成功后，只是Locked位置1，其他状态位没发生变化。



## 3.2 加锁被阻塞

假定加锁时，锁已被其他协程占用了，此时加锁过程如下图所示：

![img](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220112171825.png)

从上图可看到，当协程B对一个已被占用的锁再次加锁时，Waiter计数器增加了1，此时协程B将被阻塞，直到Locked值变为0后才会被唤醒。

## 3.3 简单解锁

假定解锁时，没有其他协程阻塞，此时解锁过程如下图所示：

![img](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220112171847.png)

由于没有其他协程阻塞等待加锁，所以此时解锁时只需要把Locked位置为0即可，不需要释放信号量。



## 3.4 解锁并唤醒协程

假定解锁时，有1个或多个协程阻塞，此时解锁过程如下图所示：

![img](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220112171902.png)

协程A解锁过程分为两个步骤，一是把Locked位置0，二是查看到Waiter>0，所以释放一个信号量，唤醒一个阻塞的协程，被唤醒的协程B把Locked位置1，于是协程B获得锁。



# 4. 自旋过程

加锁时，如果当前Locked位为1，说明该锁当前由其他协程持有，尝试加锁的协程并不是马上转入阻塞，而是会持续的探测Locked位是否变为0，这个过程即为自旋过程。

自旋时间很短，但如果在自旋过程中发现锁已被释放，那么协程可以立即获取锁。此时即便有协程被唤醒也无法获取锁，只能再次阻塞。

自旋的好处是，当加锁失败时不必立即转入阻塞，有一定机会获取到锁，这样可以避免协程的切换。



## 4.1 什么是自旋

自旋对应于CPU的"PAUSE"指令，CPU对该指令什么都不做，相当于CPU空转，对程序而言相当于sleep了一小段时间，时间非常短，当前实现是30个时钟周期。

自旋过程中会持续探测Locked是否变为0，连续两次探测间隔就是执行这些PAUSE指令，它不同于sleep，不需要将协程转为睡眠状态。



## 4.2 自旋条件

加锁时程序会自动判断是否可以自旋，无限制的自旋将会给CPU带来巨大压力，所以判断是否可以自旋就很重要了。

自旋必须满足以下所有条件：

- 自旋次数要足够小，通常为4，即自旋最多4次
- CPU核数要大于1，否则自旋没有意义，因为此时不可能有其他协程释放锁
- 协程调度机制中的Process数量要大于1，比如使用GOMAXPROCS()将处理器设置为1就不能启用自旋
- 协程调度机制中的可运行队列必须为空，否则会延迟协程调度

可见，自旋的条件是很苛刻的，总而言之就是不忙的时候才会启用自旋。



## 4.3 自旋的优势

自旋的优势是更充分的利用CPU，尽量避免协程切换。因为当前申请加锁的协程拥有CPU，如果经过短时间的自旋可以获得锁，当前协程可以继续运行，不必进入阻塞状态。

## 4.4 自旋的问题

如果自旋过程中获得锁，那么之前被阻塞的协程将无法获得锁，如果加锁的协程特别多，每次都通过自旋获得锁，那么之前被阻塞的进程将很难获得锁，从而进入饥饿状态。

为了避免协程长时间无法获取锁，自1.8版本以来增加了一个状态，即Mutex的Starving状态。这个状态下不会自旋，一旦有协程释放锁，那么一定会唤醒一个协程并成功加锁。



## 4.5 starvation模式

自旋过程中能抢到锁，一定意味着同一时刻有协程释放了锁，我们知道释放锁时如果发现有阻塞等待的协程，还会释放一个信号量来唤醒一个等待协程，被唤醒的协程得到CPU后开始运行，此时发现锁已被抢占了，自己只好再次阻塞，不过阻塞前会判断自上次阻塞到本次阻塞经过了多长时间，如果超过1ms的话，会将Mutex标记为"饥饿"模式，然后再阻塞。

处于饥饿模式下，不会启动自旋过程，也即一旦有协程释放了锁，那么一定会唤醒协程，被唤醒的协程将会成功获取锁，同时也会把等待计数减1。



# 5. 源码分析

## 加锁流程

### fast path

```go
func (m *Mutex) Lock() { 
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    } 
    m.lockSlow()
}
```

加锁的时候，一开始会通过CAS看一下能不能直接获取锁，如果可以的话，那么直接获取锁成功。

### lockSlow

```go
// 等待时间
var waitStartTime int64
// 饥饿标记
starving := false
// 唤醒标记
awoke := false
// 自旋次数
iter := 0
// 当前的锁的状态
old := m.state
for { 
    // 锁是非饥饿状态，锁还没被释放，尝试自旋
    if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
        if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
            atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
            awoke = true
        }
        // 自旋
        runtime_doSpin()
        // 自旋次数加1
        iter++
        // 设置当前锁的状态
        old = m.state
        continue
    }
    ...
}
```

进入到lockSlow方法之后首先会判断以下能否可以自旋，判断依据就是通过计算：

```
old&(mutexLocked|mutexStarving) == mutexLocked
```

可以知道当前锁的状态必须是上锁，并且不能处于饥饿状态，这个判断才为true，然后再看看iter是否满足次数的限制，如果都为true，那么则往下继续。

内层if包含了四个判断：

- 首先判断了awoke是不是唤醒状态；
- `old&mutexWoken == 0`为真表示没有其他正在唤醒的节点；
- `old>>mutexWaiterShift != 0`表明当前有正在等待的goroutine；
- CAS将state的mutexWoken状态位设置为`old|mutexWoken`，即为1是否成功。

如果都满足，那么将awoke状态设置为真，然后将自旋次数加一，并重新设置状态。



```go
new := old
if old&mutexStarving == 0 {
    // 如果当前不是饥饿模式，那么将mutexLocked状态位设置1，表示加锁
    new |= mutexLocked
}
if old&(mutexLocked|mutexStarving) != 0 {
    // 如果当前被锁定或者处于饥饿模式，则waiter加一，表示等待一个等待计数
    new += 1 << mutexWaiterShift
}
// 如果是饥饿状态，并且已经上锁了，那么mutexStarving状态位设置为1，设置为饥饿状态
if starving && old&mutexLocked != 0 {
    new |= mutexStarving
}
// awoke为true则表明当前线程在上面自旋的时候，修改mutexWoken状态成功
if awoke { 
    if new&mutexWoken == 0 {
        throw("sync: inconsistent mutex state")
    }
    // 清除唤醒标志位
    new &^= mutexWoken
}
```

走到这里有两种情况：1. 自旋超过了次数；2. 目前锁没有被持有。

所以第一个判断，如果当前加了锁，但是没有处于饥饿状态，也会重复设置`new |= mutexLocked`，即将mutexLocked状态设置为1；

如果是old已经是饥饿状态或者已经被上锁了，那么需要设置Waiter加一，表示这个goroutine下面不会获取锁，会等待；

如果starving为真，表示当前goroutine是饥饿状态，并且old已经被上锁了，那么设置`new |= mutexStarving`，即将mutexStarving状态位设置为1；

awoke如果在自旋时设置成功，那么在这里要`new &^= mutexWoken`消除mutexWoken标志位。因为后续流程很有可能当前线程会被挂起,就需要等待其他释放锁的goroutine来唤醒，如果unlock的时候发现mutexWoken的位置不是0，则就不会去唤醒，则该线程就无法再醒来加锁。

```go
if atomic.CompareAndSwapInt32(&m.state, old, new) {
    // 1.如果原来状态没有上锁，也没有饥饿，那么直接返回，表示获取到锁
    if old&(mutexLocked|mutexStarving) == 0 {
        break // locked the mutex with CAS
    }
    // 2.到这里是没有获取到锁，判断一下等待时长是否不为0
    // 如果不为0，那么加入到队列头部
    queueLifo := waitStartTime != 0
    // 3.如果等待时间为0，那么初始化等待时间
    if waitStartTime == 0 {
        waitStartTime = runtime_nanotime()
    }
    // 4.阻塞等待
    runtime_SemacquireMutex(&m.sema, queueLifo, 1)
    // 5.唤醒之后检查锁是否应该处于饥饿状态
    starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
    old = m.state
    // 6.判断是否已经处于饥饿状态
    if old&mutexStarving != 0 { 
        if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
            throw("sync: inconsistent mutex state")
        }
        // 7.加锁并且将waiter数减1
        delta := int32(mutexLocked - 1<<mutexWaiterShift)
        if !starving || old>>mutexWaiterShift == 1 { 
            // 8.如果当前goroutine不是饥饿状态，就从饥饿模式切换会正常模式
            delta -= mutexStarving
        }
        // 9.设置状态
        atomic.AddInt32(&m.state, delta)
        break
    }
    awoke = true
    iter = 0
} else {
    old = m.state
}
```


到这里，首先会CAS设置新的状态，如果设置成功则往下走，否则返回之后循环设置状态。设置成功之后：

1. 首先会判断old状态，如果没有饥饿，也没有获取到锁，那么直接返回，因为这种情况在进入到这段代码之前会将new状态设置为mutexLocked，表示已经获取到锁。这里还判断了一下old状态不能为饥饿状态，否则也不能获取到锁；
2. 判断waitStartTime是否已经初始化过了，如果是新的goroutine来抢占锁，那么queueLifo会返回false；如果不是新的goroutine来抢占锁，那么加入到等待队列头部，这样等待最久的 goroutine 优先能够获取到锁；
3. 如果等待时间为0，那么初始化等待时间；
4. 阻塞等待，当前goroutine进行休眠；
5. 唤醒之后检查锁是否应该处于饥饿状态，并设置starving变量值；
6. 判断是否已经处于饥饿状态，如果不处于饥饿状态，那么这里直接进入到下一个for循环中获取锁；
7. 加锁并且将waiter数减1，这里我看了一会，没用懂什么意思，其实需要分两步来理解，相当于state+mutexLocked，然后state再将waiter部分的数减一；
8. 如果当前goroutine不是饥饿状态或者waiter只有一个，就从饥饿模式切换会正常模式；
9. 设置状态；

下面用图例来解释：

这部分的图解是休眠前的操作，休眠前会根据old的状态来判断能不能直接获取到锁，如果old状态没有上锁，也没有饥饿，那么直接break返回，因为这种情况会在CAS中设置加上锁；

接着往下判断，waitStartTime是否等于0，如果不等于，说明不是第一次来了，而是被唤醒后来到这里，那么就不能直接放到队尾再休眠了，而是要放到队首，防止长时间抢不到锁；

![Group 5](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20220112180544.png)

