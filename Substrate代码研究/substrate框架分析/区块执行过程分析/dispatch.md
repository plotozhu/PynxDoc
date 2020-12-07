# 说明
dispatch实现了可分派的任务的动作逻辑


## 名词
weight: 重量，砝码，是指一个可分派动作需要消耗的GAS值
## 声明

```rust
# #[macro_use]
# extern crate frame_support;
# use frame_support::dispatch;
# use frame_system::{self as system, Trait, ensure_signed};
decl_module! {
	pub struct Module<T: Trait> for enum Call where origin: T::Origin {
    
        //私有的函数同样是可分派的，只是不能被其他FRAME中的托盘调用
		#[weight = 0]
		fn my_function(origin, var: u64) -> dispatch::DispatchResult {
    				// 函数的实现
				Ok(CsaActionItem::from(()))
		}

		// 公共函数可以被分派和其他的FRAME中的托盘调用 TODO 怎么调用？<Frame as ....>:xxx
		#[weight = 0]
			pub fn my_public_function(origin) -> dispatch::DispatchResult {
			// Your implementation
				Ok(CsaActionItem::from(()))
		}
		}
}
# fn main() {}
```

 声明中和区块头一起，包括：  
 * `Module`: 由宏生成的结构，实现了 `Trait`特征。
 * `Call`: 每个托盘生成的枚举，实现了[`Callable`](./dispatch/trait.Callable.html).
 * `origin`:  `T::Origin`的别名, 由[`impl_outer_origin!`]宏声明(./macro.impl_outer_origin.html) 。
 * `Result`: 托盘函数期望的返回类型

 可分派函数的第一个参数必须是`origin`。
 
 ### 简写例子
 可分派函数可以简写，宏总是将简写过的函数自动扩展成返回[`DispatchResultWithPostInfo`] 类型，下面示例中的两个函数是等价的：
  ```rust
 #[macro_use]
 extern crate frame_support;
 use frame_support::dispatch;
 use frame_system::{self as system, Trait, ensure_signed};
 decl_module! {
 	pub struct Module<T: Trait> for enum Call where origin: T::Origin {
 		#[weight = 0]
 		fn my_long_function(origin) -> dispatch::DispatchResultWithPostInfo {
				// Your implementation
 			Ok(dispatch::CsaActionItem::from(()))
 		}

 		#[weight = 0]
 		fn my_short_function(origin) {
				// Your implementation
 		}
		}
 }
 # fn main() {}
 ```
 `DispatchResultWithPostInfo`是带有PostInfo的分发返回值，它是`sp_runtime::DispatchResultWithInfo<crate::weights::PostDispatchInfo>;`的简写，如果需要更多的PostInfo信息，可以使用自定义的类来替换`crate::weights::PostDispatchInfo`
### 仅消耗注解中的部分重量
默认每个可调用函数消耗所有的通过#[weight]注解的全部静态重量。然而存在仅应该消息这个注解中的一部分重量的用例。在那场景中，静态的重量按照单个可分派（函数）增加，最后的差值在dispatch后退回。

为了实现这个特性，函数必须返回`DispatchResultWithPostInfo`而不是默认的`DispatchResult`，从而可以返回实际消耗的重量。想要在返回错误时消耗掉非默认的重量， 可以用[`WithPostDispatchInfo::with_weight`](./weight/trait.WithPostDispatchInfo.html) 实现带有重量信息参数的函数。

 ```rust
 #[macro_use]
 extern crate frame_support;
 use frame_support::dispatch::{DispatchResultWithPostInfo, WithPostDispatchInfo};
 use frame_system::{self as system, Trait, ensure_signed};
 decl_module! {
 	pub struct Module<T: Trait> for enum Call where origin: T::Origin {
 		#[weight = 1_000_000]
 		fn my_long_function(origin, do_expensive_calc: bool) -> DispatchResultWithPostInfo {
 			ensure_signed(origin).map_err(|e| e.with_weight(100_000))?;
 			if do_expensive_calc {
 				// do the expensive calculation
 				// ...
 				// return None to indicate that we are using all weight (the default)
 				return Ok(None.into());
 			}
 			// expensive calculation not executed: use only a portion of the weight
 			Ok(Some(100_000).into())
 		}
 	}
 }
 # fn main() {}
 ```

 ### 特权函数例子

一个特权函数检查调用发起者是否是`ROOT`.

 ```rust
  #[macro_use]
  extern crate frame_support;
  use frame_support::dispatch;
  use frame_system::{self as system, Trait, ensure_signed, ensure_root};
 decl_module! {
 	pub struct Module<T: Trait> for enum Call where origin: T::Origin {
 		#[weight = 0]
			fn my_privileged_function(origin) -> dispatch::DispatchResult {
 			ensure_root(origin)?;
				// 实际实现
 			Ok(dispatch::CsaActionItem::from(()))
 		}
		}
 }
 # fn main() {}
 ```

 ## 多实例模块例子

 可以创建在一个runtime支持多个实例的substrate模块，例如，可以多次添加[Balances module](../pallet_balances/index.html) 到你的runtime以实现多个独立数字资产的区块链系统。这里是一个如何通过 `decl_module!` 宏声明这个模块的例子。

 ```rust
 # #[macro_use]
 # extern crate frame_support;
 # use frame_support::dispatch;
 # use frame_system::{self as system, ensure_signed};
 # pub struct DefaultInstance;
 # pub trait Instance {}
 # impl Instance for DefaultInstance {}
 pub trait Trait<I: Instance=DefaultInstance>: system::Trait {}

 decl_module! {
 	pub struct Module<T: Trait<I>, I: Instance = DefaultInstance> for enum Call where origin: T::Origin {
 		// Your implementation
 	}
 }
 # fn main() {}
 ```

注意: 必须调用`decl_storage` 来生成 `Instance`和 `DefaultInstance`（可选）特征。

 ## Where 语句

 除了默认的`origin: T::Origin`之外,你还可以传递其他的约束到模块声明上。此处的Where约束语句将会被复制到所有使用该宏生成的类型。 不支持使用`+`链式实现多个特征的约束，如果需要对一个类型施加多个约束，需要初分开成多个约束（语句）

 ```rust
 # #[macro_use]
 # extern crate frame_support;
 # use frame_support::dispatch;
 # use frame_system::{self as system, ensure_signed};
 pub trait Trait: system::Trait where Self::AccountId: From<u32> {}

 decl_module! {
 	pub struct Module<T: Trait> for enum Call where origin: T::Origin, T::AccountId: From<u32> {
 		// Your implementation
 	}
 }
 # fn main() {}
 ```

 ## 保留函数
下列是保留函数的签名：
 
 * `deposit_event`: 放置一个事件的辅助函数 [event](https://docs.substrate.dev/docs/event-enum).
 默认操作是调用[System module](../frame_system/index.html)中的 `deposit_event` ，然而你可以在你的runtime中编写自己的实现。要使用默认的行为，把`fn deposit_event() = default;`添加到你的 `Module`中。
 下列保留函数同样使用区块高度(类型为`T::BlockNumber`)作为可选的参数输入:

 * `on_runtime_upgrade`: 在区块的最开始，当有runtime升级的时候，在`on_initialize`之前执行。这个将允许存储项在被使用之前先升级。因此, **必须避免调用其他模块**!! 使用此函数将实现
 [`OnRuntimeUpgrade`](../sp_runtime/traits/trait.OnRuntimeUpgrade.html) 特征。函数签名必须是 `fn on_runtime_upgrade() -> frame_support::weights::Weight`.

 * `on_initialize`: 在一个区块开始时被执行。使用此函数将会实现[`OnInitialize`](./trait.OnInitialize.html)特征.
 函数签名必须是以下两者之一:
   * `fn on_initialize(n: BlockNumber) -> frame_support::weights::Weight` 或
   * `fn on_initialize() -> frame_support::weights::Weight`

   * `on_finalize`: 在区块结束的时候执行。使用此函数将会实现[`OnFinalize`](./traits/trait.OnFinalize.html)特征。
    函数签名必须是以下两者之一:
   * `fn on_finalize(n: BlockNumber) -> frame_support::weights::Weight` 或
   * `fn on_finalize() -> frame_support::weights::Weight`

 * `offchain_worker`: 在一个区块开始的时候执行，提供一个为未来的区块使用的外源。使用此函数将实现
 [`OffchainWorker`](./traits/trait.OffchainWorker.html) 特征。

 ## 宏展开

```rust
pub struct Module<T: Trait> for enum Call where origin: T::Origin, T::AccountId: From<u32> {
	pub fn call_1(....) {

	}
	...
}
//Struct展开
pub struct Module<T>(sp_std::marker::PhantomData<(T)>)
where origin: T::Origin, 
T::AccountId: From<u32>;

//几个预定义的处理的展开
//...
impl<T:Trait> OnFinalize<T::BlockNumber> for Module<T> 
where origin: T::Origin, 
T::AccountId: From<u32>{
	//...
}
//...

//CALL展开
impl<T:Trait> Module<T> 
	where origin: T::Origin, 
	T::AccountId: From<u32>{
	pub fn call_1(origin:T::Origin)->DispatchResult{
		.....
	}
}
```

```rust 
	impl_outer_dispatch! {
		pub enum OuterCall for TraitImpl where origin: u32 {
			self::Test,
		}
	}

//[#...]
pub enum OuterCall { //(1)
	Test(dispatch::CallableCallFor<Test,TraitImpl>)
}

// and 
pub type CallableCallFor<A, T> = <A as Callable<T>>::Call;

// (1)相当于：
pub enum OuterCall {
	Test((Test as Callbale<TraitImpl>::Call))
}
```	