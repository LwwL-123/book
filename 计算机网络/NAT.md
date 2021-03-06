

# NAT

首先端口映射和内网穿透做到的效果其实是一样的，都是把本机的端口暴露给公网使用。

**端口映射**：端口映射就是将内网中的主机的一个端口映射到外网主机的一个端口，提供相应的服务。当用户访问外网IP的这个端口时，服务器自动将请求映射到对应局域网内部的机器上。

​	我们在内网中有一台Web服务器，但是外网中的用户是没有办法直接访问该服务器的。于是我们可以在路由器上设置一个端口映射，只要外网用户访问路由器ip的80端口，那么路由器会把自动把流量转到内网Web服务器的80端口上。并且，在路由器上还存在一个Session，当内网服务器返回数据给路由器时，路由器能准确的将消息发送给外网请求用户的主机。在这过程中，路由器充当了一个反向代理的作用，他保护了内网中主机的安全。

**端口转发：**端口转发（Port forwarding），有时被叫做隧道，是安全壳（SSH） 为网络安全通信使用的一种方法。

​	比如，我们现在在内网中，是没有办法直接访问外网的。但是我们可以通过路由器的NAT方式访问外网。假如我们内网现在有100台主机，那么我们现在都是通过路由器的这一个公网IP和外网通信的。那么，当互联网上的消息发送回来时，路由器是怎么知道这个消息是给他的，而另外消息是给你的呢？这就要我们的ip地址和路由器的端口进行绑定了，这时，在路由器中就会有一个内网ip和路由器端口对应的一张表。当路由器的10000端口收到消息时，就知道把消息发送给他，而当20000端口收到消息时，就知道把消息发送给你。这就是端口转发，其转发一个端口收到的流量，给另一个主机。



如果是端口转发，我们内网的ip是主动发起一个连接请求，这时NAT会建立你的端口，并把他转换为相应的公网ip+端口。但是如果是服务端主动发起连接请求，NAT是没有建立相关的映射关系的，所以不能由服务端主动发起请求。

**如果是两台都不具有公网ip的电脑想要连接，无论是谁先发起请求，都无法进行连接，因为总有一个NAT没有建立响应的关系表**

## 1. NAT分类

NAT(Network Address Translators)，网络地址转换：网络地址转换是在IP地址日益缺乏的情况下产生的，它的主要目的就是为了能够地址重用。NAT分为两大类，基本 的NAT和NAPT(Network Address/Port Translator)。

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173359.png)


### 1.1 基础型NAT

​    仅将内网主机的私有IP地址转换成公网的IP地址，并不将TCP/UDP端口信息进行转换，分为静态NAT和动态NAT。



### 1.2 NAPT

​    NAPT不但会改变经过这个NAT设备的IP数据报的IP地址，还会改变IP数据报的TCP/UDP端口。



#### 1.2.1 锥型NAT

1. **完全锥型（Full Cone NAT）**：在不同内网的主机A和B各自连接到服务器C，服务器收到A和B的连接后知道了他们的公网地址和NAT分配给他们的端口号，然后把这些NAT地址和端口号交叉告诉B和A。A和B给服务器所打开的“孔”可以给任何主机使用。如一私网主机地址是192.168.1.100:30000发至公网的所有请求都映射成一个公网地址172.1.20.100:20000，192.168.1.100:30000可以接收任何主机发给172.1.20.100:20000的数据报文。

2. **受限制锥型（Restricted cone）**：主机A和B同样需要各自连接服务器C，同时把A和B的地址告诉B和A，但一般情况下它们只能与服务器通信。要想直接通信需要发送消息给服务器C，如主机A发送一个UDP消息到主机B的公网地址上，与此同时，A又通过服务器C中转发送一个邀请信息给主机B，请求主机B也给主机A发送一个UDP消息到主机A的公网地址上。这时主机A向主机B的公网IP发送的信息导致NAT A打开一个处于主机A的和主机B之间的会话，与此同时，NAT B也打开了一个处于主机B和主机A的会话。一旦这个新的UDP会话各自向对方打开了，主机A和主机B之间才可以直接通信。
3. **端口受限锥型（Port-restricted）：**与受限制锥型类似，与之不同的是还要指定端口号。

#### 1.2.2对称NAT（Symmetric）

​    对不同的外网IP地址都会分配不同的端口号。

### 1.3 两者区别

​    对称NAT是一个请求对应一个端口，非对称NAT是多个请求对应一个端口(象锥形，所以叫Cone NAT)。



## 2、网络打洞

### 2.1 打洞条件

1. 中间服务器保存信息、并能发出建立UDP隧道的命令
2. 网关均要求为Cone NAT类型。Symmetric NAT不适合。
3. 完全圆锥型网关可以无需建立udp隧道，但这种情况非常少，要求双方均为这种类型网关的更少。



1. 假如X1网关为Symmetric NAT， Y1为Address Restricted Cone NAT 或Full Cone NAT型网关，各自建立隧道后，A1可通过X1发送数据报给Y1到B1(因为Y1最多只进行IP级别的甄别)，但B2发送给X1的将会被丢弃（因为发送来的数据报中端口与X1上存在会话的端口不一致，虽然IP地址一致），所以同样没有什么意义。
2. 假如双方均为Symmetric NAT的情形，新开了端口，对方可以在不知道的情况下尝试猜解，也可以达到目的，但这种情形成功率很低，且带来额外的系统开支，不是个好的解决办法。pwnat工具据说可以实现。
3. 不同网关型设置的差异在于，对内会采用替换IP的方式、使用不同端口不同会话的方式，使用相同端口不同会话的方式；对外会采用什么都不限制、限制IP地址、限制IP地址及端口。
4. 这里还没有考虑同一内网不同用户同时访问同一服务器的情形，如果此时网关采用AddressRestricted Cone NAT 或Full Cone NAT型，有可能导致不同用户客户端可收到别人的数据包，这显然是不合适的。
   

### 2.2 打洞流程

不同的网络拓扑NAT打洞的方法和流程有所区别。

#### 2.2.1 同一个NAT设备下

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173404.png)

1. clinet A与Server S建立UDP连接，公共NAT（155.99.25.11）给client A分配一个公网端口62000；
2. client B与Server S建立UDP连接，公共NAT（155.99.25.11）给client B分配一个公网端口62005；
3. client A通过Server S发送一个消息要求连接client B，S给A回应B的公网和私网地址，并转发A的公网和私网地址给B；
4. A和B根据获取的地址试图直接发送UDP数据报文；是否成功取决于NAT设备是否支持hairpin translation（端口回流）。——打开端口回流相当于与client A的数据经过NAT设备转发后才到达client B，即从外网NAT接口绕了一圈再访问到同一个子网里的client B。（优点是可以防止内部攻击）
   

#### 2.2.2 不同NAT设备下

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173406.png)

假设Client A打算与Client B直接建立一个UDP通信会话。如果Client A直接给Client B的公网地址138.76.29.7:31000发送UDP数据，NAT B很可能会无视进入的数据（除非是Full Cone NAT），Client B往Client A直接发信息也类似。

为了解决上述问题，在Client A开始给Client B的公网地址发送UDP数据的同时，Client A给Server S发送一个中继请求，要求Client B开始给Client A的公网地址发送UDP信息。Client A往Client B的输出信息会导致NAT A打开一个Client A的内网地址与Client B的外网地址之间的通讯会话，Client B往Client A亦然。当两个方向都打开会话之后，Client A和Client B就能直接通讯，而无须再通过Server S了。


UDP打洞技术有许多有用的性质。一旦一个的P2P连接建立，连接的双方都能反过来作为“引导服务器”来帮助其他中间件后的客户端进行打洞，极大减少了服务器的负载。应用程序不需要知道中间件是什么（如果有的话），因为以上的过程在没有中间件或者有多个中间件的情况下也一样能建立通信链路。

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173408.png)

1. A使用4321端口与S连接，NAT给回话在NAT分配外网62000端口（155.99.25.11:62000）与S连接；同理B以相同的方式与S连接，分配的外网地址端口是138.76.29.7:31000。
2. A往S注册消息包里包含里A的私有地址10.0.0.1:4321，此时S保存了A的地址；S给A临时分配了一个用于公网的地址（155.99.25.11:62000），同时用于观察外网数据包。
3. 同理B往S注册的消息包里也包含里B的地址，NAT同样给B临时分类了一个外网地址（138.76.29.7:31000）。
4. Client A根据以上已知信息通过打洞的方式与B连接UDP通信：
   

- Client A发送请求消息，寻求连接B；
- S给A回应B的外网和内网地址，通给给B发送A的外网和内网地址；
- A和B开始利用这些地址尝试直接发送UDP报文给彼此，不幸的是，此时A和B都无法接收对应的消息。因为A和B都是在不同的私有网络中，A和B之前都是与S通信回话，并没有与对方建立回话；即A没有为B打开一个洞，B也没有为A打开一个洞。这个过程的第一个报文需要会被拒绝同时打开对应的“洞”，随后才可以直接通信，具体如下：
  - A给B公网地址（10.0.0.1:4321 to 138.76.29.7:31000）发送的第一个报文，实际上是在A的NAT私有网络上“打洞”来为新识别的地址(10.0.0.1:4321 138.76.29.7:31000) 建立UDP会话,并经主网地址(155.99.25.11:62000 138.76.29.7:31000)来传送。
  - 如果A发送到B的公网地址的消息在B发送到A的第一个消息越过B自己的NAT之前到达B的NAT，那么B的NAT可能会将A的入站消息解释为非请求的传入通信量并丢弃它。
  - 同理，B给A公网地址方法的第一个消息也会在B的NAT上“打洞”来为地址（10.1.1.3:4321, 155.99.25.11:62000）建立回话。
  - 随后可以正常P2P通信。



### 2.3 不同NAT的穿透性

我们知道，内网穿透的作用是将两台处于NAT中的主机连接，所以上述介绍了四种NAT的类型，如果对其进行两两组合，共有10种组合方式。事实上，不同的组合方式在进行穿透时的方法也不同，甚至存在两种组合方式无法进行内网穿透（当然，如果全程用一个服务器进行转发自然是可以的，但是我们这里不考虑这种方法，另外目前有一些方式也可以实现这两种的内网穿透，只不过目前还不成熟，成功率较低，事实上不只是目前，我个人认为以后也不会成熟，因为这两种组合无法穿透是nat在设计上就存在的逻辑问题，或者说nat在设计之初就没有考虑这两种的穿透问题），下图将这些组合以及是否可以穿透列了出来。

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173412.webp)



 内网穿透的思路有三种，所有可以进行内网穿透的的组合都可以基于这三种方法或基于这三种方法稍微拓展来实现内网穿透，下面我们讲讲这三种方法。 

准备工作：内网穿透是使两个处于NAT中的网络进行连接，所以假设其中一个NAT网关是A，另一个NAT网关是B。另外内网穿透都需要有一个中心服务器来帮助，A和B都需要先连接到这台中心服务器，这里假定中心服务器的名字是server1，所以大致上拓扑应该是这样的。

![img](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20220325173414.webp)

#### 2.3.1 完全锥型NAT和完全锥型NAT进行穿透

假设A和B都是完全锥型NAT，在A和B都连接到server1后，A和B都可以借助server1的转发互相发送消息，那么此刻A和B就可以知道对方的公网ip，以及对方和server1连接的时候，使用的端口是什么（假设是100），因为两者和server1进行通信的端口已经进行了NAT映射（不然怎么通信呢），所以二者的100端口其实已经完成映射，又因为二者都在完全锥型NAT下，此刻A只需要直接给B的100端口发送建立连接的请求，B给A的100端口回复同意建立连接的请求，二者即可建立UDP连接（当然UDP连接建立要比这复杂点，不过本文重点不在于如何建立UDP连接） 

#### 2.3.2 ip限制型NAT和ip限制型NAT进行穿透

假设A和B都是ip限制型NAT，在A和B都连接到server1后，A和B都可以借助server1的转发互相发送消息，A会先发送一个UDP请求（假设自己的端口用100，目标端口用200）到B的公网ip上，理论上来说，因为B的NAT网关中，200端口没有建立NAT映射，所以这个数据包会被丢弃，但是在A发送给B的UDP请求后，A会通过server1给B发送一个邀请，邀请B也发送一个UDP请求给A（此刻B自己用的端口是200，目标端口是100），注意，在B收到来自A的UDP请求后，虽然A的数据包被B丢弃了，但是此刻，网关A暂时的建立了一个NAT映射，等待B返回的信息，虽然数据包已经被丢弃了，但是A不知道，所以A会稍微等一会B。这时，B收到了A的邀请，给A发送了一个建立连接的请求，此刻A的NAT网关恰巧暂时建立了NAT映射，所以A就可以收到B的UDP请求，接着A会给B发送一个同意建立连接的请求，因为此刻B刚发完请求在等A的回信，所以B的NAT网关也会暂时的建立一个NAT映射，所以A同意建立连接的请求就不会被B的NAT网关丢弃，最终，二者就建立了一个稳定的UDP连接



#### 2.3.3 端口限制型NAT和端口限制型NAT进行穿透

  原理和第二种其实差不多



借助这三种思想，我们就可以完成不需要服务器一直进行中转的内网穿透，而花生壳之类的内网穿透工具，其实就是借助服务器进行中转的，那么为什么对称型NAT不能使用“ip限制型NAT和ip限制型NAT进行穿透”的思想和对称型NAT穿透呢。

仔细观察一下，在第二种方法中，A邀请B给其发送一个UDP请求，在邀请的信息中，A指明了B的UDP请求的目标端口，因为在锥型NAT中，主机A的一个端口和NAT网关的映射是固定的，所以主机A可以通过server1知道自己给B发送请求是打开的端口是哪一个，也可以知道B给自己发送请求是打开的端口是哪一个，但是当换到对称NAT中时，由于一个连接对应NAT网关上的一个端口，所以主机A无法确定自己通过哪一个端口给B发信息，同样无法确定B会通过哪一个端口给自己发信息，所以二者无法建立连接。



那么端口限制型又为什么不能和对称型进行穿透呢，因为端口限制型对端口存在要求，但是我们无法确定对称型中分配的端口是哪一个，所以无法建立通信

想让二者能建立通信，目前已知较好的的方法是让处于端口限制型NAT中的电脑和对称型中的电脑建立65535个UDP连接，由于65535包含了全部的端口号，所以总有一个端口是对的。但是这样一来就在一瞬间占用了几乎所有的端口，这可能会导致对称型中的电脑来不及处理正常的TCP连接，并且会占用大量资源，而且连接的成功率也非常的低，所以我们通常认为二者不能进行穿透。 