# Kubernetes Service

和之前文章里介绍的[Pod](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485464&idx=1&sn=00ca443bbcd4b2996efdede396b6c667&chksm=fa80d98fcdf7509944d63f618264e36cd8082a77e23aa36428a3d57a2f4189bcce4e52986967&token=2051310148&lang=zh_CN&scene=21#wechat_redirect)，[ReplicaSet](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485541&idx=1&sn=59d41bef81615420319b9a721a78ecee&chksm=fa80d9f2cdf750e4250ee59d842501a55375c4c8eacf971eb72adbc3a3cad27259cbd6c27200&token=2051310148&lang=zh_CN&scene=21#wechat_redirect)，[Deployment](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485643&idx=1&sn=6460bf2e170e4b2e8ebb2882bfe7c60f&chksm=fa80d95ccdf7504ad9b5e3ba7ad3dad6a25347a7b0aad4636523cb1ba878cebbc480bf2153a0&token=2051310148&lang=zh_CN&scene=21#wechat_redirect)一样，**Service**也是**Kubernetes**里的一个API对象，而 **Kubernetes** 之所以需要 **Service**，一方面是因为`Pod` 的 IP 不是固定的，另一方面则是因为一组`Pod` 实例需要`Service`提供复杂均衡功能。**所以`Service`是在逻辑抽象层上定义了一组`Pod`，为他们提供一个统一的固定IP和访问这组`Pod`的负载均衡策略**。

下面是`Service`对象的常用属性设置：

- 使用label selector，在集群中查找目标`Pod`;
- ClusterIP设置Service的集群内IP让`kube-proxy`使用;
- 通过prot和targetPort将访问端口与目标端口建议映射（不指定targetPort时默认值和port设置的值一样）;
- Service支持多个端口映射
- Service支持HTTP（默认），TCP和UDP协议;

下面是一个典型的`Service`定义：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

## 都有哪些类型的Service

`Kubernetes`中有四种Service类型：

- **ClusterIP**。这是默认的Service类型，会将Service对象通过一个内部IP暴露给集群内部，这种类型的Service只能够在集群内部使用`<ClusterIP>:<port>`访问。
- **NodePort**。会在每个宿主机节点的一个指定的固定端口上暴露Service，与此同时还会自动创建一个ClusterIP类型的Service，NodePort类型的Service会将集群外部的请求路由给ClusterIP类型的Service。你可以使用`<NodeIP>:<NodePort>`访问NodePort类型的Service，NodePort的端口范围为30000-32767。
- **LoadBalancer**。适用于公有云上的`Kubernetes`服务，使用公有云服务的`CloudProvider`创建**LoadBalancer**类型的**Service**，同时会自动创建**NodePort**和**ClusterIP**类型的**Service**，**LoadBalancer**会把请求路由到**NodePort**和**ClusterIP**类型的**Service**上。
- **ExternalName**。ExternalName 类型的 Service，是在 kube-dns 里添加了一条 CNAME 记录。这个CNAME记录是在Service的spec.externalName里指定的，

以上四种类型除了`ExternalName`，`Kubernetes`的`kube-proxy`组件都会为`Service`提供VIP（虚拟IP），`kube-proxy`支持两种模式：**iptables**和**ipvs**。涉及到不少知识，感兴趣的可以去极客时间上看这篇文章：**Service, DNS与服务发现**[1]

上面的第三和第四种类型的`Service`在本地试验不了，所以后面的例子我们主要通过`NodePort`类型的`Service`学习它的基本用法。

## 怎么发现Service

在`Kubernetes`里的内部组件`kube-dns`会监控Kubernetes API，当有新的`Service`对象被创建出来后，`kube-dns`会为`Service`对象添加DNS A记录（从域名解析 IP 的记录）

对于 `ClusterIP` 模式的 `Service` 来说，它的 A 记录的格式是:

**serviceName.namespace.svc.cluster.local**，当你访问这条 A 记录的时候，它解析到的就是该 Service 的 VIP 地址。

对于指定了 clusterIP=None 的 Headless Service来说，它的A记录的格式跟上面一样，但是访问记录后返回的是Pod的IP地址集合。Pod 也会被分配对应的 DNS A 记录，格式为：**podName.serviceName.namesapce.svc.cluster.local**

我们会在后面的实践练习里通过`nslookup`印证DNS记录是否符合这里说的格式

## 创建和使用Service

跟其他`Kubernetes`里的API对象，`Service`也是通过`YAML`文件定义然后提交给`Kubernetes`后由`ApiManager`创建完成。一个典型的`NodePort`类型的`Service`的定义如下所示：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  type: NodePort
  selector:
    app: go-app
  ports:
    - name: http
      protocol: TCP
      nodePort: 30080
      port: 80
      targetPort: 3000
```

这里定义的Service对象会去管控我们在之前的文章《[K8s上的Go服务怎么扩容、发版更新、回滚、平滑重启？教你用Deployment全搞定！](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485643&idx=1&sn=6460bf2e170e4b2e8ebb2882bfe7c60f&chksm=fa80d95ccdf7504ad9b5e3ba7ad3dad6a25347a7b0aad4636523cb1ba878cebbc480bf2153a0&token=2057376102&lang=zh_CN&scene=21#wechat_redirect)》里用`Deployment`创建的`Go`应用的三个`Pod`副本。

```
➜ kubectl get pods -l app=go-app 
NAME                         READY   STATUS    RESTARTS   AGE
my-go-app-864496b67b-6hm7r   1/1     Running   1          16d
my-go-app-864496b67b-d87kl   1/1     Running   1          16d
my-go-app-864496b67b-qxrsr   1/1     Running   1          16d
➜ 
```

我们用**kubectl apply -f service.yaml**命令把定义好的`Service`提交给`Kubernetes`：

```
➜ kubectl apply -f service.yaml 
service/app-service created
```

被`Service`的`selector`选中的`Pod`，就称为`Service` 的 `Endpoints`，可以使用 **kubectl get ep** 命令看到它们，如下所示：

```
➜  kubectl get ep app-service
NAME          ENDPOINTS                                         AGE
app-service   172.17.0.6:3000,172.17.0.7:3000,172.17.0.8:3000   8m38s
```

需要注意的是，只有处于`Running`状态，且 [readinessProbe](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485500&idx=1&sn=6197d294cc4f409c2a62a7997c431b68&chksm=fa80d9abcdf750bd6735e7b7481c225d4cbfaad2159427b049d8b229877392f4bef11469363e&token=2057376102&lang=zh_CN&scene=21#wechat_redirect) 检查通过的`Pod`，才会出现在`Service`的 `Endpoints` 列表里。当某一个`Pod`出现问题时，`Kubernetes` 会自动把它从 `Service` 里摘除掉。

使用 **kubectl get svc**可以查看到刚才看到的`Service`的信息和状态。

```

➜ kubectl get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
app-service   NodePort    10.108.26.155   <none>        80:30080/TCP   116m
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP        89d
```

## nodePort 、port、targetPort都是啥

上面我们创建了一个`NodePort`类型的`Service`，在下面的端口映射**spec.ports**配置里，每个端口映射里出现了三种**port**：nodePort、port、targetPort。那这三种`port`都代表的什么意思呢？

- **port**：指定在集群内部暴露`Service` 所使用的端口，集群内部使用`<ClusterIP>:<port>`访问`Service`的`EndPoints` （`Service`选中的Pod）。
- **nodePort**：指定向集群外部暴露`Service` 所使用的端口，从集群外部使用`<NodeIp>:<NodePort>`访问`Service`的`EndPoints`。如果你不显式地声明 `nodePort` 字段，会随机分配可用端口来设置代理。这个端口的范围默认是 30000-32767。
- **targetPort**：`targetPort`是后面的`Pod`监听的端口，容器里的应用也应该监听这个端口，`Service`会把请求发送到这个端口。

所以结合刚才我们创建的app-service这个`Service`的信息，在集群内部使用**10.108.26.155:80** 访问`Pod`里的应用。因为我们试验使用的`minikube`是个单节点的集群，`NodeIP`可以通过 **minikube ip**命令获得。

```
➜ minikube ip

192.168.64.4
```

所以从集群外部，通过**192.168.64.4:30080**访问`Pod`里的应用。

```
➜ curl 192.168.64.4:30080
Hello World
Hostname: my-go-app-75d6d768ff-mlqnh%                                                                                                                    ➜ curl 192.168.64.4:30080
Hello World
Hostname: my-go-app-75d6d768ff-4x8p8%                                                                                                                    ➜  curl 192.168.64.4:30080
Hello World
Hostname: my-go-app-75d6d768ff-vt7dx%                                                                                                                    
```

通过多次访问，我们可以看到请求会通过`Service`发给不同的应用`Pod`。`Pod`里的应用就是在原来的文章里一直使用的例子的基础上加了一行获取系统`Hostname`的代码

最后我们进到`Pod`里看一下`Service`创建后`kube-dns`组件在集群里为`app-service`这个`Service`对象创建的DNS A记录，因为`Service`定义里指定的名字是`app-service`，命名空间的话因为没有指定就是默认的`default`命名空间，所以我们使用**nslookup app-service.default.svc.cluster.local** 查看一下这条`DNS`记录，进入到其中一个`Pod`里，执行上述查询的结果如下：

```
nslookup app-service.default.svc.cluster.local
  
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   app-service.default.svc.cluster.local
Address: 10.108.26.155
```

对于`Service`的`EndPoints` 也是有`DNS`记录的，因为不是`Headless Service`，所以需要用nslookup *.app-service.default.svc.cluster.local查询`DNS`记录。

```
nslookup *.app-service.default.svc.cluster.local
  
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   *.app-service.default.svc.cluster.local
Address: 172.17.0.8
Name:   *.app-service.default.svc.cluster.local
Address: 172.17.0.6
Name:   *.app-service.default.svc.cluster.local
Address: 172.17.0.7
```

上面查询出来三条`DNS`记录，正好跟`Service`管控的`Pod`数量能够对上。