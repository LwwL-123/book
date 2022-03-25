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



### 查找过程

划重点，Map 类型本质上是有两个 “map”。一个叫 read、一个叫 dirty，长的也差不多：

![image-20220307101658172](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173248.png)

当我们从 sync.Map 类型中读取数据时，其会先查看 read 中是否包含所需的元素：

- 若有，则通过 atomic 原子操作读取数据并返回。
- 若无，则会判断 read.readOnly 中的 amended 属性，他会告诉程序 dirty 是否包含 read.readOnly.m 中没有的数据;因此若存在，也就是 amended 为 true，将会进一步到 dirty 中查找数据。

sync.Map 的读操作性能如此之高的原因，就在于存在 read 这一巧妙的设计，其作为一个缓存层，提供了快路径(fast path)的查找。

同时其结合 amended 属性，配套解决了每次读取都涉及锁的问题，实现了读这一个使用场景的高性能。



### 写入过程

我们直接关注 sync.Map 类型的 Store 方法，该方法的作用是新增或更新一个元素。

源码如下：

```go
func (m *Map) Store(key, value interface{}) { 
 read, _ := m.read.Load().(readOnly) 
 if e, ok := read.m[key]; ok && e.tryStore(&value) { 
  return 
 } 
  ... 
} 
```

调用 Load 方法检查 m.read 中是否存在这个元素。若存在，且没有被标记为删除状态，则尝试存储。

若该元素不存在或已经被标记为删除状态，则继续走到下面流程：

```go
func (m *Map) Store(key, value interface{}) { 
 ... 
 m.mu.Lock() 
 read, _ = m.read.Load().(readOnly) 
 if e, ok := read.m[key]; ok { 
  if e.unexpungeLocked() { 
   m.dirty[key] = e 
  } 
  e.storeLocked(&value) 
 } else if e, ok := m.dirty[key]; ok { 
  e.storeLocked(&value) 
 } else { 
  if !read.amended { 
   m.dirtyLocked() 
   m.read.Store(readOnly{m: read.m, amended: true}) 
  } 
  m.dirty[key] = newEntry(value) 
 } 
 m.mu.Unlock() 
} 
```

由于已经走到了 dirty 的流程，因此开头就直接调用了 Lock 方法上互斥锁，保证数据安全，也是凸显性能变差的第一幕。

其分为以下三个处理分支：

- 若发现 read 中存在该元素，但已经被标记为已删除(expunged)，则说明 dirty 不等于 nil(dirty 中肯定不存在该元素)。其将会执行如下操作。
  - 将元素状态从已删除(expunged)更改为 nil。
  - 将元素插入 dirty 中。
- 若发现 read 中不存在该元素，但 dirty 中存在该元素，则直接写入更新 entry 的指向。
- 若发现 read 和 dirty 都不存在该元素，则从 read 中复制未被标记删除的数据，并向 dirty 中插入该元素，赋予元素值 entry 的指向。



我们理一理，写入过程的整体流程就是：

- 查 read，read 上没有，或者已标记删除状态。
- 上互斥锁(Mutex)。
- 操作 dirty，根据各种数据情况和状态进行处理。

回到最初的话题，为什么他写入性能差那么多。究其原因：

- 写入一定要会经过 read，无论如何都比别人多一层，后续还要查数据情况和状态，性能开销相较更大。
- (第三个处理分支)当初始化或者 dirty 被提升后，会从 read 中复制全量的数据，若 read 中数据量大，则会影响性能。

可得知 sync.Map 类型不适合写多的场景，读多写少是比较好的。

若有大数据量的场景，则需要考虑 read 复制数据时的偶然性能抖动是否能够接受。



## Delete

```golang
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	// 如果 read 中没有这个 key，且 dirty map 不为空
	if !ok && read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			delete(m.dirty, key) // 直接从 dirty 中删除这个 key
		}
		m.mu.Unlock()
	}
	if ok {
		e.delete() // 如果在 read 中找到了这个 key，将 p 置为 nil
	}
}
```

可以看到，基本套路还是和 Load，Store 类似，都是先从 read 里查是否有这个 key，如果有则执行 `entry.delete` 方法，将 p 置为 nil，这样 read 和 dirty 都能看到这个变化。

如果没在 read 中找到这个 key，并且 dirty 不为空，那么就要操作 dirty 了，操作之前，还是要先上锁。然后进行 double check，如果仍然没有在 read 里找到此 key，则从 dirty 中删掉这个 key。但不是真正地从 dirty 中删除，而是更新 entry 的状态。

来看下 `entry.delete` 方法：

```golang
func (e *entry) delete() (hadValue bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == nil || p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return true
		}
	}
}
```

它真正做的事情是将正常状态（指向一个 interface{}）的 p 设置成 nil。没有设置成 expunged 的原因是，当 p 为 expunged 时，表示它已经不在 dirty 中了。这是 p 的状态机决定的，在 `tryExpungeLocked` 函数中，会将 nil 原子地设置成 expunged。

`tryExpungeLocked` 是在新创建 dirty 时调用的，会将已被删除的 entry.p 从 nil 改成 expunged，这个 entry 就不会写入 dirty 了。

```golang
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		// 如果原来是 nil，说明原 key 已被删除，则将其转为 expunged。
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}
```

注意到如果 key 同时存在于 read 和 dirty 中时，删除只是做了一个标记，将 p 置为 nil；而如果仅在 dirty 中含有这个 key 时，会直接删除这个 key。原因在于，若两者都存在这个 key，仅做标记删除，可以在下次查找这个 key 时，命中 read，提升效率。若只有在 dirty 中存在时，read 起不到“缓存”的作用，直接删除。