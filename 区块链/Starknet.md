# Starknet

Starknet 是一种无需许可的去中心化 Validity-Rollup（也称为“ZK-Rollup”）。它作为以太坊上的 L2 网络运行，使任何 dApp 都可以实现无限的计算规模——而不会损害以太坊的可组合性和安全性，这要归功于 Starknet 对最安全和最具可扩展性的加密证明系统——STARK 的依赖。

- 官方文档：https://docs.starknet.io/documentation/
- Starknet：JS库用于连接到Starknet 网络。https://www.starknetjs.com/docs/API/
- FullNode全节点实现: https://github.com/eqlabs/pathfinder
- Sequencer排序器（验证人节点）：目前验证人节点只有官方中心化部署，提供API网关
  - 主网网关：https://alpha-mainnet.starknet.io
  - 测试网网关：https://alpha4.starknet.io 、 https://alpha4-2.starknet.io
- DEV开发节点：https://github.com/0xSpaceShard/starknet-devnet
- API文档合集：https://www.postman.com/starknet-edu/workspace/starknet-edu/overview
- 区块链浏览器：
  - https://goerli.voyager.online/
  - https://testnet.starkscan.co/
- Starknet钱包
  - ArgentX：主流，开源 ，https://github.com/argentlabs/
  - Braavos：只开源账户合约，https://github.com/myBraavos/

## 1. Starknet账户结构

### 1.1 EIP-4337 (账户抽象)

​	在以太坊中，个人用户帐户被称为外部拥有帐户 (EOA)。EOA 与智能合约的不同之处在于它们不受代码控制。 EOA 由一对私钥和公钥确定。虽然简单，但 EOA 有一个主要缺点，即账户行为没有灵活性，以太坊协议规定了 EOA 发起的交易有效的含义（签名方案是固定的）。特别是，对公钥的控制可以完全控制帐户。虽然这在理论上是一种安全的帐户管理方法，但在实践中它有一些缺点，例如要求您保持助记词的安全但又可供您访问，以及钱包功能的灵活性有限。
​	EIP-4337 是以太坊的一项设计提案，概述了账户抽象，所有账户都通过以太坊网络上的专用智能合约进行管理，以此来提高灵活性和可用性。您可以在基本 EOA 功能之上添加自定义逻辑，从而将帐户抽象化引入以太坊。

两种账户的关键区别：

- EOA账户：

  - 创建不需要成本

  - 可以自行发起交易

  - EOA账户之间的交易只能是ETH/代币转账

- 合约账户：

  - 创建账户需要成本，因为你用到了网络存储

  - 只能在收到交易后发送交易

  - CA代码可以自定义执行各种不同的动作，如转移代币，甚至创建一个新的合约

<img src="https://picture-1258612855.cos.ap-shanghai.myqcloud.com/202304281806142.png" alt="img" style="zoom: 67%;" />

与以太坊一样，代币有一个 ERC20 合约来管理它。该合约包含一个表格，其中列出了每个相关账户拥有的代币数量：例如，账户地址 2 拥有该 ERC20 合约的 100 个代币。

用户感觉他们的代币存储在他们的钱包里，但这绝对是错误的。您的账户合约中没有存储资产列表。事实上，一个token有自己的ERC20合约，你的账户地址拥有的token数量就存储在这个合约中。

如果你想获得代币余额，请使用 `ERC20contract.balanceOf(accountAddress)` 函数询问其 ERC20 合约。

当你想转移你拥有的一些代币时，你必须通过 `account.execute` 功能使用 ERC20 合约功能 `transfer` 。这样，Starknet.js 就会向账户合约函数 `Execute` 发送一条用私钥签名的消息。

此消息包含要在 ERC20 合约中调用的函数的名称及其可选参数。

账户合约将使用公钥检查您是否拥有私钥，然后将要求 ERC20 合约执行请求的功能。

这样，ERC20 合约就绝对确定转账函数的调用者知道这个账户的私钥。

与以太坊相反，ETH 代币与所有其他代币一样，是 Starknet 中的 ERC20。在所有网络中，它的 ERC20 合约地址是：

```typescript
const addrETH = "0x049d36570d4e46f48e99674bd3fcc84644ddd6b96f7c741b1562b82f9e004dc7";
```



帐户抽象背后的想法是允许更灵活地管理帐户，而不是在协议级别确定它们的行为。这可以通过引入账户合约来实现——具有确定用户账户的可编程逻辑的智能合约。

使用帐户抽象，您现在可以对帐户的功能进行编程。 例如:

- 确定签名有效的含义或允许您的帐户与之交互的合同。这被称为“签名抽象”
- 用不同的代币支付交易费用——这有时被称为“费用抽象”
- 设计您自己的重放保护机制并允许并行发送多个非耦合事务。将此与以太坊中使用的顺序 nonce 解决方案进行比较，这导致交易本质上是顺序的，即，即使你想并行发送两个交易，你也必须在发送第二个之前等待第一个交易的确认。否则，第二笔交易可能会因随机数无效而被拒绝。通过帐户抽象，可以讨论绕过顺序随机数需求的不同解决方案。这被称为“nonce 抽象”

`注：现在目前Starknet只支持签名抽象`



### 1.2 EIP-2645（L2层私钥派生协议）

在 ZK-Rollups 等计算完整性证明 (CIP) Layer-2 解决方案的背景下，用户需要在针对这些环境优化的新椭圆曲线上签署消息。我们利用现有的密钥派生工作（BIP32、BIP39 和 BIP44）来定义一种有效的方法来安全地生成 CIP L2 私钥，并在第 2 层应用程序之间创建域分离。

Starkware 密钥是通过以下 BIP43 兼容的派生路径派生的，直接受到 BIP44 的启发：

```
m / purpose' / layer' / application' / eth_address_1' / eth_address_2' / index
```

- `m` - the seed.
  `m` - 种子。
- `purpose` - `2645` (the number of this EIP).
  `purpose` - `2645` （这个EIP的编号）。
- `layer` - the 31 lowest bits of sha256 on the layer name. Serve as a domain separator between different technologies. In the context of `starkex`, the value would be `579218131`.
  `layer` - 层名称上 sha256 的最低 31 位。作为不同技术之间的域分隔符。在 `starkex` 的上下文中，该值将是 `579218131` 。
- `application` - the 31 lowest bits of sha256 of the application name. Serve as a domain separator between different applications. In the context of DeversiFi in June 2020, it is the 31 lowest bits of sha256(starkexdvf) and the value would be `1393043894`.
  `application` - 应用程序名称的 sha256 的最低 31 位。作为不同应用程序之间的域分隔符。在 2020 年 6 月的 DeversiFi 上下文中，它是 sha256(starkexdvf) 的最低 31 位，值为 `1393043894` 。
- `eth_address_1 / eth_address_2` - the first and second 31 lowest bits of the corresponding eth_address.
  `eth_address_1 / eth_address_2` - 对应eth_address的第一个和第二个最低31位。
- `index` - to allow multiple keys per eth_address.
  `index` - 允许每个 eth_address 有多个密钥。

`注：现在市面上两种主流钱包，Braavos和ArgentX，其中Braavos闭源，ArgentX未使用EIP-2645，使用BIP32`



## 2. Starknet交易类型

### 2.1 Starknet 四种交易

- *declare*
  - 声明合约类，contract class
- deploy
  - 部署合约（调用通用UDC合约模板）
- *invoke*
  - 普通交易，调用合约
  - 需要待执行的合约hash，调用参数
- deploy_account
  - 创建账户合约
  - 需要账户合约类模板（根据功能的不同Argent、Braavos钱包均不相同）
  - 合约地址生成：线下预计算

```ts
// 椭圆曲线随机私钥
const privateKey = stark.randomAddress();
// 计算公钥
starkKeyPub = ec.starkCurve.getStarkKey(privateKey);

// 预计算合约地址
precalculatedAddress = calculateContractAddressFromHash(
  starkKeyPub,     // salt,
  accountClassHash, // 合约类hash
  [starkKeyPub],     // calldate,根据合约要求
  0                  // deployerAddress，为0
);
```



