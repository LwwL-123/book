# 深入理解 IPFS 

![深入理解 IPFS - 分层架构总览](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20211217151014.jpg)

我们首先从官方文档中一张比较经典的层级架构图开始，右边是 IPFS 的层级图，左边是对应现实网络中的一些实现形式。

### **(一) applications 应用层**

这个很好理解，就是基于 IPFS 网络开发的应用。我们可以在 awesome 看到一些应用。生态能不能繁荣，肯定是需要开发者参与。

### **（二） naming** **命名**

SFS 中文是自我认证文件系统，熟悉其他区块链项目的同学应该很好理解。比如比特币，我们采用非对称加密算法给节点生成了公钥和私钥。通过公钥生成的地址一旦参与转账行为，就会成为比特币网络中一份子，所有节点都知道这个地址。那么怎么证明这个地址是属于你，这就是一个自证明的过程，通过与公钥配对的私钥就可证明所有权。同样对于 IPFS 来说，也采用同样的方式，每个节点的 ID 都是全局唯一的存在，你可以通过 ID 对应的私钥来证明你是这个节点的主人。

举个应用的例子，你有个静态网站想要上传到 IPFS 上，但是网站是需要一直更新内容的，更新后 hash 值变了，在 IPFS 上访问的链接都会变。解决办法是将节点 ID 指向生成的 hash 上。用户访问节点 ID 即可 。

其实不仅仅是文件系统，身份认证（OpenID, OAuth2）可能也会由于区块链的发展出现新的契机。比如 ArcBlock 在推广的 DID

### **（三）merkledag**

merkledag 是 IPFS 的核心数据结构。比特币中用 merkle tree（默克尔树）快速校验区块数据完整性，使得轻节点钱包无需下载所有交易数据就能校验交易数据。而 merkledag 是一个有向无环图，对象模型和 Git 相似。这部分以后会详细说明。

### **（四）exchange 块交换**

因为 IPFS 是一个全网文件系统，当你上传一个文件的时候，文件会被分割成 block，然后被传输到最近的节点上（XOR）。这样当你再去获取这个文件的时候，节点间就会相互沟通，每个节点都有一个 wantlist 和 havelist，最终找到所有的 block，重新获取到文件。

### **（五）routing 路由**

其实在说到 exchange 的时候，有同学可能发现已经有点熟悉了，这不就是 BitTorrent 么。说的没错，不管是当年的电驴还是现在的迅雷，BT 网络一直很坚挺的存在着。IPFS 的很多设计都是借鉴于它 。说到 BT 的发展，最开始种子是存储到中央 Tracker 服务器上，这样一旦中央服务器挂了，这个种子也就废了，所以发展出了 DHT （分布式哈希表）。节点加入 DHT 网络，所有的资源都通过 hash 来寻址，无需中央节点介入，通过询问自己已知的节点，最终找到目标节点。

对于 IPFS 来说，内容和地址寻址都是同构hash，这样可以避免转换成本。虽然说 routing 的方式不止 DHT 一种，还有 mdns，DHCP。但是作为一个去中心化的项目来说，DHT 是 IPFS 路由必不可少的一项。

### **（六）network 网络**

IPFS 没有创造新的协议来做节点间通信，而是尽可能的兼容现存的所有协议，包括 tcp, http, quic 等等。兼容是通过一个 self-describing 的方式，如 `/ip4/7.7.7.7/tcp/6543`， 这种风格我们在 IPFS 其他 repos 中也都能看到身影，如 MultiHash, MultiAddress。不得不说，这种 self-describing 的形式在之后的协议升级上会有很大的好处。同样，面对各种网络环境，NAT 穿透也是必备的，IPFS 基于 ICE 框架，支持 TURN 和 STUN。





# DHT 网络

IPFS 的网络层源码在 libp2p 中，本文用 go-libp2p 做分析。

我们假设一个场景应用，有两个节点名字分别叫 earth 和 mars，然后他们分别加入了 DHT 网络，接下来他们需要找到对方，并能够互相发送消息。

### **（一）初始化节点**

首先我们需要初始化节点

```go
ctx := context.Background()
listenAddresses, _ := multiaddr.NewMultiaddr("/ip4/127.0.0.1/tcp/8004")
host, _ := 
		libp2p.New(ctx, libp2p.ListenAddrs([]multiaddr.Multiaddr{listenAddresses}...))
fmt.Println("host->", host.ID())
```

其实初始化就一行 `libp2p.New()`，可自定义参数，比如上面我们定义了监听地址和端口 `/ip4/127.0.0.1/tcp/8004`， 等同于 `127.0.0.1:8004` 不过自解释性更强。

再举个例子，`/ip4/1.2.3.4/tcp/4321/p2p/QmcEPrat8ShnCph8WjkREzt5CPXF2RwhYxYBALDcLC1iV6` 结尾有个 PeerId `QmcEPrat8ShnCph8WjkREzt5CPXF2RwhYxYBALDcLC1iV6`



那么不仅仅可以通过 ip + port 寻址，通过 PeerId 也可以直接定位到节点。

初始化后我们生成了一个节点，节点 ID 以 btc58encode 编码：QmcEPrat8ShnCph8WjkREzt5CPXF2RwhYxYBALDcLC1iV6，也就是上文的 PeerID。



接下来我们需要给 8004 监听的端口配置 handler

```go
func handleStream(stream network.Stream) {
	log.Println("Got a new stream!", stream)
}

host.SetStreamHandler(protocol.ID('/chat/1.0'), handleStream)
```

`handleStream` 这个函数的逻辑跟普通的 socket 编程一样，拿到 stream 往里读写数据就行，这里不细讲。



### **（二）加入 DHT 网络**

节点建立完成后，接下来就需要加入 DHT 网络了。

```go
// 加入 dht 网络
kademliaDHT, err := dht.New(ctx, host)
if err != nil {
	panic(err)
}
// 设置状态为 bootstrap 模式
if err = kademliaDHT.Bootstrap(ctx); err != nil {
	panic(err)
}

var wg sync.WaitGroup
// connect 到 bootstrap 节点
for _, peerAddr := range dht.DefaultBootstrapPeers {
	peerinfo, _ := peer.AddrInfoFromP2pAddr(peerAddr)
	wg.Add(1)
	go func() {
		defer wg.Done()
		if err := host.Connect(ctx, *peerinfo); err !=nil {
			log.Println(err)
		} else {
			log.Println("Connection established with bootstrap node:", *peerinfo)
		}
	}()
}
wg.Wait()
```

不管是比特币，以太坊，还是早前的 BT 网络，任何新节点加入网络都需要种子 (bootstrap) 节点作为起点，然后扩展自己的路由表，完成初始化动作。



### **（三）广而告之**

```go
// 广而告之
nodeName := "mars"
log.Println("Announcing ourselves...")
routingDiscovery := discovery.NewRoutingDiscovery(kademliaDHT)
discovery.Advertise(ctx, routingDiscovery, nodeName)
log.Println("Successfully announced!")
```

回到我们开头的场景，假设我们初始化一个节点名叫 `mars`，我们加入 DHT 网络后，需要让所有节点都知道我是 mars 节点 。

这里先简单介绍下，原理下篇文章再分析。nodeName 其实最后被转换为一个内容的 hash，节点通过 Advertise 这个方法告诉其他节点，它拥有这个 hash，然后其他节点就会记住，更新自己的路由表。等到有请求去做这个内容的寻址时，就会告诉对方谁有这个内容，或者谁和这个内容更接近。

### **（四）寻找节点**

```go
findName := "earth"
log.Println("Searching for other peers...")
peerChan, err := routingDiscovery.FindPeers(ctx, findName)
if err != nil {
	panic(err)
}
for peer := range peerChan {
	fmt.Println("found peer", peer.ID)
	if peer.ID == host.ID() {
		continue
	}

	stream, _ := host.NewStream(ctx, peer.ID, protocol.ID("/chat/1.0.0"))
	rw := bufio.NewReadWriter(bufio.NewReader(stream), bufio.NewWriter(stream))
	go readData(rw)
	go writeData(rw)
}
```

`FindPeers` 内在实现逻辑其实是找 `earth` 这个 hash 的地址，找到就和他建立一个双工的连接，正好和前面 handleStream 实现了服务端和客户端的通信。



### **(六) 完善**

上面的例子有个问题是，谁都可以宣称自己是 `mars` 节点，通信双方没法信任，所以这种模式适用聊天室的 channel 场景。通过将内容寻址改成节点寻址，就可找到可信通信方，当然前提是你知道你要通信的节点 ID。

代码如下：

```go
findID, err := peer.Decode("QmcZf59bWwK5XFi76CZX8cbJ4BhTzzA3gU1ZjYZcYW3dw1")
kademliaDHT.FindPeer(ctx, findID)
```