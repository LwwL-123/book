# Lee's Book

## [前言](README.md)

## 1. Go底层原理

- [1.1 Go常见关键字及其原理]()
    - [1.1.1 chan](Go底层原理/Go常见关键字及其原理/chan.md)
    - [1.1.2 slice](Go底层原理/Go常见关键字及其原理/slice.md)
    - [1.1.3 map](Go底层原理/Go常见关键字及其原理/map.md)
    - [1.1.4 defer](Go底层原理/Go常见关键字及其原理/defer.md)
    - [1.1.5 select](Go底层原理/Go常见关键字及其原理/select.md)
    - [1.1.6 互斥锁Mutex](Go底层原理/Go常见关键字及其原理/Go的互斥锁Mutex.md)
    - [1.1.7 []Byte与String](Go底层原理/Go常见关键字及其原理/[]byte与string.md)
    - [1.1.8 Rune关键字](Go底层原理/Go常见关键字及其原理/rune关键字.md)
    - [1.1.9 Interface](Go底层原理/Go常见关键字及其原理/深入理解 Go Interface.md)
    - [1.1.10 iota](Go底层原理/Go常见关键字及其原理/iota.md)
    - [1.1.11 WaitGroup及内存对齐分析](Go底层原理/Go常见关键字及其原理/WaitGroup及内存对齐分析.md)
    - [1.1.12 Go闭包](Go底层原理/Go常见关键字及其原理/Go 闭包.md)
    - [1.1.13 sync.map](Go底层原理/Go常见关键字及其原理/sync_map.md)
- [1.2 Go内存管理及协程调度]()
    - [1.2.1 Go内存分配](Go底层原理/Go内存分配.md)
    - [1.2.2 内存对齐](Go底层原理/内存对齐.md)
    - [1.2.3 逃逸分析](Go底层原理/逃逸分析.md)
    - [1.2.4 Go垃圾回收原理](Go底层原理/Go垃圾回收原理.md)
    - [1.2.5 Go协程的栈内存管理](Go底层原理/Go协程的栈内存管理.md)
    - [1.2.6 GMP并发调度器深度解析](Go底层原理/GMP并发调度器深度解析.md)
- [1.3 Go context]()
    - [1.3.1 Go context](Go底层原理/Go常见关键字及其原理/context.md)
    - [1.3.2 context取消goroutine](Go面试/context取消goroutine执行的方法.md)

## 2. Go面试

- [2.1 Go面试题总结](面试/Go面试题总结.md)
- [2.2 Go结构体比较](面试/Go结构体比较.md)
- [2.3 Go语言CSP：通信顺序进程简述](面试/Go语言CSP：通信顺序进程简述.md)
- [2.4 Goroutine数量](面试/Goroutine数量.md)
- [2.5 new和make](面试/new_make.md)
- [2.6 空结构体](面试/Go 语言中 Set 的最佳实现方案.md)
- [**面试题大汇总**](面试/面试题总结.md)



## 3. 数据库

- [3.1 锁](数据库/锁.md)
- [3.2 对象存储](数据库/对象存储.md)
- [3.3 Redis](数据库/Redis.md)
- [3.4 数据库设计规范](数据库/数据库设计规范.md)
- [3.5 Databus-低延迟的分布式数据库同步系统](数据库/Databus-低延迟的分布式数据库同步系统.md)
- [3.6 索引下推](数据库/索引下推.md)
- [3.7 API分页方案](数据库/分页方案.md)
- [3.8 ON DUPLICATE KEY UPDATE](数据库/为什么不建议使用ON DUPLICATE KEY UPDATE？.md)
- [3.9 MySQL 是怎么加锁的](数据库/MySQL 是怎么加锁的？.md)

## 4. 计算机组成原理

- [4.1 CPU三级缓存](计算机组成原理/CPU三级缓存.md)

## 5. 计算机网络

- [5.1 TCP](计算机网络/TCP.md)
- [5.2 TCP三次握手四次挥手](计算机网络/TCP三次握手四次挥手.md)
- [5.3 网络面试题](计算机网络/网络面试题.md)
- [5.4 NAT](计算机网络/NAT.md)
- [5.5 JWT](计算机网络/JWT.md)

## 6. 操作系统

- [6.1 Linux常见命令](Linux/Linux常见命令.md)
- [6.2 虚拟内存](Linux/Linux虚拟内存.md)
- [6.3 进程与线程](Linux/Linux进程与线程.md)
- [6.6 栈帧](Linux/栈帧.md)
- [6.7 IO多路复用](Linux/IO多路复用.md)

## 7. 容器

- [7.1 Docker文件系统](Linux/docker文件系统.md)

- [7.2 Cgroups与Docker资源限制](Linux/Cgroups与Docker资源限制.md)

- [7.3 容器网络原理](Linux/容器网络原理.md)

- [7.4 PromQL查询解析](容器/PromQL查询解析.md)

## 8. 分布式与微服务

- [8.1 CAP理论](分布式/CAP理论.md)
- [8.2 数据一致性问题](分布式/数据一致性问题.md)
- [8.3 共识算法]()
  - [Raft](分布式/共识算法/Raft.md)
  - [链式共识算法](分布式/共识算法/链式共识算法.md)
  - [PoW](分布式/共识算法/POW工作量证明共识机制.md)
- [8.4 Etcd]()
  - [Etcd](etcd/etcd.md)
- [8.5 SOA和微服务区别](微服务/SOA和微服务区别.md)
- [8.6 分布式事务](微服务/分布式事务.md)
- [8.7 API网关与BFF](微服务/API网关与BFF.md)

- [8.8 如何设计一个高可用系统](微服务/如何设计一个高可用系统.md)
- [8.9 负载均衡](微服务/负载均衡.md)

## 9. Ipfs

- [Libp2p](ipfs/libp2p.md)
- [深入理解 IPFS](ipfs/深入理解 IPFS.md)

## 10. Substrate

- [Substrate的Staking模块分析](substrate/Substrate的Staking模块分析.md)
- [权重Weight和基准Benchmarking](substrate/权重Weight和基准Benchmarking.md)

## 11. 设计模式

- [设计模式](设计模式/设计模式.md)



## 12. Git

- [Git Flow规范](Git/Git Flow规范.md)
- [git rebase与git merge](Git/git rebase与git merge.md)
- [git四个区域](Git/Git四个区域.md)



## 13. 实战

- [Golang问题排查](实战/golang 内存问题排查.md)

## 14. 区块链

- [密码学](区块链/密码学.md)
- [椭圆曲线签名算法](区块链/椭圆曲线签名算法.md)

## 15. Leetcode刷题

- [动态规划](算法/动态规划.md)
- [贪心算法](算法/贪心算法.md)
- [树](算法/树.md)
- [二叉搜索树](算法/二叉搜索树.md)
- [二分搜索](算法/二分搜索.md)
- [双指针](算法/双指针.md)
- [页面置换算法](算法/页面置换算法.md)
- [岛屿问题](算法/岛屿问题.md)
- [反转链表](算法/反转链表.md)
- [链表](算法/链表.md)
- [回溯](算法/回溯.md)
- [单调栈](算法/单调栈.md)
- [排序算法](算法/排序算法.md)
- [字符串](算法/字符串.md)
- [哈希表](算法/哈希表.md)
- [并查集](算法/并查集.md)
- [前缀和](算法/前缀和.md)
- [其他](算法/other.md)
