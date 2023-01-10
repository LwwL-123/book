# K8S上的Web服务域名解析

> 本文来自：https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247489112&idx=1&sn=481f3298831004384748fc5beceab094&chksm=fa80c7cfcdf74ed963afaf2417f80bc64f4a96742c2c19d187190f93c983548c2ad785aa2245&scene=178&cur_album_id=1394839706508148737#rd

## 为什么NodePort不适合做域名解析

`NodePort` 类型的`Service` 是向集群外暴露服务的最原始方式，也是最好让人理解的。`NodePort`，顾名思义，会在所有节点（宿主机或者是VM）上打开一个特定的端口，发送到这个端口的任何流量都会转发给`Service`。

`NodePort` Service 的原理可以用下面这个图表示：

![图片](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221109115341.jpeg)

上图我们顺着流量的流向箭头从下往上看，流量通过`NodeIP+NodePort`的方式进入集群，上图三个节点的30001上的流量都会转发给Service，再由Service给到后端端点`Pod`。

NodePort Service的优点是简单，好理解，通过IP+端口的方式就能访问，但是它的缺点也很明显，比如：

- 每向外暴露一个服务都要占用所有Node的一个端口，如果多了难以管理。
- NodePort的端口区间固定，只能使用30000–32767间的端口。
- 如果Node的IP发生改变，负载均衡代理需要跟着改后端端点IP才行。