# Kubernetes Ingress

> 本文来自：https://visualgo.net/zh/recursion

**Ingress**也是Kubernetes项目里的一种 **API** 对象，它公开了从集群外部到集群内`Service`的 **HTTP** 和 **HTTPS** 路由，这些路由由 Ingress 资源上定义的规则控制。

```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

如果用一句话概况`Ingress`的话就是：`Ingress`是`Service`们的反向代理。通过看`Ingress`对象的定义你会感觉自己在看`Nginx`的配置文件一样。

`Ingress`资源对象的`YAML`定义。与大多数Kubernetes资源一样，也需要`apiVersion`，`Kind`，`Metadata`和`Spec` 这些组成部分。

一个典型的`Ingress`对象的定义如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: app-service
              servicePort: 80
          - path: /v2
            backend:
              serviceName: app-service-v2
              servicePort: 80
```

在上面这个`Ingress`的`YAML`定义中，最值得我们关注的，是**spec.rules** 字段。在 Kubernetes 里，这个字段叫作：**IngressRule**。

`IngressRule` 里面 **host** 字段定义的值，就是这个`Ingress`的入口。当访问 `app.example.com` 的时候，实际上访问到的是这个 `Ingress` 对象。这样就能使用 IngressRule 来对请求进行下一步转发。

`IngressRule` 的定义，主要依赖于`path`字段。可以简单地理解为，这里的每一个`path`都对应一个后端 `Service`。上面的例子里，定义了两个`path`，它们分别对应：**app-service**和**app-service-v2** 两个后端`Service`。通过`Service`，流量最终会到达`Service`后面的`Pod`。

**servicePort**字段指定的端口值就是在`Service`的定义里`port`字段的值，上一篇讲Service的文章里已经科普了一下`Service`定义里的`nodePort`，`port`和`targetPort`都是什么意思，感觉到懵圈（我自己都有点懵）的读者大大可以翻阅一下文章《[学练结合，快速掌握Kubernetes Service](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486082&idx=1&sn=42a9bc8fcfc9da09445e9e2f4cf2fb96&chksm=fa80db15cdf752039494992f71a3bc488cf386841bd1aaaa44115f5e7f155ba55ce468ec89ee&token=1964476830&lang=zh_CN&scene=21#wechat_redirect)》里的这部分内容。

所以 `Ingress` 对象，其实就是 `Kubernetes` 项目对**"反向代理"**的一种抽象。一个 `Ingress`对象的主要内容，实际上就是一个"反向代理"服务的配置文件的描述。而这个代理服务对应的转发规则，就是 `IngressRule`。这就是为什么在每条 `IngressRule` 里，需要有一个 **host** 字段来作为这条 `IngressRule` 的入口，然后还需要有一系列 `path` 字段来声明具体的转发策略。

有了`Ingress`后我们还需要`Ingress Controller`，它会根据你定义的`Ingress`对象，提供对应的代理能力。目前，业界常用的各种反向代理项目，比如 Nginx、Envoy 等，都已经为 Kubernetes 专门维护了对应的 Ingress Controller。

下面我就用最常用的Nginx Ingress Controller给这个系列教程一直以来用的Demo实践应用一下Ingress

## 安装Ingress Controller

因为`Minikube`里边内置了Nginx Ingress Controller这个插件， 默认没有启用，所以如果是在`Minikube`这个单节点集群里实践的话只需要执行下面的命令：

```
minikube addons enable ingress
```

检查验证 Nginx Ingress 控制器处于运行状态：

```
kubectl get pods -n kube-system --filed-selector=Running
```

有下图红框里的`Pod`就证明已经安装成功了：

![图片](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221108145721.png)

> 因为这个Pod用的官方镜像是在红帽软件的镜像库里，所以安装时间可能会有点长，也可能会失败，如果网络条件允许的话可以在准备阶段先执行 docker pull http://quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.32.0 把镜像下载到本地。

此外还有不少安装`Nginx Ingress Controller`的方式，比如用Kubernetes的包管理工具**Helm**安装，这些安装方式可以参考**官方的部署指南**[1]。

## 创建Ingress

因为我们之前给应用`Pod`创建的`Service`名字叫**app-service**，`port`字段指定的是端口80

```
➜  ~ kubectl get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
app-service   NodePort    10.108.26.155   <none>        80:30080/TCP   6d16h
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP        96d

➜  ~ kubectl describe svc app-service
Name:                     app-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=go-app
Type:                     NodePort
IP:                       10.108.26.155
Port:                     http  80/TCP
TargetPort:               3000/TCP
NodePort:                 http  30080/TCP
Endpoints:                10.1.0.24:3000,10.1.0.25:3000
```

这就确定了我们要创建建的`Ingress`对象，第一个`path` 里要设置的**backend.serviceName**和**backend.servicePort**字段的值，Ingress对象的`YAML`定义如下：

```
# app-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: app-service
              servicePort: 80
```

> 说明：因为目前为止我们只创建了一个Service，所以在Ingress里也只能设置一个path。

这个Ingress定义里设置的`IngressRules`是把所有对app.example.com入口的请求都路由到`app-service`这个`Service`的80端口。

定义好`Ingress`对象后，也是执行**kubectl apply -f**创建对象，可以看到所有的对象的创建和更新都可以用**apply**操作搞定，这就是`Kubernetes`项目声明式定义的好处。

```
kubectl apply -f app-ingress.yaml --record

# 执行后的输出
ingress.networking.k8s.io/app-ingress created
```

## 验证Ingress

查询Ingress是否创建成功，使用通用的**kubectl get**命令：

```
➜  ~ kubectl get ingress
NAME          CLASS    HOSTS             ADDRESS        PORTS   AGE
app-ingress   <none>   app.example.com   192.168.64.4   80      20s
```

> 有可能需要在提交创建操作几分钟后才能在集群里查询到Ingress

在集群里查询到Ingress后，就可以通过**kubctl describe ingress**命令查看`Ingress`对象是否按照我们的定义成功代理了`app-service`这个`Service`：

```
➜  ~ kubectl describe ingress app-ingress
Name:             app-ingress
Namespace:        default
Address:          192.168.64.4
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host             Path  Backends
  ----             ----  --------
  app.example.com
                   /   app-service:80 (10.1.0.24:3000,10.1.0.25:3000)
Annotations:       kubernetes.io/change-cause: kubectl apply --filename=app-ingress.yaml --record=true
Events:            <none>
```

上面输出里的`Rlues`部分可以清楚的看到，把**Host:  app.example.com**所有请求（定义了Path是/）都代理到了后端`app-service`的80端口，Service后面的`Pod`正是它的`Endpoints`，与上面的**kubctl describe svc app-service**命令输出里的`Endpoints`信息一致。

接下来在`/etc/hosts`文件里追加下面的内容，就能通过域名访问我们的服务了：

```
192.168.64.4 app.example.com
```

你们在练习的时候，可以自己尝试新增一个`Service`，然后更新Ingress，再指定一个`/v2`之类的Path，让所有匹配这个规则的请求都能路由给新的Service。

## 接下来

`Ingress`还有很多其他的配置，想要简单的讲完，还是挺难的。最常用的比如怎么设置TLS私钥和证书这些配置在**Kubernetes官方文档-Ingress**[2] 部分都有提到，后面自己练习的时候可以试试给`Ingress`启用`HTTPs`访问的功能。