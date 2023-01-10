# UTXO 与账户余额模型

目前的绝大多数区块链项目不是使用 *UTXO 模型*作为底层的数据结构，就是使用*账户余额模型*存储交易相关的信息。

![internal-implementaion-of-blockchain](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101110607.png)

在这篇文章中，我们会分别展示两种不同区块链模型的实现方式以及优缺点，我们会以 Bitcoin 和 Ethereum 为例分别介绍 UTXO 模型和账户余额模型。

## 区块与区块链

在具体介绍 UTXO 模型和账户余额模型之前，我们不得不首先介绍它们两者、甚至所有区块链应用中最重要的概念和数据结构，也就是*区块（Block）*。区块链其实就是由一个长度不断增长的链表组成的，其中包含了很多记录，也就是区块。

![blockchain-and-blocks](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101110631.png)

在上述区块链网络中，绿色的区块都被包含在主链中，所有黄色的区块都是孤块（Orphan Block），它们没有被主链接受，在每一个区块链网络中只能有一条主链，也就是**最长的有效链**，也是当前区块链网络中所有节点达成的共识。

### 区块

想要了解区块到底是什么，最简单快捷的办法就是分析它的数据结构，以 Bitcoin 中的区块 [#514095](https://blockchain.info/rawblock/00000000000000000018b0a6ae560fa33c469b6528bc9e0fb0c669319a186c33) 为例：

```javascript
{
　　"hash":"00000000000000000018b0a6ae560fa33c469b6528bc9e0fb0c669319a186c33",
　　"confirmations":1009,
　　"strippedsize":956228,
　　"size":1112639,
　　"weight":3981323,
　　"height":514095,
　　"version":536870912,
　　"versionHex":"20000000",
　　"merkleroot":"5f8f8e053fd4c0c3175c10ac5189c15e6ba218909319850936fe54934dcbfeac",
　　"tx":[
　　  // ...
　　],
　　"time":1521380124,
　　"mediantime":1521377506,
　　"nonce":3001236454,
　　"bits":"17514a49",
　　"difficulty":3462542391191.563,
　　"chainwork":"0000000000000000000000000000000000000000014d2b41a340e60b72292430",
　　"previousblockhash":"000000000000000000481ab128418847dc25db4dafec464baa5a33e66490990b",
　　"nextblockhash":"0000000000000000000c74966205813839ad1c6d55d75f95c9c5f821db9c3510"
}
```

在这个 Block 的结构体中，`previousblockhash` 和 `merkleroot` 是两个最重要的字段；前者是一个哈希指针，它其实是前一个 Block 的哈希，通过 `previousblockhash` 我们能递归地找到全部的 Block，也就是整条主链，后者是一个 Merkle 树的根，Merkle 树中包含整个 Block 中的全部交易，通过保存 `merkleroot`，我们可以保证当前 Block 中任意交易都不会被修改。

Ethereum 的区块链模型虽然与 Bitcoin 有非常大的不同，但是它的 Block 结构中也有着类似的信息：

```javascript
{
   "jsonrpc":"2.0",
   "result":{
      "author":"0x00d8ae40d9a06d0e7a2877b62e32eb959afbe16d",
      "difficulty":"0x785042b0",
      "extraData":"0x414952412f7630",
      "gasLimit":"0x47b784",
      "gasUsed":"0x44218a",
      "hash":"0x4de91e4af8d135e061d50ddd6d0d6f4119cd0f7062ebe8ff2d79c5af0e8344b9",
      "logsBloom":"0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
      "miner":"0x00d8ae40d9a06d0e7a2877b62e32eb959afbe16d",
      "mixHash":"0xb8155224974967443d8b83e484402fb6e1e18ff69a8fc5acdda32f2bcc6dd443",
      "nonce":"0xad14fb6803147c7c",
      "number":"0x2000f1",
      "parentHash":"0x31919e2bf29306778f50bbc376bd490a7d056ddfd5b1f615752e79f32c7f1a38",
      "receiptsRoot":"0xa2a7af5e3b9e1bbb6252ba82a09302321b8f0eea7ec8e3bb977401e4f473e672",
      "sealFields":[
         "0xa0b8155224974967443d8b83e484402fb6e1e18ff69a8fc5acdda32f2bcc6dd443",
         "0x88ad14fb6803147c7c"
      ],
      "sha3Uncles":"0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
      "size":"0x276",
      "stateRoot":"0x87e7e54cf229003014f453d64f0344e2ba4fc7ee3b95c7dd2642cca389fa1efe",
      "timestamp":"0x5a10968a",
      "totalDifficulty":"0x1804de0c47ffe1",
      "transactions":[...],
      "transactionsRoot":"0xc2091b032961ca23cf8323ea827e8956fe6dda9e68d75bcfaa8b910035397e35",
      "uncles":[]
   },
   "id":1
}
```

`parentHash` 和 `transactionsRoot` 分别对应着 Bitcoin 中 `previousblockhash` 和 `merkleroot`，这两者在整个区块链网络中是非常重要的。

### 哈希指针

Block 结构体中的哈希指针在区块链中有两个作用，它不仅能够连接不同的区块，还能够对 Block 进行验证，保证 Block 中的数据不会被其他恶意节点篡改。

![block-and-prevhash](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101111329.png)

除了第一个 Block，每一个 Block 中的 `prev_hash` 都是前一个 Block 的哈希，如果某一个节点想要修改主链上 Block 的交易，就会改变当前 Block 的哈希，后面的 Block 就没有办法通过 `prev_hash` 找到前面的链，所以当前节点篡改交易的行为就会被其他节点发现。

### Merkle Tree

另一个字段 `merkleroot` 其实就是一个 [Merkle 树](https://en.wikipedia.org/wiki/Merkle_tree) 的根节点，它其实是一种使用哈希指针连接的数据结构；虽然 Merkle 树有叶节点和非叶节点，但是它只有叶节点会存储数据，所有的非叶结点都是用于验证数据完整性的哈希。

![merkle-tree](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101111552.png)每一个 Block 中的全部交易都是存储在这个 Merkle 树中并将 `merkleroot` 保存在 Block 的结构体中，保证当前 Block 中任意交易的篡改都能被立刻发现。

### 小结

`prev_hash` 和 `merkleroot` 分别通过『指针』的方式保证所有的 Block 和交易都是连接起来的，最终保证 Block 和交易不会被恶意节点或攻击者篡改，几乎全部的区块链项目都会使用类似方式连接不同的 Block 和交易，这可以说是区块链项目的基础设施和标配了。



## UTXO 模型

作为最早出现的加密货币，Bitcoin 就采用了 UTXO 模型作为其底层存储的数据结构，其全称为 *Unspent Transaction output*，也就是未被使用的交易输出。

在 Bitcoin 以及其他使用 UTXO 模型的加密货币中，某一个『账户』中的余额并不是由一个数字表示的，而是由当前区块链网络中所有跟当前『账户』有关的 UTXO 组成的。

![balance-with-utxo-model](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101111654.png)

上图中所有绿色的交易输出才是 UTXO，红色的交易输出已经被当前『账户』使用了，所以在计算当前账户的余额时只会考虑绿色的交易输出，也就是 UTXO。

```javascript
{
   "addr":"14uhqGYDEhqwfdoP59QdLWdt4ha5CHttwQ",
   "n":1,
   "script":"76a9142ae017a5bd24a3f935897085253e503fbfd66f4e88ac",
   "spent":false,
   "tx_index":335926477,
   "type":0,
   "value":21680000
}
```

上述的 UTXO 中包含了很多信息，例如：包含当前 UTXO 属于的交易索引 `tx_index`、交易接收方的地址 `addr`、交易的数额 `value`。

### 交易

UTXO 其实就是交易的一部分，基于 UTXO 模型的交易由输入和输出两个部分组成：

```javascript
{
   "txid":"5be7a9e47f56c98e5297a44df52da0475f448ece98bb51489103cdf70653092f",
   "hash":"5be7a9e47f56c98e5297a44df52da0475f448ece98bb51489103cdf70653092f",
   "version":1,
   "size":224,
   "vsize":224,
   "locktime":0,
   "vin": [...],
   "vout": [...],
   "hex":"0100000001a90b4101e6cbb75e1ff885b6358264627581e9f96db9ae609acec98d72422067000000006b483045022100c42c89eb2b10aeefe27caea63f562837b20290f0a095bda39bec37f2651af56b02204ee4260e81e31947d9297e7e9e027a231f5a7ae5e21015aabfdbdb9c6bbcc76e0121025e6e9ba5111117d49cfca477b9a0a5fba1dfcd18ef91724bc963f709c52128c4ffffffff02a037a0000000000017a91477df4f8c95e3d35a414d7946362460d3844c2c3187e6f6030b000000001976a914aba7915d5964406e8a02c3202f1f8a4a63e95c1388ac00000000",
   "blockhash":"0000000000000000000c23ca00756364067ce5e815deb5982969df476bfc0b5c",
   "confirmations":5,
   "time":1521981077,
   "blocktime":1521981077
}
```

交易对象中的大多数其它字段并没有什么意义，只是对当前的交易进行了一些描述，让我们能够更好的理解当前交易的相关信息，例如：上述交易中的 `size` 和 `vsize` 字段可以从交易其他部分计算出来。

在每一笔合法的交易中，所有的输入的 `value` 之和必须大于所有输出的 `value` 之和，这两者之间的差值就是矿工费：

```
sum(inputs.value) = sum(outputs.value) + fee
```

基于 UTXO 的交易模型，与我们在日常生活中使用纸币的场景是非常相似的，每一张纸币都是**不可分割的**整体，当我们想要使用现金购买商品或者服务时，往往都会获得**找零**。

```
inputs = price + change + fee
```

每一个 UTXO 和纸币一样，只可能有两种状态，要么是没有被花费的，要么就是已经被花费，所有权变成了其他人或者地址，成为其他地址的 UTXO。

![change-or-consolidate](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101111928.png)

在基于 UTXO 的区块链网络中，除了找零（Change）非常常见之外，将多个 UTXO 整合（Consolidate）成一个 UTXO 的操作也比较常见，在 Bitcoin 的网络中，无论当前的 UTXO 中有多少钱，每一个 UTXO 的大小都是差不多的，所以在进行大额转账时，往往需要多个 UTXO 作为输入，这样会明显的增加交易的矿工费。

#### 输入和签名

UTXO 模型中的每一笔交易都是由多个交易输入组成的，这些输入其实就是 UTXO + 签名：

```javascript
{
   "vin":[
      {
         "txid":"672042728dc9ce9a60aeb96df9e9817562648235b685f81f5eb7cbe601410ba9",
         "vout":0,
         "scriptSig":{
            "asm":"3045022100c42c89eb2b10aeefe27caea63f562837b20290f0a095bda39bec37f2651af56b02204ee4260e81e31947d9297e7e9e027a231f5a7ae5e21015aabfdbdb9c6bbcc76e[ALL] 025e6e9ba5111117d49cfca477b9a0a5fba1dfcd18ef91724bc963f709c52128c4",
            "hex":"483045022100c42c89eb2b10aeefe27caea63f562837b20290f0a095bda39bec37f2651af56b02204ee4260e81e31947d9297e7e9e027a231f5a7ae5e21015aabfdbdb9c6bbcc76e0121025e6e9ba5111117d49cfca477b9a0a5fba1dfcd18ef91724bc963f709c52128c4"
         },
         "sequence":4294967295
      }
   ]
}
```

上述 JSON 其实就是 Bitcoin 交易 [#338309214](https://blockchain.info/rawtx/5be7a9e47f56c98e5297a44df52da0475f448ece98bb51489103cdf70653092f) 的输入，这里的 `prev_out` 就来自于另一笔交易 [#338283541](https://blockchain.info/rawtx/672042728dc9ce9a60aeb96df9e9817562648235b685f81f5eb7cbe601410ba9) 的输出，通过不停的回溯，最终我们会找到当前交易涉及的 Coinbase，也就是当前 UTXO 相关 Bitcoin 被挖出来的 Block 的首笔交易。

通过 `txid` 和 `vout` 两个字段，我们能够在区块链网络中定位到唯一一个 UTXO，这个 UTXO 加上持有当前 UTXO 的地址对交易的签名构成了一个交易输入。

#### 输出

每一个交易都可能会有多个输出，也就是 `vout` 数组，每一个 `vout` 都可以指向不同的地址，其中也有当前输出包含的值 `value`，在这里也就是 Bitcoin 的单位：

```javascript
{
   "vout":[
      {
         "value":0.10500000,
         "n":0,
         "scriptPubKey":{
            "asm":"OP_HASH160 77df4f8c95e3d35a414d7946362460d3844c2c31 OP_EQUAL",
            "hex":"a91477df4f8c95e3d35a414d7946362460d3844c2c3187",
            "reqSigs":1,
            "type":"scripthash",
            "addresses":[
               "3CcqrGq4oQcfx3u75ijj4tDiqf4HJvhoeP"
            ]
         }
      },
      {
         "value":1.84809190,
         "n":1,
         "scriptPubKey":{
            "asm":"OP_DUP OP_HASH160 aba7915d5964406e8a02c3202f1f8a4a63e95c13 OP_EQUALVERIFY OP_CHECKSIG",
            "hex":"76a914aba7915d5964406e8a02c3202f1f8a4a63e95c1388ac",
            "reqSigs":1,
            "type":"pubkeyhash",
            "addresses":[
               "1GedHcxdxq2tab98hqAmREUK9BBYHKznof"
            ]
         }
      }
   ]
}

```

每一个未被使用的 `vout` 就是一个 UTXO（Unspent Transaction Output），我们可以通过其中的 `addresses` 字段找到持有当前输出的地址。

### 小结 

UTXO 模型通过链式的方式组织所有交易的输入和输出，每一个交易的输出最终都能追寻到一个 Coinbase，也就是当前 Bitcoin 被挖出时的区块的第一笔交易。

由于在 UTXO 中没有账户的概念，所以并行地处理交易不会出现任何问题，同时不可变的账本能够让我们在 Bitcoin 节点快速更新时，也能分析某一时刻整个网络中数据的快照。

虽然 UTXO 模型的不可变账本条目带来一些好处，但是当我们需要计算某个地址中的余额时，需要遍历整个网络中的全部相关区块，同时，并行的处理交易虽然可行，不过并行的创建交易却会出现很多问题，例如多笔交易使用了同一个 UTXO，导致双花，最终只有一笔交易能够被网络确认。

UTXO 模型确实能够解决区块链世界中的各种问题，它的核心思想就是保证已经写入的数据不可变，链式的 UTXO 就是基于这一核心思想的，通过哈希指针连接不同交易的输入和输出，保证所有交易的合法性。

## 账户余额模型

与 UTXO 模型不同的就是账户余额模型，它跟现实世界中的银行账户非常相似，Ethereum 就使用了账户余额模型存储区块链中的数据。

![ethereum-and-accounts-mode](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101112329.png)

相比于 UTXO 模型，账户余额模型更加容易实现和理解，如果忽略很多 Ethereum 的实现细节，那么在整个网络中只存在三种对象：账户、交易和区块。在这里，我们会介绍该模型中的三个最重要的概念，虽然 Block 并不是账户余额模型中独有的概念，但是我们也会介绍它在 Ethereum 中有什么特殊之处。

### 账户

Ethereum 其实就是一个巨大的状态机，其中的状态都是由多个账户组成的，每一个账户都包含四个字段 `(nonce, ether_balance, contact_code, storage)`：

![ethereum-accounts](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101112648.png)

在 Ethereum 中有两种类型的账户，一种是被私钥控制的账户，它没有任何的代码，与 Bitcoin 地址基本有完全相同的功能，能够向网络中发送已签名的交易；另一种是被合约代码控制的账户，能够在每一次收到消息时，执行保存在 `contract_code` 中的代码，所有的合约在网络中都能够响应其他账户的请求和消息并提供一些服务。

所有账户的 `nonce` 都必须从 `0` 开始递增，当前账户每使用 `nonce` 签发并广播一笔交易之后，都会将其 `+1`；UTXO 模型决定了来自同一地址的多笔金额相同的交易完全不同，从原理上避免了重放攻击，因为账户余额模型中不存在 UTXO，交易仅仅是账户 `ether_balance` 的变动，所以在这里引入 `nonce` 来解决重放攻击的问题。

#### 合约账户

被私钥控制的账户与 Bitcoin 中地址其实差不多，所以没有什么可以解释的，这里简单介绍一些合约账户。目前 Ethereum 网络上最多的合约账户应该就是 ERC20 的合约了，我们平时经常说的 Token 就是 Ethereum 上的合约，这些合约其实也是 Ethereum 账户：

```c
contract ERC20Interface {
    function totalSupply() public constant returns (uint);
    function balanceOf(address tokenOwner) public constant returns (uint balance);
    function allowance(address tokenOwner, address spender) public constant returns (uint remaining);
    function transfer(address to, uint tokens) public returns (bool success);
    function approve(address spender, uint tokens) public returns (bool success);
    function transferFrom(address from, address to, uint tokens) public returns (bool success);

    event Transfer(address indexed from, address indexed to, uint tokens);
    event Approval(address indexed tokenOwner, address indexed spender, uint tokens);
}
```

Token 的转账其实就是合约的调用，所有的账户余额都是存储在这个合约的 `balances` 中：

```c
mapping(address => uint256) balances;
```

每一笔 Token 的转账都会改变这个 `mapping` 中对应地址的值并发出 `Transfer` 事件，这也就是 Token 的实现原理；部署 Token 的智能合约其实非常简单，很多加密货币项目其实都只是一个 ERC20 的 Token，发行的成本几乎为 0。

### 交易

由于没有 Bitcoin 复杂的 UTXO 模型，Ethereum 的交易模型也简单，交易中没有输入和输出的 `TransactionIO` 结构，只有 `from` 和 `to` 两个地址：

```javascript
{
   "blockHash":"0xb4a992ff99a487db8421f516be998920f06dfe5d355d88e3b7f22b7422e6340d",
   "blockNumber":"0x24f85c",
   "chainId":null,
   "condition":null,
   "creates":null,
   "from":"0x8b56adcf332ff80a1f1bf433975dcb28b730d110",
   "to":"0xe94b04a0fed112f3664e45adb2b8915693dd5ff3",
   "value":"0x10d43fb8311ca800",
   "gas":"0x2062a",
   "gasPrice":"0x560aab7c5",
   "hash":"0xfea448d11cfa863c8b3c38d3b65649e66c1957f9ac16638e3a0edff45a6b3d84",
   "input":"0x0f2c9329000000000000000000000000fbb1b73c4f0bda4f67dca266ce6ef42f520fbb98000000000000000000000000e592b0d8baa2cb677034389b76a71b0d1823e0d1",
   "nonce":"0x3fe",
   "publicKey":"0x765b0f012e49f6a4cc5c917fb176984b24814bdaf5f9464db1a7f9ffcc730cb678f69e49f78aa9de8249cce138bbc25cf8842374d8d09089dff7f1ef6906f4fb",
   "r":"0x4275d35821dec971f6d58c2adae077ffcdfa3ec74af542a2d29ab4e5239d8b25",
   "raw":"0xf8b48203fe850560aab7c58302062a94e94b04a0fed112f3664e45adb2b8915693dd5ff38810d43fb8311ca800b8440f2c9329000000000000000000000000fbb1b73c4f0bda4f67dca266ce6ef42f520fbb98000000000000000000000000e592b0d8baa2cb677034389b76a71b0d1823e0d11ca04275d35821dec971f6d58c2adae077ffcdfa3ec74af542a2d29ab4e5239d8b25a036221b525c758c45e60f964eec698ae33208dfa74bea7f77dff002ceec418b0a",
   "s":"0x36221b525c758c45e60f964eec698ae33208dfa74bea7f77dff002ceec418b0a",
   "standardV":"0x1",
   "transactionIndex":"0x8",
   "v":"0x1c"
}
```

交易的手续费也不再是交易输入输出 `value` 的差值，而是使用 `gas` 和 `gasPrice` 来表示，为了防止重放攻击，这里也引入了 `nonce` 字段。

![ethereum-transaction](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101112849.png)

`(nonce, from, to, value, input)` 是一个 Transaction 包含的最重要的几个字段，通过 `nonce` 防止重放攻击，`from` 和 `to` 分别表示了当前交易的发出者和接受者，`value` 是当前交易包含的 `Ether`，`input` 中包含了合约调用相关的二进制信息。

每当一个 Transaction 被 Ethereum 主网挖到后，`from` 和 `to` 账户的 `Ether` 余额就会变动，Ethereum 就像一个状态机，它接受一个又一个的 Transaction 并不停改变自己的状态。

![state-machine](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/20221101112909.png)

### 小结

账户余额模型是一种非常容易理解的区块链应用模型，它与我们生活中的账户模型非常相似，只是为了保证账户的安全，使用了签名以及 `nonce` 的机制阻止恶意的攻击。这种基于账户余额模型的应用包含了一个包含所有账户余额的全局状态，在进行转账时，需要由节点对账户的余额进行验证，判断当前账户是否有足够的 `Ether` 进行转账。

## 总结

无论是 UTXO 模型还是账户余额模型，都能够很好地解决区块链世界中的『安全』问题，保证交易的合法，从原理上杜绝一些可能的攻击行为，实现原理的不同其实也只是由于出发点不同，在设计时权衡了利弊；UTXO 模型相比于账户余额模型有以下的两个优点：

- 如果用户启用了新的地址用于转账和交易，新地址与原地址之间的关系很难被追踪，更好地保证用户的隐私；
- UTXO 模型理论上来说可以并行地利用不同的 UTXO 签发多笔交易，并广播到网络中；

而账户余额模型也有它的优点：

- 非常容易理解和编码实现；
- 每一笔交易都只需要有一个签名，交易的输入和输出都是地址，能够节省存储空间；
- 由于在区块链层级没有『币的来源』这一概念，很难实现对来源的追踪和回溯；
- 因为创建交易时不需要对过去的 UTXO 进行签名，可以从任何一个时间点开始同步区块的状态，利于编写轻量级客户端；

总而言之，软件开发到最后就是进行权衡的过程，选择一些优势必然会在另外一些领域上处于劣势，如何在不同的需求进行权衡是开发区块链应用以及所有应用都需要考虑的事情。