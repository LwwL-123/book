# Substrate的Staking模块分析



# 1. 概述

​		质押经济本质上来说也是一种挖矿，但和我们通常所说的比特币挖矿，以太坊挖矿不同。

- 比特币，莱特币，以太坊，BCH等这些数字货币都是基于工作量证明Proof-of-Work（POW）的数字货币，产生新的货币都是比拼算力。
  - Staking(质押)则是另外一种挖矿方式。通常基于**权益证明Proof-of-Stake（POS）**和提名权益证明Nominated Proof-of-Stake（NPOS）。
  - 在这种挖矿方式中，区块链系统中的节点不需要太高的算力，而只需要质押一定数量的代币，运行一段时间后就可以产生新的货币，而产生的新货币就是通过质押得到的收益。
  - **这就相当于我们把钱存在银行，每年能够得到一定的利息一样。**



![image-20210615112721198](https://gitee.com/lzw657434763/pictures/raw/master/Blog/20210624151959.png)



假设在第一年中，Alice和Bob在网络上都有1%的网络代币。Alice选择通过staking的方式维护网络安全，参与网络建设，而Bob没有。

到第5年，我们看到Alice的代币（绿色实线）随着网络总体代币量（黑色实线）的增长而提升，而Bob（浅绿虚线）还停留在原地。实际上，Bob占有的网络份额在下降，Alice的份额获得提升，部分所有权从Bob转移给了Alice。



- Staking 是管理网络维护者的抵押资金的模块。
- 网络维护者也称为Authorities (出块人)或者Validators (验证人)
- 它们是基于抵押资金被选中，它们在正常履行职责的情况下会获得奖励，如果行为不当则会受到惩罚(没收一定的资金)。



# 2. NPoS流程

Staking Module 有两个重要的角色：验证人和提名人：

- ​	`验证人（Validator）`:一个验证者它的角色是验证区块和保证区块最终一致性，维护网络的诚实。验证者应该避免任何恶意的错误行为和离线。声明有兴趣成为验证者的绑定账号不会立即被选择为验证者，相反，他们被宣布为候选人，他们可能在下届选举中被选为验证者。选举结果由提名者及其投票决定。
- `提名人（Nominator）`:一个提名者在维护网络时不承担任何直接的角色，而是对要选举的一组验证人进行投票。账号一旦声明了提名的利益，将在下一轮选举中生效。提名人stash账号中的资产表明其投票的重要性，验证者获得的奖励和惩罚都由验证者和提名者共享。这条规则鼓励提名者尽可能不要投票给行为不端/离线的验证者，因为如果提名者投错了票，他们也会失去资产。



## 2.1 账户

​	为了保障用户资金安全，Staking Module设计了两层结构的独立密钥类型，采用两个不同的账户来管理资金， 我们称为：存储账户（Stash Account）和控制账户（Controller Account）（就好比房东和中介的关系）

- `Stash` ：存储账户主要用来存放用于质押的资金，存储账户可以指定一个控制账户，将申请提名人、验证人等功能委托给控制账户，存储账户的密钥可以长期保存在冷钱包中，以此保证用户资金的安全性。

- `Controller`：控制帐户是存储账户的代理，有申请提名人和验证人、设置收款账户和佣金的权利。如果是验证人，它还可以设置 session keys 会话密钥。只需要保证控制账户有足够的资金来支付交易手续费。

  <img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20210623103043.png" alt="image-20210623103033966" style="zoom:33%;" />

## 2.2 质押资金（fn bond）

​	用户需要质押一定的资金（质押金额不得小于限定的最小金额）来获取成为验证人或提名人的资格，质押行为由存储账户发起，质押过程可以设置控制账户、质押金额、收款账户。

## 2.3 设置验证人（fn validate）

​	设置验证人，包括设置验证人的佣金，佣金是按比例收取的，当分配staking奖励时，会优先支付验证人的佣金，剩余奖励才会分配给提名人。

​	注意：同一个stash account 只能成为验证人或提名人，验证人可以通过自抵押的方式提名自己，但不可以通过提名的方式。已经是提名人的stash account 不可以作为验证人。

## 2.4 提名验证人（fn nominate）

​	控制账户可以提交一份支持的信誉良好的候选验证人名单（提名最多只能有16个,即`MAX_NOMINATIONS` ）成为提名人。在下一个Era，具有最多 节点 支持的一定数量的验证人被选中，如果提名人支持的验证人被选中，就可以分享验证人出块奖励或惩罚。提名过程只能发生在非候选验证人选举阶段。

​	一旦提名阶段结束，NPoS选举机制以提名人及其投票为输入，输出一组数量有限的验证人，使任何验证人的支持度最大化，并尽可能均匀分布。这种选举机制的目标是最大限度地提高网络的安全性，实现提名人的公平代表。

## 2.5 冻结验证人或提名人（fn chill）

​	冻结是从活跃验证人节点池中移除验证人的行为，这意味着，如果他们是提名者，他们将不再被视为选民，如果他们是验证者，他们将不再是下次选举的候选人。



## 2.6 slot/session/era

<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20210623103233.png" alt="image-20210623103226657" style="zoom:50%;" />

- Slot（插槽）：每一个Slot都会产生一个新的区块——基于Babe共识机制

- Epoch（时期）：固定长度的Slot的集合，每一个epoch会更新验证人集合

- Era（时代）：多个epoch的集合，一个era结束后，结算奖励

# 3. NPoS机制

NPoS（Nominated Proof of Stake，提名权益证明）是Polkadot基于PoS算法设计的共识算法，验证人（ Validator）运行节点参与生产和确认区块，提名人（Nominator）可以抵押自己的dot获得提名权，并提名自己信任的验证人，获得奖励。

## 3.1 为什么会产生NPoS

​	所谓共识，就是区块链中，各个节点维护系统的稳定运行所达成的一致性。如果节点越多，那么共识越强大，整个区块链系统就会越安全。

​	 在波卡的设计思想中，中继链就是设计成了维护平行链的共识。 也就是说，加入波卡的平行链，不需要记账了！由波卡的中继链统一给你记账，统一维护你的区块链的安全！ 

<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20210623102731.png" alt="image-20210623102719318" style="zoom:50%;" />

在传统的PoS中：

- 如果总量恒定，那么越来越多人质押币的话，币会越来越少。
- 如果总量恒定，挖出来的币又会被拿去质押，加速币的稀少。

 那到后面没币可挖、没币可质押，还怎么去激励那些节点。 所以就必须有通胀（每年增发）了，靠每年增发出来的币来作为节点的激励。这就是权益证明POS，它的成本来自它的通胀。

 在波卡的设计思想中，中继链就是设计成了维护平行链的共识。 也就是说，加入波卡的平行链，不需要记账了。由波卡的中继链统一给你记账，统一维护你的区块链的安全！

<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20210623102757.png" alt="image-20210623102751147" style="zoom:50%;" />

​	可以看出，波卡给人最大的安全感就是因为中继链维护共识的能力，那么中继链是如何维护共识的呢，如果中继链的节点不够分散，被一些节点垄断了，即使节点再多也不会有安全感。

 **如何让节点足够分散并且足够去中心化，这是波卡共享安全最核心的要素。**

 波卡的共识机制是提名权益证明（NPOS），在POS的基础上改良，完美解决了节点垄断的问题，使得网络足够去中心化，杜绝节点窜通作恶现象。

1、单个节点的收益
 2、验证人和提名人的收益


 3、市场调节

对于验证人来说，佣金是可以自由设定的，有的节点佣金高，有的节点佣金低，市场自由竞争。这就造成下面的情况： 

佣金如果很少，提名人就不会投你，切换投票到分成更多的验证人节点，就有出局的风险。 

市场会自己调节，验证人佣金会逐渐回归到一个合适的范围，比方说5%-10%。 那么问题来了，如果你是提名人，在大多数验证人节点佣金差不多的情况下，你会投给哪个节点呢？ 聪明的提名人一定会投给质押DOT总数低的节点。

为什么？因为由于平均分配，单个节点日收益都是一样的，每个验证人节点的佣金又都差不多，投给质押DOT总数低的节点，你质押的DOT占比就会更大，在所有提名人的分配中占据优势。

划重点：正因为提名人会更愿意投票给质押总数低的节点，才会创建有平等质押量的验证人节点池，足够去中心化

## 3.2 Phragmén选举算法

​	验证人选举算法是NPoS机制的核心，选举过程要具有公平代表性和安全性，Polkadot为此设计了`Phragmén`算法,确保每次选举都具有这种性质。

​	公平代表性：任何持有总股份至少 1/n 的提名人都保证至少有一个他们信任的验证人当选

​	安全性：我们希望尽可能让对候选验证人很难获得一个验证人，他们只有得到足够高的支持才能做到这一点。因此，我们将选举结果的安全级别等同于被选验证人的最小支持数量。

<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20210623102251.png" alt="image-20210615134833761" style="zoom:50%;" />

# 4. 奖励和惩罚

​	奖励和惩罚是抵押模块的核心，试图包含有效的行为，同时惩罚任何不当行为或缺乏可用性的行为。一旦有不当行为被报告，惩罚可以发生在任何时间点。一旦确定了惩罚，一个惩罚的值将从验证者的余额中扣除，同时也将从所有投票给该验证者的提名者的余额中扣除。与惩罚类似，奖励也在验证者和提名者之间共享。然而，奖励资金并不总是转移到stash账号。

- Staked：奖励支付给存储账户并用来质押
- Stash：奖励支付给存储账户，但奖励不用来质押
- Contriller：奖励支付给控制账户
- Account：奖励支付给一个指定账户



<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20210623102945.png" alt="image-20210623102939847" style="zoom:50%;" />

## 4.1 奖励结算

​	每个区块生成后，`authorship->on_initialize`会记录区块生产者的`ErasRewardPoints`, 并在每个`end_era` 进行结算。

​	奖励点数增加规则：

- 主链区块生产者增加20点
- 叔区块生产者增加2点
- 引用叔区块的生产者增加1点



## 4.2 奖励分配（fn do_payout_stakers）

<img src="https://gitee.com/lzw657434763/pictures/raw/master/Blog/20210623103004.png" alt="image-20210623102959766" style="zoom:50%;" />

两个验证人在相同的工作中获得相同数量的 DOT，即它们的报酬与每个验证人的 stake 质押数量不成比例

奖励的一部分（commission 按奖励百分比设置）优先用于支付验证人的佣金，其余部分按比例（即与 stake 成比例）支付给提名人和验证人

验证人将获得两次奖励：一次作为验证的佣金，一次作为用自抵押提名自己的佣金



## 4.3 惩罚（Slash）

​	如果验证人在网络中行为不当（例如脱机、攻击网络或运行修改过的软件），则会发生 slash 惩罚。他们和他们的提名人会因为 slash 惩罚而失去一部分 DOT。总质押数较大的验证池将受到更严厉的 slash 惩罚。在`start_era`时会对slash进行清算处理。

​	在pallet-offences模块中定义了三种违规行为：

- UnresponsivenessOffence (无响应)

- GrandpaEquivocationOffence （重复签名,voting）

- BabeEquivocationOffence （重复出块）

如果验证人因任何一项违规行为被举报，他们将被从验证人节点池中移除（也就是冻结），并且在他们移除的时候不会得到奖励。他们将立即被视为不活跃的验证人，并将失去提名人。他们需要重新发布验证意图和收集提名人的支持。

# 5. 源码分析

```rust
parameter_types! {
	pub const SessionsPerEra: sp_staking::SessionIndex = 6;
	pub const BondingDuration: pallet_staking::EraIndex = 24 * 28;
	pub const SlashDeferDuration: pallet_staking::EraIndex = 24 * 7; // 1/4 the bonding duration.
	pub const RewardCurve: &'static PiecewiseLinear<'static> = &REWARD_CURVE;
	pub const MaxNominatorRewardedPerValidator: u32 = 256;
	pub OffchainRepeat: BlockNumber = 5;
}

use frame_election_provider_support::onchain;
impl pallet_staking::Config for Runtime {
	const MAX_NOMINATIONS: u32 = MAX_NOMINATIONS;
	//质押余额
	//调用Balances模块
	type Currency = Balances;
	//用于计算周期持续的时间，它保证启动的时候在on_finalize，在创世模块的时候不使用
	//调用Timestamp模块
	type UnixTime = Timestamp;
	//将余额转换为选举用的数字
	type CurrencyToVote = U128CurrencyToVote;
	//抵押者的总奖励=年通膨胀率*代币发行总量/每年周期数           //年通膨胀率=npos_token_staked / total_tokens
	// staker_payout = yearly_inflation(npos_token_staked / total_tokens) * total_tokens / era_per_year
	//RewardRemainder剩余的奖励 = 每年最大膨胀率*代币总数/每年周期数-给抵押者的总奖励
	//remaining_payout = max_yearly_inflation * total_tokens / era_per_year - staker_payout
	//如果最大奖励减去实际奖励还有剩余奖励，将剩余奖励收归国库，用于支持生态发展支出
	//Treasury模块：提供一个资金池，能够由抵押者们来管理，在这个国库系统中，能够从这个资金池中发起发费提案。
	type RewardRemainder = Treasury;
	//类型时间
	type Event = Event;
	//将惩罚的钱收入国库
	type Slash = Treasury; // send the slashed funds to the treasury.
	//奖励，奖励会在主函数进行
	type Reward = (); // rewards are minted from the void
	//每个周期Session数量
	type SessionsPerEra = SessionsPerEra;
	//必须存放的时间
	type BondingDuration = BondingDuration;
	//惩罚延迟的时间，必须小于BondingDuration，设置成0就会立刻惩罚，没有时间干预
	type SlashDeferDuration = SlashDeferDuration;
	/// A super-majority of the council can cancel the slash.
	//议会的绝大多数人认同可以取消延迟的惩罚
	type SlashCancelOrigin = EnsureOneOf<
		AccountId,
		EnsureRoot<AccountId>,
		pallet_collective::EnsureProportionAtLeast<_3, _4, AccountId, CouncilCollective>
	>;
	//给session提供的接口
	type SessionInterface = Self;
	//每个周期的花费
	type EraPayout = pallet_staking::ConvertCurve<RewardCurve>;
	//准确估计下一个session的改变，或者做一个最好的猜测
	type NextNewSession = Session;
	//为每个验证者奖励的提名者的最大数目。
	//对于每个验证者，只有$ MaxNominatorrewardedPervalidator最大的Stakers可以申请他们的奖励。 这用于限制提名人支付的I / O成本。
	type MaxNominatorRewardedPerValidator = MaxNominatorRewardedPerValidator;
	//提供选举功能
	type ElectionProvider = ElectionProviderMultiPhase;
	type GenesisElectionProvider =
		onchain::OnChainSequentialPhragmen<pallet_election_provider_multi_phase::OnChainConfig<Self>>;
	//权重信息
	type WeightInfo = pallet_staking::weights::SubstrateWeight<Runtime>;
}
```

```rust
//基本模块
		System: frame_system::{Pallet, Call, Config, Storage, Event<T>},
		Utility: pallet_utility::{Pallet, Call, Event},

		//session必备前置模块
		//BABE 模块通过从 VRF 算法的输出中收集链上随机因子，和有效管理区块的周期更替来实现 BABE 共识机制的部份功能。
		Babe: pallet_babe::{Pallet, Call, Storage, Config, ValidateUnsigned},
		//Timestamp模块提供了获取和设置链上时间的功能。
		Timestamp: pallet_timestamp::{Pallet, Call, Storage, Inherent},
		//Indices：为新创建的帐户分配索引。索引是地址的缩写形式。
		Indices: pallet_indices::{Pallet, Call, Storage, Config<T>, Event<T>},
		//Balances 模块提供了帐户和余额的管理功能。
		Balances: pallet_balances::{Pallet, Call, Storage, Config<T>, Event<T>},
		//TransactionPayment：提供计算预分派事务费用的基本逻辑。
		TransactionPayment: pallet_transaction_payment::{Pallet, Storage},

		//共识模块
		//Authorship 模块用于追踪当前区块的创建者，以及邻近的 “叔块”。
		Authorship: pallet_authorship::{Pallet, Call, Storage, Inherent},
		//选举模块
		ElectionProviderMultiPhase: pallet_election_provider_multi_phase::{Pallet, Call, Storage, Event<T>, ValidateUnsigned},
		//质押模块
		Staking: pallet_staking::{Pallet, Call, Config<T>, Storage, Event<T>},
		//Session 模块允许验证人管理其会话密钥，提供了更改会话长度的及处理会话轮换的功能。
		Session: pallet_session::{Pallet, Call, Storage, Event, Config<T>},
		//GRANDPA模块通过维护一个服务于native代码的GRANDPA权威集，以拓展GRANDPA的共识系统。
		Grandpa: pallet_grandpa::{Pallet, Call, Storage, Config, Event, ValidateUnsigned},
		//历史模块
		Historical: pallet_session_historical::{Pallet},
		//I'm Online 模块允许验证人在每次新会话中广播一次心跳，以表明该节点处于在线状态。
		ImOnline: pallet_im_online::{Pallet, Call, Storage, Event<T>, ValidateUnsigned, Config<T>},
		//Substrate 的 core/authority-discovery 库使用了 Authority Discovery 模块来获取当前权威集，获取本节点权威 ID，以及签署和验证与本节点与其他权威节点之间交换的消息。
		AuthorityDiscovery: pallet_authority_discovery::{Pallet, Config},
		//Offences 模块用于记录着被举报的违规行为。
		Offences: pallet_offences::{Pallet, Storage, Event},

		//管理模块
		//奖金模块
		Bounties: pallet_bounties::{Pallet, Call, Storage, Event<T>},
		//
		Tips: pallet_tips::{Pallet, Call, Storage, Event<T>},
		//Treasury pallet提供了一个由系统利益相关者管理的资金池，以及一个用于从该资金池中做支出提案的结构。
		Treasury: pallet_treasury::{Pallet, Call, Storage, Config, Event<T>},
		//Collective 模块可使某些指定账号集可通过让某些特殊来源的函数调用来得知它们的一些集体信息。
		Council: pallet_collective::<Instance1>::{Pallet, Call, Storage, Origin<T>, Event<T>, Config<T>},
		TechnicalCommittee: pallet_collective::<Instance2>::{Pallet, Call, Storage, Origin<T>, Event<T>, Config<T>},
		//Democracy 模块提供了一个民主系统来处理持份者的投票事宜。
		Democracy: pallet_democracy::{Pallet, Call, Storage, Config<T>, Event<T>},
		//选举模块
		Elections: pallet_elections_phragmen::{Pallet, Call, Storage, Event<T>, Config<T>},
		//Membership 模块允许控制一组AccountId的成员资格，它对于管理集合类型的成员资格十分有用。
		TechnicalMembership: pallet_membership::<Instance1>::{Pallet, Call, Storage, Event<T>, Config<T>},


		//其他
		//Sudo pallet用来授予某个账户 (称为 "sudo key") 权限去执行需要Root权限的交易函数，或者是指定一个新账户来替代掉原来的sudo key。
		Sudo: pallet_sudo::{Pallet, Call, Config<T>, Storage, Event<T>},
		//Contracts 模块为 runtime 提供了部署和执行 WebAssembly 智能合约的能力。
		Contracts: pallet_contracts::{Pallet, Call, Storage, Event<T>},
		RandomnessCollectiveFlip: pallet_randomness_collective_flip::{Pallet, Storage},
		Identity: pallet_identity::{Pallet, Call, Storage, Event<T>},
		Society: pallet_society::{Pallet, Call, Storage, Event<T>, Config<T>},
		Recovery: pallet_recovery::{Pallet, Call, Storage, Event<T>},
		Vesting: pallet_vesting::{Pallet, Call, Storage, Event<T>, Config<T>},
		Scheduler: pallet_scheduler::{Pallet, Call, Storage, Event<T>},
		Proxy: pallet_proxy::{Pallet, Call, Storage, Event<T>},
		Multisig: pallet_multisig::{Pallet, Call, Storage, Event<T>},
		Assets: pallet_assets::{Pallet, Call, Storage, Event<T>},
		Mmr: pallet_mmr::{Pallet, Storage},
		Lottery: pallet_lottery::{Pallet, Call, Storage, Event<T>},
		Gilt: pallet_gilt::{Pallet, Call, Storage, Event<T>, Config},
		Uniques: pallet_uniques::{Pallet, Call, Storage, Event<T>},
		TransactionStorage: pallet_transaction_storage::{Pallet, Call, Storage, Inherent, Config<T>, Event<T>},
```

