# sync.map

Go 语言原生 map 并不是线程安全的，对它进行并发读写操作的时候，需要加锁。而 `sync.map` 则是一种并发安全的 map

- 在读和删场景上的性能是最佳的，领先原生一倍有多。
- 在写入场景上的性能非常差，落后原生 map+锁整整有一倍之多。



## 数据结构

先来看下 map 的数据结构。去掉大段的注释后：

```golang
type Map struct { 
 mu Mutex 
 read atomic.Value // readOnly 
 dirty map[interface{}]*entry 
 misses int 
} 
 
// Map.read 属性实际存储的是 readOnly。 
type readOnly struct { 
 m       map[interface{}]*entry 
 amended bool 
} 
```

- mu：互斥锁，用于保护 read 和 dirty。
- read：只读数据，支持并发读取(atomic.Value 类型)。如果涉及到更新操作，则只需要加锁来保证数据安全。
- read 实际存储的是 readOnly 结构体，内部也是一个原生 map，amended 属性用于标记 read 和 dirty 的数据是否一致。
- dirty：读写数据，是一个原生 map，也就是非线程安全。操作 dirty 需要加锁来保证数据安全。
- misses：统计有多少次读取 read 没有命中。每次 read 中读取失败后，misses 的计数值都会加 1。



在 read 和 dirty 中，都有涉及到的结构体：

```go
type entry struct { 
 p unsafe.Pointer // *interface{} 
} 
```

其包含一个指针 p, 用于指向用户存储的元素(key)所指向的 value 值。