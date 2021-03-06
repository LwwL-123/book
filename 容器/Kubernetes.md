# Kubernetes

在大规模集群中各种任务之间运行，实际上存在各种各样的关系。处理这些关系才是作业编排和调度最苦难的地方。

过去的很多集群管理项目所擅长的，是把一个容器按照某种规则放到某个最佳节点上运行。这种功能成为”调度“，而kubernetes所擅长的，是按照用户的意愿和整个系统的规则，完全处理好容器之间的各种关系，这种功能，就叫做“编排”



## pod里容器的启动顺序

在kubernetes项目里，pod实现需要使用一个中间容器，这个容器叫infra，在这个pod中，infra容器永远都是第一个被创建的，用户定义的其他容器通过join network namespace的方式与infra关联在一起。这样多个容器就是对等关系，而不是拓扑了。

infra容器是一个极少资源的容器，永远处于暂停状态。

这也意味着，对于Pod里的容器A和B来说：

- 可以直接使用localhost通信
- 可以到看网络设备和infra完全一样
- 一个pod只有一个ip地址，也就是这个pod的ip地址
- pod生命周期只和infra有关，和A、B容器无关
- 对于一个pod里的所有用户容器来说，他们的进出流量可以认为是通过infra容器完成的
