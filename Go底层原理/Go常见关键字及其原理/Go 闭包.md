# Go 闭包

首先一道题目，以下代码输出什么？

```GO
package main

import "fmt"

func app() func(string) string {
 t := "Hi"
 c := func(b string) string {
  t = t + " " + b
  return t
 }
 return c
}

func main() {
 a := app()
 b := app()
 a("go")
 fmt.Println(b("All"))
 fmt.Println(a("All"))
}
///
/// Hi All
/// Hi go All
```



## 01 什么是闭包

维基百科对闭包的定义：

> 在计算机科学中，闭包（英语：Closure），又称词法闭包（Lexical Closure）或函数闭包（function closures），是在支持头等函数的编程语言中实现词法绑定的一种技术。闭包在实现上是一个结构体，它存储了一个函数（通常是其入口地址）和一个关联的环境（相当于一个符号查找表）。环境里是若干对符号和值的对应关系，它既要包括约束变量（该函数内部绑定的符号），也要包括自由变量（在函数外部定义但在函数内被引用），有些函数也可能没有自由变量。闭包跟函数最大的不同在于，当捕捉闭包的时候，它的自由变量会在捕捉时被确定，这样即便脱离了捕捉时的上下文，它也能照常运行。捕捉时对于值的处理可以是值拷贝，也可以是名称引用，这通常由语言设计者决定，也可能由用户自行指定（如 C++）。

关于（函数）闭包，有几个关键点：

- 函数是一等公民；
- 闭包所处环境，可以引用环境里的值



> 在支持函数是一等公民的语言中，一个函数的返回值是另一个函数，被返回的函数可以访问父函数内的变量，当这个被返回的函数在外部执行时，就产生了闭包。

所以，上面题目中，函数 app 的返回值是另一个函数，因此产生了闭包。



## 02 Go 中的闭包

日常开发中，闭包是很常见的。举几个例子。

### 标准库

在 net/http 包中的函数 ProxyURL，实现如下：

```go
// ProxyURL returns a proxy function (for use in a Transport)
// that always returns the same URL.
func ProxyURL(fixedURL *url.URL) func(*Request) (*url.URL, error) {
 return func(*Request) (*url.URL, error) {
  return fixedURL, nil
 }
}
```

它的返回值是另一个函数，签名是：

```go
func(*Request) (*url.URL, error)
```

在返回的函数中，引用了父函数（ProxyURL）的参数 fixedURL，因此这是闭包。



### Web 中间件

在 Web 开发中，中间件一般都会使用闭包。比如 Echo 框架中的一个中间件：

```go
// BasicAuthWithConfig returns an BasicAuth middleware with config.
// See `BasicAuth()`.
func BasicAuthWithConfig(config BasicAuthConfig) echo.MiddlewareFunc {
 // Defaults
 if config.Validator == nil {
  panic("echo: basic-auth middleware requires a validator function")
 }
  ...
 return func(next echo.HandlerFunc) echo.HandlerFunc {
  return func(c echo.Context) error {
   /// 省略很多代码
      ...
  }
 }
}
```

首先，echo.MiddlewareFunc 是一个函数：

```go
type MiddlewareFunc func(HandlerFunc) HandlerFunc
```

而 echo.HandlerFunc 也是一个函数：

```go
type HandlerFunc func(Context) error
```

所以，上面的函数嵌套了几层，是典型的闭包。



### 这是闭包吗？

在 Go 中不支持函数嵌套定义，函数内嵌套函数，必须通过匿名函数的形式。匿名函数在 Go 中是很常见的，比如开启一个 goroutine，通常通过匿名函数。

现在有一个问题，以下代码是闭包吗？

现在有一个问题，以下代码是闭包吗？

```go
package main

import (  
    "fmt"
)

func main() {  
    a := 5
    func() {
        fmt.Println("a =", a)
    }()
}
```

如果按照上面网上一般的回答，这不是闭包，因为并没有返回函数。但按照维基百科的定义，这个属于闭包。有没有其他证据呢？



在 Go 语言规范中，关于函数字面值（匿名函数）有这么一句话：

> Function literals are *closures*: they may refer to variables defined in a surrounding function. Those variables are then shared between the surrounding function and the function literal, and they survive as long as they are accessible.

也就是说，函数字面值（匿名函数）是闭包，它们可以引用外层函数定义的变量。

此外，在官方 FAQ 中有这样的说明：

What happens with closures running as goroutines?
