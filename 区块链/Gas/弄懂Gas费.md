# 弄懂Gas费



## 1. gas是什么？

gas是指在以太坊上执行操作所需的“燃料”。

以太坊提供了虚拟机(EVM，Ethereum Virtual Machine)，开发者可以在其上开发各种应用。EVM相对BTC的好处是“图灵完备”，但这带来一个潜在风险，就是一个程序可能无休止的运行下去，对EVM而言，这是不能容忍的。

所以，运行程序要花gas。就好比开车要花油费或电费，油或电用完了，车自然会停下来。



## 2. gas、gas price、gas fee是一个东西吗？

你使用EVM执行一个交易，需要若干个gas，这被称为gas数。类似于油的升数、电的度数。

而每一个gas都是需要花钱的，gas有价格，称为gasprice。这类似于一升汽油的价格、一度电的价格。

gas费就是gas数乘以gas价格。

```
比如，你想部署一个合约，需要3,000,000个gas，gasprice是200gwei。那么你要花的钱是：
3* 10^6 * 200 gwei = 3* 10^6 * 200 * 10^(-9) = 0.6 ETH
```

gasprice的计量单位为：gwei，一个gwei为是1g个wei，即10^9 wei。

由于1 wei = 10^(-18) ETH，所以: 1 gwei = 10^9 wei = 10^(-9) ETH。

```
wei是ETH的最小单位，以b-money的创造者Wei Dai的名字命名。
```



## 3. gasprice由谁来定？

gas的价格不像我们所想象的，由政府统一定价，No！也不是由矿工定价。

gas的价格由交易的发送者来指定，在伦敦升级之前，发送者在交易中要指定两个值，一个是`gaslimit`，一个是`gasprice`。



## 4. 为什么以太坊的gasprice这么贵？

因为以太坊流行，很多人都想在上面执行交易，谁出的价格高，谁的交易就更可能被矿工执行并打入（include）区块，矿工显然喜欢更高的gas费。

所以，这更像是一个拍卖，想交易的人给出各种gasprice，矿工优先选择那些出价高的上链。



## 5. gaslimit是什么？

越复杂的运算，需要消耗的gas越多。交易发送者有时候也搞不清自己需要花多少gas费来执行操作，所以需要加上一个消耗gas的上限，避免自己的钱一不小心被花光了。（如果没有这个机制就可能发生这事）

发送者设置一个gaslimit，如果没有花到这个数，会打回剩余的值。

如果gaslimit耗尽还未执行完交易，EVM会抛出异常，结束代码执行，回退发生的变更。不过，但由于矿工们已经干了活，花费了成本，所以，已经花掉的gas是不退的。

所以，gaslimit要宁可高一点，也不要太低，因为高了没关系，没花完的会退回来，低了，一旦out of gas，不仅你想要的操作没有完成，而且消耗的gas也不会退你，可谓鸡飞蛋打一场空。

一个示例：

```
张三向李四转移1 ETH（也即ether的transfer操作）。
 
张三将gaslimit设为3万，gasprice设为200 gwei。
 
以太坊规定，transfer操作花费21000个gas，所以实际发生总费用是：
 
21,000 * 200 = 4,200,000 gwei，即0.0042 ETH。
 
这样，张三发送1.006 ETH，李四获得1 ETH，矿工获得0.0042 ETH，然后退还张三0.0018 ETH。
 
张三虽然将gaslimit设为3万，但实际只花了2.1万个gas，实际支出1.0042 ETH。
 
但如果张三将gaslimit设置为1万，这个操作就无法完成，而且10000个gas也没了，白白损失10000*200=200万gwei=0.002 ETH
```



## 6. gas数到底是怎么算的？

具体的计算有点复杂，但有标准可以查。你可以在GitHub上的evm-opcodes1和DynamicgasCosts2查看。

在EVM里面，每个运算、操作、存储都需要gas的，比如：

- ADD：加法操作 3gas
- MUL：乘法操作 5gas
- SUB：减法操作 3gas
- DIV：除法操作 5gas
- JUMP：跳转操作 8gas
- MSTORE：内容存储操作 3gas
- MLOAD：内容读取操作 3gas
- CREATE：创建合约 32000gas (if tx.to == null)
- SSTORE：存入存储区 20000gas （从0设为非0值）
- SHA3：Keccak256哈希 30gas + 6gas * (size of input in words) + mem_expansion_cost

交易基本费用：21000gas （比如Transfer就要这么多）



## 7. 具体交易时，我该如何指定这两个值？

一般而言，钱包或者开发工具，会帮你做好这两件事。(不需要劳动您亲自去设)

比如，在小狐狸中，供用户选择的有三个选项：高、中、低。

你不需要亲自指定gaslimit和gasprice，你顶多只需要选择高、中、低就好了。

大致的意思就是，如果你想让你的交易快点执行，就选择高费用，如果不着急，就选择低费用。

![7313611014843ca16fcbcabf1c80bc2d.png](https://picture-1258612855.cos.ap-shanghai.myqcloud.com/202307031410276.jpeg)

这里面的“燃料限制”就是刚才讲过的gaslimit，但是并没有出现刚才说的gasprice，而是出现了最高优先费用（max priority fee）和最高费用（max fee），这是因为在伦敦升级后，gas费有了新的规则，下面会介绍。

## 8. 我听说，不仅仅是交易有gaslimit，区块也有gaslimit？

是的，每个block有gas的上限（即区块内所有交易的gas总数），达到这个上限，就必须出块了，不能再往区块里打入（include）新的交易了。

而这个上限，是矿工投票投出来的。

现在（伦敦升级后）上限是30M（3千万个gas），注意不是30M的存储空间，是30M个gas。

一个区块中包含的gas总数可以称为区块的大小。

注：本文首次发布日期为2022年2月7日

## 9. 伦敦升级到底带来了什么变化？

伦敦升级后（2021年8月5日以太坊完成伦敦升级），区块的大小是弹性的，目标大小是15M，上限是30M。

伦敦升级后，gas费总额计算仍然是：gaslimit * gasprice。

但gasprice分为两部分：

gasprice =baseFee+ Tip

其中basefee（基本费用）由协议根据区块大小自动计算，但这部分会被烧掉，矿工也拿不到。

矿工只能拿到Tip。Tip就是小费，或者书面一点，称之为优先费用（PriorityFee）。

当区块比15M大的时候，gas的价格就会上升（通过自动调节basefee），目的是让更多的交易望而却步，这样区块大小就会缩回到15M；反之，区块小于15M时，gasprice就会便宜，交易就会增多，这样，区块大小就总是在目标大小左右浮动。

伦敦升级（尤其是燃烧basefee）带来很多好处，这里不一一列举，有兴趣可以看看这两篇文章：Analysis of EIP-15593、All you need to know about EIP-15594。

下面是伦敦升级后gas费计算的一个示例：

```
张三给李四转1个ETH。
 
gaslimit为21,000，basefee为100 gwei，Tip为10 gwei，可以计算得出gas费为21,000 * (100 + 10) = 2,310,000 gwei，即0.00231 ETH。
 
当张三汇款时，1.00231 ETH将从张三的账户中扣除。李四将得到1个ETH。基本费用为0.0021 ETH会被烧掉，矿工获得0.00021 ETH的Tip。
```

## 10. 这些值如何设置？

当然这些也都由钱包或开发工具帮用户设置了。

basefee不用设置（钱包也不管），是以太坊通过算法，自动根据上一个block的大小和上一个basefee计算出来的。

为了让用户更好地控制自己的钱，实际交易中，设置的是maxfee（愿意给出的最大gasprice）和maxpriorityfee(愿意给出的最大Tip)。

maxfee必须要大于basefee，然后，矿工会按如下的算法计算Tip：

Tip = min(maxpriorityfee, maxfee - basefee)
如果maxfee > basefee + Tip，多余的费用就会被矿工退回交易发送者。

比如：

```
张三给李四转1个ETH。
 
gaslimit为21,000，maxfee为150 gwei，maxpriorityfee设为10 gwei。
 
basefee为100 gwei，所以，Tip= min（ 10，150-100） = 10 gwei，可以计算得出gas费为21,000 * (100 + 10) = 2,310,000 gwei，即0.00231 ETH。
 
当张三汇款时，21000 * 150 = 1.00315 ETH将从张三的账户中扣除。李四将得到1个ETH。基本费用为0.0021 ETH会被烧掉，矿工获得0.00021 ETH的Tip，矿工退回张三0.00084 ETH（即21000 * 40 gwei）。
```

> 注意一点：上面说的maxfe、basefee、Tip，都是针对per gas的，计算真正费用的时候，还需要乘上gas数。
>
> 你在不同场合看到的这些符号，有些是带pergas的，有些不带，注意要区分一下，一般指的都是pergas的。
>
> 还有一点，如果将maxfee和和maxpriorityfee设置为相同的值，maxfee就相当于以前的gasprice了。

## 11. basefee究竟是怎么自动计算出来的？

basefee是由以太坊的协议自动计算的，该算法将上一个区块的大小与目标大小(15M)进行比较。如果超过目标块大小，下一个块的基本费用将按照比目标块多出的比例增加，最多将增加12.5%。

比如一个块是30M，达到了最高限，它的大小比目标大小多出一倍，它的basefee就是上一个basefee的112.5%。

如果区块一直保持在30M的大小，每个basefee都比上一个要高出12.5%，这就产生了指数级的增长，basefee高到人们舍不得交易的时候，区块大小自然就会回落。

在EIP-15595中，描述了这个算法的具体细节(看着繁，其实不难懂)：

```go
Note: // is integer division, round down.
BASE_FEE_MAX_CHANGE_DENOMINATOR = 8
if parent_gas_used == parent_gas_target:
            expected_base_fee_per_gas = parent_base_fee_per_gas
elif parent_gas_used > parent_gas_target:
            gas_used_delta = parent_gas_used - parent_gas_target
            base_fee_per_gas_delta = max(parent_base_fee_per_gas * gas_used_delta // parent_gas_target // BASE_FEE_MAX_CHANGE_DENOMINATOR, 1)
            expected_base_fee_per_gas = parent_base_fee_per_gas + base_fee_per_gas_delta
else:
            gas_used_delta = parent_gas_target - parent_gas_used
            base_fee_per_gas_delta = parent_base_fee_per_gas * gas_used_delta // parent_gas_target // BASE_FEE_MAX_CHANGE_DENOMINATOR
            expected_base_fee_per_gas = parent_base_fee_per_gas - base_fee_per_gas_delta
```

### 12. 交易报文中会体现这些值吗？

会的，下面是一个交易的示例6：

```
{
  from: "0xEA674fdDe714fd979de3EdF0F56AA9716B898ec8",
  to: "0xac03bb73b6a9e108530aff4df5077c2b3d481e5a",
  gaslimit: "21000",
  maxFeePergas: "300",
  maxPriorityFeePergas: "10",
  nonce: "0",
  value: "10000000000"
}
```

看到里面和gas相关的东西了吧。

在传统交易报文中，用户直接指定gasprice；而在如上的EIP-1559交易报文中，用户指定的是MaxFeePerGas和MaxPriorityFeePerGas。

实际上你为每gas支付的单价是：

```
min( MaxPriorityFeePerGas + basefee, MaxFeePerGas )
```





与项目方沟通，starknet必须要正确的、可以上链的签名数据才能去预估gas费，否则预估gas费接口会报错（包括错误nonce，超出余额的转账金额等）

现在移动端的逻辑是用户在签名交易的时输入密码后，才能获取私钥，在此之前无法预估gas费。调研竞品bitkeep后，发现bitkeep仅支持转账交易，不支持dapp。与@李超群讨论后，现有几个方案：

1. 转账交易移动端与cefi方案保持一致，采用固定私钥生成的固定交易串去预估gas（因为都为transfer方法，gas费一样），dapp只由插件端接入。
2. 移动端修改对应逻辑，新增页面适配，在预估gas时就要求用户输入密码。（移动端工作量很大，改动多）
3. 新增接口去获取发起合约调用时，前几笔链上的相同类型交易gas的平均值。（gas估计不准确，据项目方描述，starknet支持不同类型的签名，gas费差距较大，可能会上链失败）
4. 用我们自己的私钥组一笔相同的交易，去预估gas（可能会预估失败，例如用户的转账金额超出了我们地址的余额）
