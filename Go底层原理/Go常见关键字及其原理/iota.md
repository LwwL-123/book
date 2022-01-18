# iota

我们知道iota常用于const表达式中，我们还知道其值是从零开始，const声明块中每增加一行iota值自增1。

使用iota可以简化常量定义，但其规则必须要牢牢掌握，否则在我们阅读别人源码时可能会造成误解或障碍。本节我们尝试全面的总结其使用场景，另外花一小部分时间看一下其实现原理，从原理上把握可以更深刻的记忆这些规则。

# 2. 热身

按照惯例，我们看几个有意思的小例子，用于检测我们对于iota的理解是否准确。

## 2.1 题目一

下面常量定义源于GO源码，下面每个常量的值是多少？

```go
type Priority int
const (
    LOG_EMERG Priority = iota
    LOG_ALERT
    LOG_CRIT
    LOG_ERR
    LOG_WARNING
    LOG_NOTICE
    LOG_INFO
    LOG_DEBUG
)
```

题目解释：

上面代码源于日志模块，定义了一组代表日志级别的常量，常量类型为Priority，实际为int类型。

参考答案：

iota初始值为0，也即LOG_EMERG值为0，下面每个常量递增1。

## 2.2 题目二

下面代码取自Go源码，请问每个常量值是多少？

```go
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota
    starvationThresholdNs = 1e6
)
```

题目解释：

以上代码取自Go互斥锁Mutex的实现，用于指示各种状态位的地址偏移。

参考答案：

mutexLocked == 1；mutexWoken == 2；mutexStarving == 4；mutexWaiterShift == 3；starvationThresholdNs == 1000000。

## 2.3 题目三

请问每个常量值是多少？

```go
const (
    bit0, mask0 = 1 << iota, 1<<iota - 1
    bit1, mask1
    _, _
    bit3, mask3
)
```

题目解释：

以上代码取自Go官方文档。

参考答案：

bit0 == 1， mask0 == 0， bit1 == 2， mask1 == 1， bit3 == 8， mask3 == 7

# 3. 规则

很多书上或博客描述的规则是这样的：

1. iota在const关键字出现时被重置为0
2. const声明块中每新增一行iota值自增1

我曾经也这么理解，看过编译器代码后发现，其实规则只有一条：

- iota代表了const声明块的行索引（下标从0开始）

这样理解更贴近编译器实现逻辑，也更准确。除此之外，const声明还有个特点，即第一个常量必须指定一个表达式，后续的常量如果没有表达式，则继承上面的表达式。

下面再来根据这个规则看下这段代码：

```GO
const (
    bit0, mask0 = 1 << iota, 1<<iota - 1   //const声明第0行，即iota==0
    bit1, mask1                            //const声明第1行，即iota==1, 表达式继承上面的语句
    _, _                                   //const声明第2行，即iota==2
    bit3, mask3                            //const声明第3行，即iota==3
)
```

- 第0行的表达式展开即`bit0, mask0 = 1 << 0, 1<<0 - 1`，所以bit0 == 1，mask0 == 0；
- 第1行没有指定表达式继承第一行，即`bit1, mask1 = 1 << 1, 1<<1 - 1`，所以bit1 == 2，mask1 == 1；
- 第2行没有定义常量
- 第3行没有指定表达式继承第一行，即`bit3, mask3 = 1 << 3, 1<<3 - 1`，所以bit0 == 8，mask0 == 7；

# 4. 编译原理

const块中每一行在GO中使用spec数据结构描述，spec声明如下：

```GO
    // A ValueSpec node represents a constant or variable declaration
    // (ConstSpec or VarSpec production).
    //
    ValueSpec struct {
        Doc     *CommentGroup // associated documentation; or nil
        Names   []*Ident      // value names (len(Names) > 0)
        Type    Expr          // value type; or nil
        Values  []Expr        // initial values; or nil
        Comment *CommentGroup // line comments; or nil
    }
```

这里我们只关注ValueSpec.Names， 这个切片中保存了一行中定义的常量，如果一行定义N个常量，那么ValueSpec.Names切片长度即为N。

const块实际上是spec类型的切片，用于表示const中的多行。

所以编译期间构造常量时的伪算法如下：

```GO
    for iota, spec := range ValueSpecs {
        for i, name := range spec.Names {
            obj := NewConst(name, iota...) //此处将iota传入，用于构造常量
            ...
        }
    }
```

从上面可以更清晰的看出iota实际上是遍历const块的索引，每行中即便多次使用iota，其值也不会递增。