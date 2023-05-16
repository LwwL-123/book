# SIP-005 (Blocks, Transactions, and Accounts)

该 SIP 描述了 Stacks 区块链中交易和区块的结构、验证和生命周期，并描述了每个节点如何维护处理区块链交易序列中编码的所有状态转换效果的物化视图。它介绍了 Stacks 区块链的账户模型，并描述了账户如何授权和支付处理网络上的交易。

## 1. Introduction

Stacks 区块链是一个复制的状态机。交易对 Stacks 区块链上的单个状态转换进行编码。 Stacks 区块链的状态通过具体化一系列交易的影响而演变——即，通过将每个交易的编码状态转换应用于区块链的状态。

Stacks 区块链中的交易编码各种状态转换，主要的是：

- 实例化智能合约（参见 SIP 002）
- 调用公共智能合约功能
- 在账户之间转移 STX 代币
- 惩罚分叉他们的微块流的领导者（见 SIP 001）
- 允许领导者执行有限的链上信号

处理交易不是免费的。验证和执行交易过程中的每个步骤都会产生非零计算成本。为了激励同行和领导者执行交易，交易的计算成本由账户支付。

账户是执行和/或支付交易的逻辑实体。交易的执行由三个账户管理，这三个账户可能不同，也可能不同：

- 发起账户是创建和发送交易的账户。这始终是用户拥有的帐户。每笔交易均由其原始帐户授权。
- 支付账户是领导者为验证和执行交易的成本而收取费用的账户。这也始终是用户拥有的帐户。如果未在交易中标识，则付款账户和原始账户是同一账户。
- 发送账户是标识当前正在执行交易的账户。发送账户可以在交易执行过程中通过 Clarity 函数 `as-contract` 更改，该函数将提供的代码块作为当前合约的账户执行。每笔交易的初始发送账户是其原始账户——即授权交易的账户。智能合约使用 `tx-sender` 内置函数确定发送账户的本金。

本文档将 Stacks 区块链中的账户作为处理交易的代理单位。事务执行的任务用于通知有关哪些数据进入事务以及进入块的数据的决策。因此，了解 Stacks 区块链中的区块和交易首先需要了解账户。

## 2. Specification

### 2.1 Accounts

Stacks 区块链中的交易源自账户，由账户支付，并在账户授权下执行。一个帐户由以下信息完整描述：

- **Address**: 这是唯一标识帐户的版本化加密哈希。帐户类型（如下所述）决定了对哪些信息进行哈希处理以得出地址。地址本身包含两个或三个字段：
  - A 1-byte **version**, 一个 1 字节的版本号，指示该地址是否对应于主网或测试网帐户以及使用哪种哈希算法来生成其哈希。
  - A 20-byte **public key hash**, 一个 20 字节的公钥哈希，使用地址版本和帐户拥有的公钥计算得出。
  - A variable-length **name**， 可变长度名称。仅在合约账户中使用，标识属于该账户的代码体。名称最长可达 128 个字节。属于用户的帐户没有此字段。
- **Nonce**：随机数。这是一个 Lamport 时钟，用于订购源自帐户的交易和从帐户支付的交易。 nonce 确保交易最多被处理一次。随机数计算帐户所有者授权交易的次数（见下文）。来自账户的第一笔交易的随机数值将为 0，第二笔交易的随机数值将为 1，依此类推。来自该账户所有者的有效交易授权必须包括该账户的下一个随机值；当交易被对等网络接受时，nonce 在该账户的物化视图中递增。

- **Assets**：资产。这是所有 Stacks 资产类型与帐户拥有的每种类型的数量之间的映射。这包括 STX 代币，以及 Clarity 智能合约声明的任何其他链上资产（即，可替代和不可替代的代币）。

据说所有可能地址的所有帐户都存在，但几乎所有帐户都是“空的”——它们的随机数值为 0，并且它们的资产映射不包含任何条目。一旦 Stacks 对等网络处理为其提供资金的交易，帐户的状态就会延迟具体化。也就是说，仅当交易的状态转换将条目插入到某个资产的某些（可能为零）数量的帐户资产映射中时，帐户状态才会具体化。即使该账户耗尽了所有资产持有量，它仍然是物化的。物化账户与空账户的区别在于前者都代表了领导者对其区块链状态的物化视图的承诺（如下所述）。

#### 2.1.1 Account Types

Stacks 区块链支持两种账户：

- **Standard accounts**： 标准账户。这些是由一个或多个私钥拥有的帐户。只有标准账户才能发起和支付交易。源自标准帐户的交易仅在其私钥阈值对其进行签名时才有效。标准帐户的地址是此阈值和所有允许的公钥的哈希值。由于需要向后兼容 Stacks v1，有四种方法可以对帐户的公钥和阈值进行哈希处理，它们与比特币的 pay-to-public-key-hash、multisig pay-to-script-hash、pay -to-witness-public-key-hash 和 multisig pay-to-witness-script-hash 哈希算法（见附录）。
- **Contract accounts**： 合约账户。这些是每当实例化智能合约时都会具体化的帐户。每份合约仅与一个合约账户配对。它不能授权或支付交易，但可以通过 Clarity 的 `as-contract` 功能充当当前执行交易的发送账户。合约地址的公钥哈希与创建它的标准账户的公钥哈希匹配，每个合约账户的地址都包含其代码体的名称。该名称在标准帐户实例化的代码主体集中是唯一的。

这两种账户都可能拥有链上资产。但是，合约账户的随机数必须始终为 0，因为它不能用于发起或支付交易。

#### 2.1.2 Account Assets

如 SIP 002 中所述，Stacks 区块链支持链上资产作为一流的数据类型——特别是支持可替代和不可替代资产。所有资产（STX 除外）都在特定合约范围内，因为它们是由合约创建的。在一个合约中，资产类型是唯一的。因此，所有资产类型都可以通过其在合约中的标识符及其完全限定的合约名称进行全局寻址。

无论在何处声明资产类型，资产的特定实例始终只属于一个账户。一旦合约声明了一种资产类型，该资产的实例就可以发送给其他账户并由其他账户拥有。

## 3. Transactions

交易是 Stacks 区块链中的基本执行单元。每笔交易都源自一个标准账户，并永久保留在 Stacks 区块链历史中。事务是原子的——它们要么完全执行其他事务，要么根本不执行。此外，交易由所有 Stacks 节点以相同的总顺序处理。

交易的核心是授权声明（见下文）、一段可执行的 Clarity 代码，以及交易被接受之前必须为真的后置条件列表。交易主体向 Stacks 区块链提供此代码，以及所有必要的元数据来描述交易应如何执行。不同类型的 Stacks 交易编码不同的元数据，因此具有不同的验证规则。

所有交易都源自一组拥有标准账户的私钥，即使它尚未具体化。这些私钥的所有者签署交易，附加交易费用，并将其转发到 Stacks 对等网络。如果事务格式正确，那么它将传播到所有可达的 Stacks 对等点。最终，假设交易在节点的记忆中保留了足够长的时间，Stacks 领导者将选择交易包含在最长的分叉的下一个区块中。一旦发生这种情况，由交易编码的状态转换就会在所有对等点的区块链状态副本中具体化。

###  3.1 Transaction Authorizations 交易授权

Stacks 区块链支持两种授权交易的方式：标准授权和赞助授权。区别在于原始帐户是否也是付款帐户。在具有标准授权的交易中，来源账户和支付账户是相同的。在具有赞助授权的交易中，原始账户和支付账户是不同的，并且两个账户都必须签署交易才能使其有效（首先是原始账户，然后是消费者）。

赞助授权的预期用例是使开发人员和/或基础设施运营商能够为用户付费以调用他们的智能合约，即使用户没有 STX 也可以这样做。赞助交易的签名流程是让用户首先用他们的原始账户签署交易，意图是赞助（即，用户必须明确允许赞助商签名），然后让赞助商用他们的支付签名账户支付用户的交易手续费。

### 3.2 Transaction Payloads 交易负载

Stacks 事务有效负载之间的主要区别在于它们可以从 Clarity VM 获得哪些功能（以及可以实现哪些副作用）。区分这些类型的交易的原因是为了使某些常见用例的静态分析成本更低，并为拥有该帐户的用户提供更高的安全性。

- Type-0: Transferring an Asse: 转移资产
  - 类型 0 交易只能将单一资产从一个账户转移到另一个账户。它可能不会直接执行 Clarity 代码。 type-0 交易只能发送 STX。它不能有后置条件（见下文）。
- Type-1: Instantiating a Smart Contract : 实例化智能合约
  - 类型 1 交易可以不受限制地访问 Clarity VM，并且在成功评估后，将实现一个新的智能合约帐户。 Type-1 交易旨在实例化智能合约，并调用多个智能合约函数和/或以原子方式访问它们的状态。
- Type-2: Calling an Existing Smart Contract : 调用现有的智能合约
  - 类型 2 事务限制了对 Clarity VM 的访问。类型 2 交易只能包含单个公共函数调用（通过 `contract-call?` ），并且只能提供 Clarity `Value` 作为其参数。这些交易不会具体化合约账户。
- Type-3: Punishing an Equivocating Stacks Leader : 惩罚违规的Stack领导人
  - 类型 3 交易编码两个格式正确、已签名但相互冲突的微块标头。也就是说，标头不同，但具有相同的序列号和/或父块哈希。如果在区块奖励到期之前开采，该交易将导致违规领导者失去其区块奖励，并导致该交易的发送者收到丢失的 coinbase 的一小部分作为发现不良行为的奖励。此事务无权访问 Clarity VM。
- Type-4: Coinbase
  - 类型 4 交易编码一个 32 字节的暂存空间供区块领导者自己使用，例如发出网络升级信号或宣布一组可用对等方的摘要。此交易必须是锚定块中的第一笔交易，以便该块被认为是格式良好的。此事务无权访问 Clarity VM。每个时期只能开采一个 coinbase 交易。

###  3.3 Transaction Encoding 交易编码

交易包括以下信息。多字节字段被编码为 big-endian。

- TransactionVersion: 一个 1 字节的版本号，标识交易是作为主网交易还是测试网交易。

- ChainID: 一个 4 字节的链 ID，标识该交易的目的地是哪个 Stacks 链。

- Authorization: 交易授权结构，如下所述，它对以下信息进行编码：

  -  原始帐户的地址。
  -  原始帐户的签名和签名阈值。
  -  赞助商帐户的地址，如果这是赞助交易。
  -  发起人帐户的签名和签名阈值（如果给定）。
  -  支付的费用，以 microSTX 计价。

- AnchorMode: 一个 1 字节的锚点模式，标识交易应该如何被挖掘。它采用以下值之一：

  - `0x01` ：交易必须包含在anchored块中
  - `0x02`: 交易必须包含在microblock中
  - `0x03` : leader 可以选择在哪里包含交易。

- Payload: 交易负载

- PostConditionMode: 一个 1 字节的后置条件模式，标识后置条件是否必须完全覆盖所有转移的资产。它可以采用以下值：

  - `0x01` ：本次交易可能会影响后置条件中未列出的其他资产。

  - `0x02` : 除了后置条件中列出的资产外，此交易可能不会影响其他资产。

- LengthPrefixedList: 以长度为前缀的后置条件列表，描述一旦交易完成执行，原始账户资产必须为真的属性。它的编码如下：

  - 4字节长度，表示后置条件的个数。
  - 零个或多个后置条件的列表，其编码如下所述。

```
export class StacksTransaction {
  version: TransactionVersion;
  chainId: ChainID;
  auth: Authorization;
  anchorMode: AnchorMode;
  payload: Payload;
  postConditionMode: PostConditionMode;
  postConditions: LengthPrefixedList;
}
```

#### 3.3.1 Version Number

```tsx
enum TransactionVersion {
  Mainnet = 0x00,
  Testnet = 0x80,
}
```

版本号标识交易是主网交易还是测试网交易。主网交易必须清除其最高位，测试网交易必须设置最高位（即， `version & 0x80` 对于测试网必须为非零，对于主网必须为零）。现在忽略低 7 位。

####  3.3.2 Chain ID

链 ID 标识此交易的目的地是 Stacks 区块链的哪个实例。由于主 Stacks 区块链和 Stacks 应用链（在未来的 SIP 中描述）共享相同的交易电汇格式，因此该字段用于区分每个链的交易。 Stacks 主区块链的交易必须有一个 `0x00000000` 的链 ID。

```tsx
enum ChainID {
  Testnet = 0x80000000,
  Mainnet = 0x00000001,
}
```

#### 3.3.3 Transaction Authorization

每笔交易都包含一个交易授权结构，Stacks 节点使用该结构来识别发起账户和赞助账户，确定支出账户将支付的费用，并确定是否允许进行编码状态转换。它的编码如下：

- 1-byte **authorization type** ,一个 1 字节的授权类型字段，指示交易是否具有标准授权或赞助授权。

  - 对于标准授权，这个值必须是 `0x04` 。
  - 对于赞助授权，这个值必须是 `0x05` 。

- 一两个支出条件，其编码如下所述。如果交易的授权类型字节表示是标准授权，则有一个消费条件。如果是赞助授权，则有两个支出条件。

  *Spending conditions* are encoded as follows, 支出条件编码如下：

  - **hash mode**： 一个 1 字节的哈希模式字段，指示应如何使用原始帐户授权的公钥和签名来计算帐户地址。支持四种模式，用于模拟 Stacks v1（使用比特币哈希例程）中支持的四种哈希模式：

    ```tsx
    enum AddressHashMode {
      // serialization modes for public keys to addresses.
      // We support four different modes due to legacy compatibility with Stacks v1 addresses:
      /** SingleSigHashMode - hash160(public-key), same as bitcoin's p2pkh */
      // 使用单个公钥。像比特币 P2PKH 输出一样散列它。
      SerializeP2PKH = 0x00,
      /** MultiSigHashMode - hash160(multisig-redeem-script), same as bitcoin's multisig p2sh */
      // 使用了一个或多个公钥。将它们散列为比特币多重签名 P2SH 兑换脚本
      SerializeP2SH = 0x01,
      /** SingleSigHashMode - hash160(segwit-program-00(p2pkh)), same as bitcoin's p2sh-p2wpkh */
      // 使用单个公钥。像比特币 P2WPKH-P2SH 输出一样散列它
      SerializeP2WPKH = 0x02,
      // 使用了一个或多个公钥。将它们散列为比特币 P2WSH-P2SH 输出
      /** MultiSigHashMode - hash160(segwit-program-00(public-keys)), same as bitcoin's p2sh-p2wsh */
      SerializeP2WSH = 0x03,
    }
    ```

  - A 20-byte **public key hash**, 一个 20 字节的公钥哈希，它是根据散列模式标识的散列例程从公钥派生的。hash mode和public key hash唯一标识原始账户，hash mode用于推导出相应的账户版本号。

  - An 8-byte **nonce**. 一个 8 字节的随机数。

  - An 8-byte **fee**. 一个 8 字节的费用。

  - Either a **single-signature spending condition** or a **multisig spending condition**, 单一签名支出条件或多重签名支出条件，如下所述。如果哈希模式字节是 `0x00` 或 `0x02` ，则遵循单一签名支出条件。否则，将遵循多重签名支出条件。

    - A *single-signature spending condition* is encoded as follows: 单签名消费条件编码如下：
      - A 1-byte **public key encoding**，一个 1 字节的公钥编码字段，用于指示在散列之前是否应压缩公钥。这将是：
        - `0x00` 压缩
        - `0x01` 未压缩
      - A 65-byte **recoverable ECDSA signature**，一个 65 字节的可恢复 ECDSA 签名，其中包含 secp256k1 签名的签名和元数据。
    - A *multisig spending condition* is encoded as follows: 多重签名支出条件编码如下：
      - A 2-byte **signature count** ， 一个 2 字节的签名计数，指示授权有效所需的签名数。
      - 支出授权字段的长度前缀数组，如下所述。
        - A 1-byte **field ID**, 1 字节的字段 ID，可以是 `0x00` 、 `0x01` 、 `0x02` 或 `0x03` 。
        - The **spending field body**, 
        - 支出字段正文，具体如下，具体取决于字段 ID：
          -  `0x00` 或 `0x01` ：接下来的 33 个字节是压缩的 secp256k1 公钥。如果字段 ID 是 `0x00` ，则密钥将作为压缩的 secp256k1 公钥加载。如果是 `0x01` ，则密钥将作为未压缩的 secp256k1 公钥加载。
          -  `0x02` 或 `0x03` ：接下来的 65 个字节是可恢复的 secp256k1 ECDSA 签名。如果字段 ID 为 `0x02` ，则恢复的公钥将作为压缩公钥加载。如果是 `0x03` ，那么恢复的公钥将作为未压缩的公钥加载。

支出条件结构中所需签名的数量和公钥列表唯一标识一个标准帐户。并可用于根据以下规则生成其地址： 

| Hash mode | Spending Condition | Mainnet version | Hash algorithm             |
| --------- | ------------------ | --------------- | -------------------------- |
| `0x00`    | Single-signature   | 22              | Bitcoin P2PKH              |
| `0x01`    | Multi-signature    | 20              | Bitcoin redeem script P2SH |
| `0x02`    | Single-signature   | 20              | Bitcoin P2WPK-P2SH         |
| `0x03`    | Multi-signature    | 20              | Bitcoin P2WSH-P2SH 比特币  |



下面简要介绍哈希算法，以及今天在比特币中使用的镜像哈希算法。这对于向后兼容 Stacks v1 账户是必要的，Stacks v1 账户依赖比特币的脚本语言进行授权。

- Hash160：取其输入的SHA256哈希，然后取32字节的RIPEMD160哈希
- 比特币P2PKH：该算法从单签名消费条件中取出ECDSA可恢复签名和公钥编码字节，将它们转换为公钥，然后计算密钥字节表示的Hash160（即通过将密钥序列化为压缩包）或未压缩的 secp256k1 公钥）。

- *Bitcoin redeem script P2SH*, 比特币赎回脚本 P2SH：该算法将多重签名支出条件的公钥和可恢复签名转换为比特币 BIP16 P2SH 赎回脚本，并根据赎回脚本的字节计算 Hash160（如 BIP16 中所做的那样）。它将给定的 ECDSA 可恢复签名和公钥编码字节值转换为它们各自的（未）压缩的 secp256k1 公钥以执行此操作。
- *Bitcoin P2WPKH-P2SH*， 比特币P2WPKH-P2SH：该算法从单签名消费条件中取出ECDSA可恢复签名和公钥编码字节，转化为公钥，生成P2WPKH见证程序，P2SH赎回脚本，最后是赎回的Hash160用于获取地址的公钥哈希的脚本。
- *Bitcoin P2WSH-P2SH*， 比特币 P2WSH-P2SH：该算法采用 ECDSA 可恢复签名和公钥编码字节，以及任何给定的公钥，并将它们转换为多重签名 P2WSH 见证程序。然后它从见证程序生成一个P2SH赎回脚本，并从赎回脚本的Hash160中获取地址的公钥哈希。

#### 3.3.4 Transaction Post-Conditions

后置条件列表编码如下：

- A 4-byte length prefix: 一个 4 字节长度的前缀

- Zero or more post-conditions, 零个或多个后置条件。后置条件可以采用以下形式之一：

  - A 1-byte **post-condition type ID** 一个 1 字节的后置条件类型 ID

  ```tsx
  enum PostConditionType {
    // STX 后置条件，与原始账户的 STX 有关
    STX = 0x00,
    // 可替代令牌后置条件，属于原始帐户的可替代令牌之一。
    Fungible = 0x01,
    // 不可替代令牌后置条件，与原始帐户的不可替代令牌之一有关。
    NonFungible = 0x02,
  }
  ```

  - A variable-length **post-condition** 可变长度的后置条件
    - STX 后置条件体编码如下：
      - A variable-length **principal**, containing the address of the standard account or contract account
        可变长度主体，包含标准账户或合约账户的地址
      - A 1-byte **fungible condition code**, described below
        一个 1 字节的可替代条件代码，如下所述
      - An 8-byte value encoding the literal number of microSTX
        编码 microSTX 字面量的 8 字节值
    - Fungible token 后置条件体编码如下：
      - A variable-length **principal**, containing the address of the standard account or contract account
        可变长度主体，包含标准账户或合约账户的地址
      - A variable-length **asset info** structure that identifies the token type, described below
        标识令牌类型的可变长度资产信息结构，如下所述
      - A 1-byte **fungible condition code** 一个 1 字节的可替代条件代码
      - An 8-byte value encoding the literal number of token units
        一个 8 字节的值，编码令牌单元的字面数量
    - Non-fungible token 后置条件体编码如下：
      - A variable-length **principal**, containing the address of the standard account or contract account
        可变长度主体，包含标准账户或合约账户的地址
      - A variable-length **asset info** structure that identifies the token type
        标识令牌类型的可变长度资产信息结构
      - A variable-length **asset name**, which is the Clarity value that names the token instance, serialized according to the Clarity value serialization format.
        可变长度的资产名称，即命名通证实例的 Clarity 值，根据 Clarity 值序列化格式进行序列化。
      - A 1-byte **non-fungible condition code**
        一个 1 字节的不可替代条件代码

主体结构编码标准帐户地址或合约帐户地址。

- 一个标准的账户地址被编码为一个 1 字节的版本号和一个 20 字节的 Hash160。
- 合约账户地址编码为1字节的版本号、20字节的Hash160、1字节的名称长度和最多128个字符的可变长度名称。名称字符必须是有效的合同名称（见下文）。

主体后置条件字段的标准主体变体编码如下：

- A 1-byte type prefix of `0x02`
  `0x02` 的 1 字节类型前缀
- The standard principal's 1-byte version
  标准主体的 1 字节版本
- The standard principal's 20-byte Hash160
  标准主体的20字节Hash160


主体后置条件字段的合同主体变体编码如下：

- A 1-byte type prefix of `0x03`
  `0x03` 的 1 字节类型前缀
- The 1-byte version of the standard principal that issued the contract
  发布合约的标准主体的 1 字节版本
- The 20-byte Hash160 of the standard principal that issued the contract
  发行合约的标准委托人的20字节Hash160
- A 1-byte length of the contract name, up to 128
  1字节长度的合约名称，最多128个
- The contract name 合约名称

#### 3.3.5 Transaction Payloads 交易负载

有五种不同类型的交易有效载荷。每个有效载荷编码如下：

-  A 1-byte **payload type ID**, 一个 1 字节的负载类型 ID，介于 0 和 5 之间。
-  A variable-length **payload**, 一种可变长度的有效载荷，其中有五种。

```tsx
enum PayloadType {
  TokenTransfer = 0x00,
  SmartContract = 0x01,
  ContractCall = 0x02,
  PoisonMicroblock = 0x03,
  Coinbase = 0x04,
  CoinbaseToAltRecipient = 0x05,
  VersionedSmartContract = 0x06,
}
```



## 4. Blocks and microblocks

Stacks 区块链允许使用称为微块的机制增加交易吞吐量。比特币和 Stacks 步调一致，它们的区块同时得到确认。在 Stacks 上，这被称为“锚块”。 Stacks 交易的整个区块对应于单个比特币交易。这显着提高了处理 Stacks 事务的成本/字节比。由于同步块生产，比特币充当创建 Stacks 块的速率限制器，从而防止对其对等网络的拒绝服务攻击。

然而，在比特币区块链上的 Stacks 锚块之间，也有不同数量的微块，允许以高度自信的方式快速结算 Stacks 交易。这允许 Stacks 交易吞吐量独立于比特币扩展，同时仍然定期与比特币链建立最终确定性。 Stacks 区块链采用块流模型，每个领导者可以在交易到达内存池时自适应地选择并将交易打包到他们的块中。因此，当一个锚块被确认时，父微块流中的所有交易都被打包和处理。这是一种前所未有的实现可扩展性的方法，无需创建与比特币完全独立的协议。

![stx-microblock](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/202305081549818.png)

