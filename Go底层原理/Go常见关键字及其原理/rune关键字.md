# Go关键字rune



查询,官方的解释如下：

```golang
//int32的别名，几乎在所有方面等同于int32
//它用来区分字符值和整数值

type rune = int32
```



例如：

```golang
func main() {
    var str = "hello 你好"
    fmt.Println("len(str):", len(str)) 
  	// 12
}
```

输出的结果为12，是因为在golang中string底层是通过byte数组实现的。中文字符在在utf-8编码下占3个字节，而golang默认编码正好是utf-8。

所以打印的不是字符串的长度，而是字符串底层占的字节长度



想获得字符串的长度，可以采用以下两种方法：

```go
  //以下两种都可以得到str的字符串长度

  //golang中的unicode/utf8包提供了用utf-8获取长度的方法
  fmt.Println("RuneCountInString:", utf8.RuneCountInString(str))

  //通过rune类型处理unicode字符
  fmt.Println("rune:", len([]rune(str)))
```

