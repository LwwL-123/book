

# 权重Weight和基准Benchmarking



默认的Substrate块生产系统以一致的间隔生产块。这就是所谓的目标块时间。考虑到这一要求，基于Substrate的区块链只能在每个区块执行有限数量的extrinsics。根据计算复杂度、存储复杂度、使用的硬件和许多其他因素，执行一个外部因素所需的时间可能会有所不同。我们使用称为权重的通用度量来表示一个块可以容纳多少extrinsics。

我们使用称为权重的通用度量来表示一个块可以容纳多少extrinsics。

```
10^12 weight units = 1 second
1,000 weight units = 1 nanosecond
```

 这是在特定的参考硬件上测量的:Intel Core i7-7700K CPU, 64GB RAM和NVMe SSD。



Substrate期望基准测试能够为执行外部构件的最坏情况提供一个近似的最大值。如果采用了这种最坏的情况，用户将被收取费用，如果外部资源需要的资源更少，一些估计的权重和费用可以被返还。这在交易权重和费用一章中有进一步的解释。



## 步骤Steps

### 1. 编写benchmarking.rs

```rust
benchmarks! {
	//这将测量 [1..100] 范围内 b 的 `do_something` 的执行时间。
	do_something {
		let s in 0 .. 100;
		let caller: T::AccountId = whitelisted_caller();
	}: _(RawOrigin::Signed(caller), s)
	verify {
		assert_eq!(Something::<T>::get(), Some(s));
	}

	set_dummy_benchmark {
		// This is the benchmark setup phase
		let b in 1 .. 1000;
	}: set_dummy(RawOrigin::Root, b.into()) // 执行阶段只是运行 `set_dummy` 外部调用
	verify {
		// 这是可选的基准验证阶段，测试某些状态。
		assert_eq!(Pallet::<T>::dummy(), Some(b.into()))
	}

	accumulate_dummy {
		let b in 1 .. 1000;
		// 调用者帐户被基准测试宏列入数据库读写白名单.
		let caller: T::AccountId = whitelisted_caller();
	}: _(RawOrigin::Signed(caller), b.into())
}
```



### 2. 编译

```
// 编译
build --release --features runtime-benchmarks
```



### 3. 生成weights.rs文件

使用基准测试CLI，我们可以指定步骤和重复次数。这意味着遍历每个变量范围将采取多少步骤，以及执行状态将重复多少次。

```
// 根据模板生成weights文件
./target/release/node-template benchmark \         
    --chain ttc \																			# 链的启动方式
    --execution wasm \																# 永远用Wasm测试
    --wasm-execution compiled \												# 永远用`wasm-time`
    --pallet pallet_template \												# 选择pallet
    --extrinsic '*' \																	# 选择基准案例名称，使用'*'代替all
    --steps 20 \																			# 跨组件范围的步骤数(b = 0..1000,则依次增加50)
    --repeat 10 \																			# 重复基准测试的次数
    --raw \																						# 可选地将原始基准数据输出到标准输出
    --output ./pallets/template/src/weights.rs \			# 输出文件位置
    --template ./.maintian/frame-weight-template.hbs	# 可以选择模板文件生成
    
    
    
./target/release/node-template benchmark \
--chain dev \
--execution wasm \
--wasm-execution compiled \
--pallet pallet_resource_order \
--extrinsic '*' \
--steps 20 \
--repeat 10 \
--raw \
--output ./pallets/resource-order/src/weights.rs \
--template ./.maintian/frame-weight-template.hbs
```



### 4. 将 `WeightInfo` 加入到pallet

在`pallets/example/src/lib.rs`中

```rust
pub mod weights;
pub use weights::*;

// -- snip --

pub trait Config: frame_system::Config {
    // -- snip --

    /// Information on runtime weights.
    type WeightInfo: WeightInfo;
}
```



### 5. 编写自定义weight声明

对于每个可分派对象，引入适当的权重行，使用配置的WeightInfo类型确定权重。例如，T::WeightInfo::example将是weight返回的weight函数。

```rust
#[pallet::weight(T::WeightInfo::example(x.len()))]
fn example(origin: OriginFor<T>, arg: Vec<u32>) -> DispatchResult {
  // -- snip --
}
```



### 6. 添加WeightInfo

在`mock.rs`和`runtime`中定义`WeightInfo`

```rust
impl pallet_example::Config for Runtime {
  // -- snip --
  type WeightInfo = pallet_example::weights::SubstrateWeight<Runtime>;
}
```



例子:

```
git clone -b add_weights https://gitlab.servicechain.newtouch.com/tt-chain/ttchain-v2.git
```



