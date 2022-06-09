# Go语言常见Bug

## 1. nil map

在程序中声明（定义）了一个 map，然后直接写入数据。如下代码：

```go
func main() {
 var m map[string]string
 m["煎鱼"] = "进脑子了"
}
```

输出结果：

```
panic: assignment to entry in nil map
```

会直接抛出一个 panic。



## 2. 空指针的引用

#### 问题

我们在 Go 经常会利用结构体去声明一系列的方法，他看起来向面向对象中的 ”类“，在业务代码中非常常见。

如下代码：

```go
type Point struct {
    X, Y float64
}

func (p *Point) Abs() float64 {
    return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

func main() {
    var p *Point
    fmt.Println(p.Abs())
}
```

结果：

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10a3143]

goroutine 1 [running]:
main.(*Point).Abs(...)
        /Users/eddycjy/awesomeProject/main.go:13
main.main()
        /Users/eddycj/awesomeProject/main.go:18 +0x23
```

直接就恐慌了，由于空指针的引用。

#### 解决方法

如果变量 p 是一个指针，则必须要进行初始化才可以进行调用。如下代码：

```go
func main() {
    var p *Point = new(Point)
    fmt.Println(p.Abs())
}
```

又或是用值对象的方法来解决：

```go
func main() {
    var p Point // has zero value Point{X:0, Y:0}
    fmt.Println(p.Abs())
}
```

>**开发中一定要注意：结构体嵌套指针问题！！**
>
>**非常容易panic！！！**



## 3. 使用对循环迭代器变量的引用

#### 问题

在 Go 中，循环迭代器变量是一个单一的变量，在每个循环迭代中取不同的值。这如果使用不当，可能会导致非预期的行为。

如下代码：

```go
func main() {
 var out []*int
 for i := 0; i < 3; i++ {
  out = append(out, &i)
 }
 fmt.Println("Values:", *out[0], *out[1], *out[2])
 fmt.Println("Addresses:", out[0], out[1], out[2])
}
```

输出结果：

```
Values: 3 3 3
Addresses: 0x40e020 0x40e020 0x40e020
```

值都是 3，地址都是同一个指向。

#### 解决方法

其中一种解决方法是将循环变量复制到一个新变量中：

```go
 for i := 0; i < 3; i++ {
  _i := i // Copy i into a new variable.
  out = append(out, &_i)
 }
```

输出结果：

```go
Values: 0 1 2
Addresses: 0x40e020 0x40e024 0x40e028
```

原因是：在每次迭代中，我们将 i 的地址追加到 out 切片中，但由于它是同一个变量，我们实际上追加的是相同的地址，该地址最终包含分配给 i 的最后一个值。

所以只需要拷贝一份，让两者脱离关联就可以了。

## 4. 在循环迭代器变量上使用 goroutine

#### 问题

在 Go 中进行循环时，我们经常会使用 goroutine 来并发处理数据。最经典的就是会结合闭包来编写业务逻辑。

如下代码：

```go
values := []int{1, 2, 3, 4, 5}
for _, val := range values {
 go func() {
  fmt.Println(val)
 }()
}

time.Sleep(time.Second)
```

但在实际的运行中，上述 for 循环可能无法达到您的预期，你想的可能是顺序输出切片中的值。

输出的结果是：

```
5
5
4
5
5
```

你可能会看到每次迭代打印的最后一个元素，甚至你会发现，每次输出的结果还不一样...

如果去掉休眠代码，会发现 goroutine 可能根本不会开始执行，程序就结束了。

#### 解决方法

这其实就是闭包使用上的一个常见问题，编写该闭包循环的正确方法是：

```go
values := []int{1, 2, 3, 4, 5}
 for _, val := range values {
  go func(val int) {
   fmt.Println(val)
  }(val)
 }
```

通过将 val 作为参数添加到闭包中，在每次循环时，变量 val 都会被存储在 goroutine 的堆栈中，以确保最终 goroutine 执行时值是对的。

当然，这里还有一个隐性问题。大家总会以为是按顺序输出 1, 2, 3, 4, 5。其实不然，因为 goroutine 的执行是具有随机性的，没法确保顺序。

注：经常会变形出现在许多 Go 的面试题当中，一旦复杂起来就容易让人迷惑。

## 5. 数组不会被改变

#### 问题

切片和数字是我们在 Go 程序中应用最广泛的数据类型，但他常常会有一些奇奇怪怪的问题。

如下代码：

```go
func Foo(a [2]int) {
 a[0] = 8
}

func main() {
 a := [2]int{1, 2}
 Foo(a)       
 fmt.Println(a) 
}
```

输出结是什么。是 [8 2]，对吗？

输出结果：

```
[1 2]
```

这是为什么，函数里修改了个寂寞？

#### 解决方法

实际上在 Go 中，所有的函数传递都是值传递。也就是将数组传递给函数时，会复制该数组。如果真的是需要传进函数内修改，可以改用切片。

如下代码：

```go
func Foo(a []int) {
    if len(a) > 0 {
        a[0] = 8
    }
}

func main() {
    a := []int{1, 2}
    Foo(a)         
    fmt.Println(a)
}
```

输出结果：

```
[8 2]
```

原因是：切片不会存储任何的数据，他的底层 data 会指向一个底层数组。因此在修改切片的元素时，会修改其底层数组的相应元素，共享同一个底层数组的其他切片会一并修改。

你以为这就万事大吉，解决了？并不。当切片扩容时，Go 底层会重新申请新的更大空间，存在与原有切片分离的场景。

因此还是要及时将变更的值返回出来，在主流程上统一处理元数据会更好。