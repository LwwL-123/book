# defer

```go
package main

func main() {
   println(f1())
   println(f2())
   println(f3())
}

func f1() (r int) {
   defer func() {
      r++
   }()
   return 0
}

func f2() (r int) {
   t := 5
   defer func() {
      t = t + 5
   }()
   return t
}

func f3() (r int) {
   defer func(r int) {
      r = r + 5
   }(r)
   return 1
}
```

输出:

```
1
5
1
```



## 为什么

1. 返回r，将0赋值给r，随后在defer中，对r进行+1，所以返回1
2. 返回t=5，将5赋值给r，随后对t进行+5，对r没有影响，所以返回5
3. 将1赋值给r，但defer中是函数变量r，与返回值没有关系，所以返回1



## 总结

1. 多个defer的执行顺序是`先进后出，后进先出`，与栈相同

2. **有返回值的且带有 defer 函数的方法中， return 语句执行顺序：**

   ```
   1. 返回值赋值
   2. 调用 defer 函数 (在这里是可以修改返回值的)
   3. return 返回值
   ```

3. 匿名返回值是在 return 执行时被声明，有名返回值则是在函数声明的同时被声明，因此在 defer 语句中只能访问有名返回值，而不能直接访问匿名返回值；





# defer底层原理

源码包`src/src/runtime/runtime2.go:_defer`定义了defer的数据结构：

```go
type _defer struct {
    sp      uintptr   //函数栈指针
    pc      uintptr   //程序计数器
    fn      *funcval  //函数地址
    link    *_defer   //指向自身结构的指针，用于链接多个defer
}
```

我们知道defer后面一定要接一个函数的，所以defer的数据结构跟一般函数类似，也有栈地址、程序计数器、函数地址等等。

与函数不同的一点是它含有一个指针，可用于指向另一个defer，每个goroutine数据结构中实际上也有一个defer指针，该指针指向一个defer的单链表，每次声明一个defer时就将defer插入到单链表表头，每次执行defer时就从单链表表头取出一个defer执行。



新声明的defer总是添加到链表头部。

函数返回前执行defer则是从链表首部依次取出执行，不再赘述。

一个goroutine可能连续调用多个函数，defer添加过程跟上述流程一致，进入函数时添加defer，离开函数时取出defer，所以即便调用多个函数，也总是能保证defer是按FIFO方式执行的。



## defer的创建和执行

源码包`src/runtime/panic.go`定义了两个方法分别用于创建defer和执行defer。

- deferproc()： 在声明defer处调用，其将defer函数存入goroutine的链表中；
- deferreturn()：在return指令，准确的讲是在ret指令前调用，其将defer从goroutine链表中取出并执行。

可以简单这么理解，在编译在阶段，声明defer处插入了函数deferproc()，在函数return前插入了函数deferreturn()。

