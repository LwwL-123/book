# 深入理解 Go Interface

## 1. interface

Go中interface是什么呢？interface是一组方法集合。我们可以把它看成一种定义内部方法的动态数据类型，任意实现了这些方法的数据类型都可以认为是特定的数据类型。



在 Golang 中，interface 是一组 method 的集合，是 duck-type programming 的一种体现。不关心属性（数据），只关心行为（方法）。具体使用中你可以自定义自己的 struct，并提供特定的 interface 里面的 method 就可以把它当成 interface 来使用。下面是一种 interface 的典型用法，定义函数的时候参数定义成 interface，调用函数的时候就可以做到非常的灵活。

```go
type MyInterface interface{
    Print()
}

func TestFunc(x MyInterface) {}
type MyStruct struct {}
func (me MyStruct) Print() {}

func main() {
    var me MyStruct
    TestFunc(me)
}
```

## 2. Why Interface

Gopher China 上给出了下面上个理由：

- writing generic algorithm （泛型编程）
- hiding implementation detail （隐藏具体实现）
- providing interception points （提供切入点）

### 2.1 writing generic algorithm

严格来说，在 Golang 中并不支持泛型编程。在 C++ 等高级语言中使用泛型编程非常的简单，所以泛型编程一直是 Golang 诟病最多的地方。但是使用 interface 我们可以实现泛型编程，我这里简单说一下，具体可以参考我前面给出来的那篇文章。比如我们现在要写一个泛型算法，形参定义采用 interface 就可以了，以标准库的 sort 为例。

```go
package sort

// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package.  The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}

...

// Sort sorts data.
// It makes one call to data.Len to determine n, and O(n*log(n)) calls to
// data.Less and data.Swap. The sort is not guaranteed to be stable.
func Sort(data Interface) {
    // Switch to heapsort if depth of 2*ceil(lg(n+1)) is reached.
    n := data.Len()
    maxDepth := 0
    for i := n; i > 0; i >>= 1 {
        maxDepth++
    }
    maxDepth *= 2
    quickSort(data, 0, n, maxDepth)
}
```

Sort 函数的形参是一个 interface，包含了三个方法：`Len()`，`Less(i,j int)`，`Swap(i, j int)`。使用的时候不管数组的元素类型是什么类型（int, float, string…），只要我们实现了这三个方法就可以使用 Sort 函数，这样就实现了“泛型编程”。有一点比较麻烦的是，我们需要将数组自定义一下。下面是一个例子。

```go
type Person struct {
    Name string
    Age  int
}

func (p Person) String() string {
    return fmt.Sprintf("%s: %d", p.Name, p.Age)
}

// ByAge implements sort.Interface for []Person based on
// the Age field.
type ByAge []Person //自定义

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

func main() {
    people := []Person{
        {"Bob", 31},
        {"John", 42},
        {"Michael", 17},
        {"Jenny", 26},
    }

    fmt.Println(people)
    sort.Sort(ByAge(people))
    fmt.Println(people)
}
```

### 2.2 hiding implement detail

隐藏具体实现，这个很好理解。比如我设计一个函数给你返回一个 interface，那么你只能通过 interface 里面的方法来做一些操作，但是内部的具体实现是完全不知道的。Francesc 举了个 context 的例子。 context 最先由 google 提供，现在已经纳入了标准库，而且在原有 context 的基础上增加了：cancelCtx，timerCtx，valueCtx。语言的表达有时候略显苍白无力，看一下 context 包的代码吧。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}
```

表明上 WithCancel 函数返回的还是一个 Context interface，但是这个 interface 的具体实现是 cancelCtx struct。

```go
// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{
        Context: parent,
        done:    make(chan struct{}),
    }
}

// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
    Context     //注意一下这个地方

    done chan struct{} // closed by the first cancel call.
    mu       sync.Mutex
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}

func (c *cancelCtx) Done() <-chan struct{} {
    return c.done
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.err
}

func (c *cancelCtx) String() string {
    return fmt.Sprintf("%v.WithCancel", c.Context)
}
```

尽管内部实现上下面三个函数返回的具体 struct （都实现了 Context interface）不同，但是对于使用者来说是完全无感知的。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)    //返回 cancelCtx
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) //返回 timerCtx
func WithValue(parent Context, key, val interface{}) Context    //返回 valueCtx
```

### 2.3 providing interception points

Francesc 这里的 interception 想表达的意思我理解应该是 wrapper 或者装饰器，他给出了一个例子如下：

```go
type header struct {
    rt  http.RoundTripper
    v   map[string]string
}

func (h header) RoundTrip(r *http.Request) *http.Response {
    for k, v := range h.v {
        r.Header.Set(k,v)
    }
    return h.rt.RoundTrip(r)
}
```

通过 interface，我们可以通过类似这种方式实现 dynamic dispatch。

这种写法真的很方便，而且不用去显示的 import io package，interface 底层实现的时候会动态的检测。这样也会引入一些问题：

1. 性能下降。使用 interface 作为函数参数，runtime 的时候会动态的确定行为。而使用 struct 作为参数，编译期间就可以确定了。
2. 不知道 struct 实现哪些 interface。这个问题可以使用 guru 工具来解决。

综上，Golang interface 的这种非侵入实现真的很难说它是好，还是坏。但是可以肯定的一点是，对开发人员来说代码写起来更简单了。



## 3. interface type assertion

interface 像其他类型转换的时候一般我们称作断言，举个例子。

```go
func do(v interface{}) {
    n := v.(int)    // might panic
}
```

这样写的坏处在于：一旦断言失败，程序将会 panic。一种避免 panic 的写法是使用 type assertion。

```go
func do(v interface{}) {
    n, ok := v.(int)
    if !ok {
        // 断言失败处理
    }
}
```

对于 interface 的操作可以使用 reflect 包来处理，关于 reflect 包的原理和使用可以看下文

## 举例

### 函数传参slice

先从一个问题开始：

> 有一个函数的输入参数是slice，但是slice里面的元素类型未知，如何来定义这个函数？

听上去很简单，很多人可能会写出下面的代码。

```go
func MethodTakeinSlice(in []interface{}){...}
...
slice := []int{1,2,3}
MethodTakeinSlice(slice)
```





很遗憾，这样会得到一个错误信息：cannot use slice (type []int) as type []interface {} in argument to MethodTakeinSlice。一个简单的解决方案是写一个convert函数。

```go
func convert(in []AnyType) (out []interface{}) {
    out = make([]interface{}, len(in))
    for i, v := range in {
        out[i] = v
    }
    return
    }
```

但这样相当于泛型的优势又没了，因为每一种特定类型都得做一个转换。这里就引入了go语言的另外一个特性reflect。



### reflect

reflect，中文一般叫做反射。反射机制是指在运行时态能够调用对象的方法和属性。很多人比较熟悉的是Java的反射机制，其实go语言中也提供了反射机制，import reflect就可以使用。在go语言中，主要用在函数的参数是interface{}类型，运行时根据传入的参数的特定类型执行不同的动作。reflect针对很多数据类型都提供了一些方法以及属性。下面以Struct为例。

```go
file: src/reflect/type.go
// A StructField describes a single field in a struct.
type StructField struct {
    // Name is the field name.
    Name string
    // PkgPath is the package path that qualifies a lower case (unexported)
    // field name.  It is empty for upper case (exported) field names.
    // See https://golang.org/ref/spec#Uniqueness_of_identifiers
    PkgPath string
    
    Type      Type      // field type
    Tag       StructTag // field tag string
    Offset    uintptr   // offset within struct, in bytes
    Index     []int     // index sequence for Type.FieldByIndex
    Anonymous bool      // is an embedded field
}
```

这是reflect为Struct统一定义的属性，我们可以通过这些属性来操作struct，下面是一个简单的例子。

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    kelu := Person{"kelu", 25}
    t := reflect.TypeOf(kelu)
    n := t.NumField()
    for i := 0; i < n; i++ {
        fmt.Println(t.Field(i).Name)
        fmt.Println(t.Field(i).Type)
    }
}
//output as follow
//name
//string
//age
//int
```



### 用reflect解决上面的问题

用reflect实现上面的问题无非就是通过reflect检验传入interface{}的类型，然后操作。

```go
package main

import (
    "fmt"
    "reflect"
)

type Person struct {
    name string
    age  int
}

func Method(in interface{}) (ok bool) {
    v := reflect.ValueOf(in)
    if v.Kind() == reflect.Slice {
    	ok = true
    }
    else {
    	//panic
    }
    
    num := v.Len()
    for i := 0; i < num; i++ {
    	fmt.Println(v.Index(i).Interface())
    }
    return ok
}

func main() {
    s := []int{1, 3, 5, 7, 9}
    b := []float64{1.2, 3.4, 5.6, 7.8}
    Method(s)
    Method(b)
}
```

其中refelct.Slice表示Slice，还有其他数据类型，一共25种。



### reflect源码浅析

reflect中数据类型表示

```go
const (
    Invalid Kind = iota
    Bool
    Int
    Int8
    ...
    Ptr
    Slice
    String
    Struct
    UnsafePointer
)
```

其中Bool为1，Int为2，依次递增1。可以用代码验证一下。

```go
fmt.Printf("%d\n", reflect.Bool)	//output: 1
fmt.Println(reflect.Bool)	//output: bool
```

下面为上面会输出bool呢，这是因为type.go中实现了String()方法。

```go
func (k Kind) String() string {
    if int(k) < len(kindNames) {
    	return kindNames[k]
    }
    return "kind" + strconv.Itoa(int(k))
}
...
var kindNames = []string{
    Invalid:       "invalid",
    Bool:          "bool",
    Int:           "int",
    Int8:          "int8",
    Int16:         "int16",
    Int32:         "int32",
    Int64:         "int64",
    ...
    Interface:     "interface",
    Map:           "map",
    Ptr:           "ptr",
    Slice:         "slice",
    String:        "string",
    Struct:        "struct",
    UnsafePointer: "unsafe.Pointer",
}
```



类型反射解析

我们按ValueOf()函数的流程走一遍。

```go
func ValueOf(i interface{}) Value {
    if i == nil {
    	return Value{}
    }
    
    // TODO(rsc): Eliminate this terrible hack.
    // In the call to unpackEface, i.typ doesn't escape,
    // and i.word is an integer.  So it looks like
    // i doesn't escape.  But really it does,
    // because i.word is actually a pointer.
    escapes(i)
    
    return unpackEface(i)
}
```

escapes(i)是一个特殊处理，可以先不管，我们先看一下Value struct结构和unpackEface()函数。

```go
// unpackEface converts the empty interface i to a Value.
type Value struct {
    // typ holds the type of the value represented by a Value.
    typ *rtype
    
    // Pointer-valued data or, if flagIndir is set, pointer to data.
    // Valid when either flagIndir is set or typ.pointers() is true.
    ptr unsafe.Pointer
    
    //too much comment, delete it
    flag
}
...
func unpackEface(i interface{}) Value {
    e := (*emptyInterface)(unsafe.Pointer(&i))
    // NOTE: don't read e.word until we know whether it is really a pointer or not.
    t := e.typ
    if t == nil {
    	return Value{}
    }
    f := flag(t.Kind())
    if ifaceIndir(t) {
    	f |= flagIndir
    }
    return Value{t, unsafe.Pointer(e.word), f}
}
```

Value struct结构是核心，rtype是数据的底层实现，flag是元数据，可用用来表征Value的数据类型。unpackEface是把interface{}转换成Value数据。代码中ifaceIndir()用来确定是不是指针。
我们上面代码中还出现了Kind()，Kind()内不是实现的是kind()，从下面代码可以看出主要就是一个&的运算。这里还有个一个小细节。1<<flagKindWidth - 1在这里的结果为(1<<flagKindWidth) - 1，而C语言中的结果为1<<(flagKindWidth-1)。