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

