# 示例托盘

## 示例：
一个简单的FRAME托盘演示了大多数FRAME运行时通用的概念，API和结构。

## 文档注释要求
（即`///注释`）-应该伴随着托盘函数并仅限于托盘接口，而不是托盘的内部实现。仅状态输入，输出，并简要说明是否需要root来调用它，并且无需重复源代码详细信息。

将每个文档注释的首字母大写，并以结尾
一个句号。详情查看[使用文档注释注释源代码的通用示例]("https:github.com/paritytech/substrate#72-contributing-to-documentation-for-substrate-packages")
* 自我记录代码-尝试将代码重构为自我记录。
* 代码注释-用简短的解释补充复杂的代码，而不是每行代码。
* 标识符-被反引号包围（即`INHERENT_IDENTIFIER`，`InherentType`，`u64`
* 使用场景-应该是简单的doctest。编译器应确保它们保持有效。
* 扩展教程-应该移至外部文件并进行参考。
* 必填-包括指定了**必须**的所有节/小节。
* 可选-可以选择包括指定了**允许**的部分/子部分。

## 文档模板

将[文档模板](./自定义pallet的文档模板.md)从frame/example/src/lib.rs复制并粘贴到文件中
您自己的自定义托盘的`frame/<插入自定义托盘名称>/src/lib.rs`并完成它。  

## 定制交易权重示例
为调度调用`set_dummy（）`量身定制的自定义权重计算器。实际上这个检查并根据参数决定权重。  
`WeightData <T>`特征可以访问调用中的参数，并且为其分配权重。尽管如此，特征本身不能对泛型中的参数（`T`）做任何假设。根据我们的需求，我们可以在实现特征时，用更具体的类型替换`T`。 `decl_module！`期望用任何实现了`WeighData <T>`的调度参数的元组来替换`T`。这正是我们下面的实现的结果。

`WeightForSetDummy`的规则如下：
- 每次调度的最终权重计算为调用的参数乘以`WeightForSetDummy`的构造函数中的参数。
- 如果调用的参数大于1000，则分配一个调度类`operational`。
```rust 
struct WeightForSetDummy<T: pallet_balances::Trait>(BalanceOf<T>);

impl<T: pallet_balances::Trait> WeighData<(&BalanceOf<T>,)> for WeightForSetDummy<T>
{
    fn weigh_data(&self, target: (&BalanceOf<T>,)) -> Weight {
        let multiplier = self.0;
        (*target.0 * multiplier).saturated_into::<Weight>()
    }
}

impl<T: pallet_balances::Trait> ClassifyDispatch<(&BalanceOf<T>,)> for WeightForSetDummy<T> {
    fn classify_dispatch(&self, target: (&BalanceOf<T>,)) -> DispatchClass {
        if *target.0 > <BalanceOf<T>>::from(1000u32) {
            DispatchClass::Operational
        } else {
            DispatchClass::Normal
        }
    }
}

impl<T: pallet_balances::Trait> PaysFee<(&BalanceOf<T>,)> for WeightForSetDummy<T> {
    fn pays_fee(&self, _target: (&BalanceOf<T>,)) -> bool {
        true
    }
}
``` 
## 配置特征
我们的货盘的配置特征。我们所有的类型和常量都在这里。如果这个托盘依赖于其他特定的托盘，它们的配置特征也应该添加到我们的隐式特征列表中。

`frame_system :: Trait`应该始终包含在我们的隐含特征中。
```rust
pub trait Trait: pallet_balances::Trait {
    /// The overarching event type.
    type Event: From<Event<Self>> + Into<<Self as frame_system::Trait>::Event>;
}
```
## decl_storage!
此托盘的"存储"特征及其实现的宏。这允许安全地使用Substrate存储数据库，因此您可以在模块之间隔离相关的内容。  
更新存储名称非常重要，这样你的存储项目才能与其他托盘隔离。  
这是一个存储声明的格式：  
`pub? Name get(fn getter_name)? [config()|config(myname)] [build(|_| {...})] : <type> (= <new_default_value>)?;`  
此处`Type`是下面两者之一：
- `Type` (基本类型值的条目); or
- `map hasher(HasherKind) KeyType => ValueType` (映射表条目)。  

注意有存储类型的声明两个可选的修饰符：
 - `Foo: Option<u32>`:
   - `Foo::put(1); Foo::get()` 返回 `Some(1)`;
   - `Foo::kill(); Foo::get()` 返回 `None`.
 - `Foo: u32`:
   - `Foo::put(1); Foo::get()` 返回 `1`;
   - `Foo::kill(); Foo::get()` 返回 `0` (u32::default()).

例如：`Foo: u32`;  
又例如：`pub Bar get(fn bar): map hasher(blake2_128_concat) T::AccountId => Vec<(T::Balance, u64)>`;  

对于基本类型值的条目，您将得到实现了`frame_support::StorageValue`的类型，对于映射类型条目，您将得到实现了`frame_support::StorageMap`的类型。 
如果有一个读取函数：(`get(getter_name)`)，那么您的托盘将为基本类型值条目配备`fn getter_name() -> Type`接口，或者是为映射表类型配备了`fn getter_name(key: KeyType) -> ValueType`接口。
``` rust 
decl_storage! {

    trait Store for Module<T: Trait> as Example {
      
        Dummy get(fn dummy) config(): Option<T::Balance>;

        Bar get(fn bar) config(): map hasher(blake2_128_concat) T::AccountId => T::Balance;

        Foo get(fn foo) config(): T::Balance;
    }
}
```
## decl_event!
事件仅仅意味着报告特定的状态、环境发生了，因此用户、Dapp和/或链浏览器可以发现他们感兴趣的部分，否则会难以检测。
```rust
decl_event!(
    pub enum Event<T> where B = <T as pallet_balances::Trait>::Balance {
        // 仅是一个简单的'enum'，这里是一个'空'事件，确定可以被编译
        /// Dummy 事件,这里用了一个泛型
        Dummy(B),
    }
);
```
## decl_module! 
模块定义，这个定义指出了我们处理的入口点，宏将会管理参数和分发函数的调度。

通过签名和提交一个外源操作（交易），任何人都可以执行这些函数。需要保证执行这些调用所需的时间、内存和存储空间与调用者支付的成本相对应，否则就会导致调用难以被执行。

通常您可能希望这个分解成三组：
- 由外部帐户签名的公开调用
- 仅有管理系统产生的根（链）调用
- 未签名的调用，可能为以下两者之一：
  - "固有数据外源"：可能由构造区块的区块创建者生成的的；
  - "未签名的交易"：网络可识别的内部信息，可以由Runtime验证。

初始化这个可分发函数的信息（创建者）是作为第一个参数提供的(origin)。因此函数总是看起来象这样：
`fn foo(origin, bar: Bar, baz: Baz) -> Result;`
`Result`是语法需要的（将被展开成常规的分发器结果：`Result<(), &'static str>`）。

当您在后续的托盘`impl`中实现他们时，您必须指定`origin`的完整类型：
`fn foo(origin: T::Origin, bar: Bar, baz: Baz) { ... }`

与上述说明对应，在`frame_system::Origin`中有三种来源类型的枚举：`::Signed(AccountId)`, `::Root` 和 `::None`。 在您的函数中，第一件事就是需要匹配他们。在系统中为您准备了三个方便的函数来匹配他们并且返回常规结果：`ensure_signed`,
// `ensure_root` 和 `ensure_none`

```rust
decl_module! {
    // Simple declaration of the `Module` type. Lets the macro know what its working on.
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        /// 尽管这个托盘的事件之一使用了默认的实现，仍然可以进行自定义的实现，对于非泛型事件，需要丢弃泛型的参数，
        // 因此它看起来是这样：`fn deposit_event() = default;`     
        fn deposit_event() = default;

        /// 这里您的公开接口，要特别小心！
        /// 这是外部世界如何与托盘交互的一个简单示例
        //  它仅通过`increase_by`来增加`Dummy`值 .
        //
        // 因为这是一个分发函数，需要记住两个特别重要的事：
        //
        // - 坚决不能PANIC:无论何种情况下（保存，或者存储系统被破坏进入不可修复的状态）都不可以进行panic.
        // - 在错误时不能有任何副作用: 这个函数必须要么完全成功（返回`Ok(())`），或者在错误时（返回`Err('Some reason')`）必须不能对存储系统产生任何副作用。
        //
        // 第一个相关性很容易实现 -- 确认在产品代码的所有的逻辑中移除panic产生器（你随便可以怎么做，对吧？）。
        // 要确认第二个，你应该在函数之上进行所有的测试来验证，这个和检查发送者(`origin`)的方式是类似的，或者这个状态使操作有意义。
        //
        // 一旦你确认所有的都是好的，然后就可以执行操作并修改存储。
        // 如果你无法通过充分的计算确认操作是否会成功，那么你就有了经典的区块链攻击场景。
        //通常处理这个的方式是在操作上附加一个约束（保证金）。在存储操作前，可以先预订发送者的帐户里的一些资产值（`Balances`托盘具有适用用此场景的`reserve`函数），
        //如果事实上您无法继续完成此操作，则此金额应足以支付已经实质执行了的操作费用。
        //
        // 如果最终证明操作正常，因此检查的费用应由网络承担，则可以退还保留的押金。但是，如果发现该操作无效并且浪费了算力，则可以将其销毁此资产或移交给其他人（出块者或是任何帐户）。

        //
        // 安全绑定通过与该系统交互的任何人要么（提交）可以执行（的操作），要么为无法执行的操作付费的方式，确保攻击者无法使用它。 

        //
        // 如果你不尊重这些规则，您的链将非常有可能是可攻击的。
        //
        // 每个交易都定义一个 `#[weight]` 属性来收集一组关于分派的静态（权重）信息。 FRAME系统以及FRAME执行托盘使用这些信息来正确地执行交易，从而将维持链的负载维持在一定的速率上（指的是TPS，每个区块限定了交易数，也就限制了TPS）。
        //
        //  `#[weight]` 属性的"右值"可以是实现了名为[`WeighData`] 和 [`ClassifyDispatch`]特征的任意类型，前者收集了交易的权重（代表完全的执行时间和难度的数值），后者演示了调用的[`DispatchClass`]。更高的权重音意味着更大的交易（在一个区块中只能放更少的交易）。
        #[weight = SimpleDispatchInfo::FixedNormal(10_000)]
        fn accumulate_dummy(origin, increase_by: T::Balance) -> DispatchResult {
            // 这个是公开的函数接口，所以我们需要保证origin是某个签名的帐户。
            let _sender = ensure_signed(origin)?;

            // 从存储中读取dummy的值
            // let dummy = Self::dummy();
            // 将同样使用存储条目自身的的`::get` :
            // let dummy = <Dummy<T>>::get();

            // 计算新值
            // let new_dummy = dummy.map_or(increase_by, |dummy| dummy + increase_by);

            // 把新值放回给存储系统.
            // <Dummy<T>>::put(new_dummy);
            // 使用引用也同样可以工作：
            // <Dummy<T>>::put(&new_dummy);

            // 这是一个新的读取并且修改值的方法。（这个是原子操作哦）
            <Dummy<T>>::mutate(|dummy| {
                let new_dummy = dummy.map_or(increase_by, |dummy| dummy + increase_by);
                *dummy = Some(new_dummy);
            });

            // 让我们放置一个事件，让外部世界了解这事发生了。
            Self::deposit_event(RawEvent::Dummy(increase_by));

            // 可以了
            Ok(())
        }

        /// 特权调用，这个案例将用一个新的值来重置dummy。
        // 这个特权调用的实现中，`origin`必须是ROOT因此他不是（直接）从外源，而是整个系统决定执行的。
        //不同的runtime因不同的原因允许特权调用，我们并不关心为什么。因为这个是特权调用，
        //我们可以假设这是一次性操作，并且实际的处理/存储/内存可以在无需担心游戏或是攻击场景下使用。
        //如果没有明确指定`Result`作为返回值，它将被自动添加，并且将会返回 `Ok(())`
        #[weight = WeightForSetDummy::<T>(<BalanceOf<T>>::from(100u32))]
        fn set_dummy(origin, #[compact] new_value: T::Balance) {
            ensure_root(origin)?;
            // Put the new value into storage.
            <Dummy<T>>::put(new_value);
        }

        // 这个函数签名也可以类似这样: `fn on_initialize()`.
        // 该函数同样可以使用和其它函数一样的权重签注。唯一的差别是如果没有签注，默认的是`SimpleDispatchInfo::zero()`, 这个的解析结果将是无权重.
        fn on_initialize(_n: T::BlockNumber) -> Weight {
            // 任何在区块开始的时候，需要执行的动作。
            // 现在这里什么都没有做

            SimpleDispatchInfo::default().weigh_data(())
        }

        // 这个函数签名可以类似这样： `fn on_finalize()`
        fn on_finalize(_n: T::BlockNumber) {
            // 任何需要在区块结束的时候做的事
            // 这里仅仅是删除了dummy存储条目
            <Dummy<T>>::kill();
        }

        // 将会在每个区块结果后运行的，可以访问一组扩展API的运行时代码
        //
        // 例如，您可以生成为即将生成的产生的区块生成外源操作
        fn offchain_worker(_n: T::BlockNumber) {
            // 这里我们什么都没有做
            // 但是我们可以使用`sp_io::submit_extrinsic`来分派extrinsic (transaction/unsigned/inherent)  
        }
    }
}
```

## 实现模块
托盘的主要实现块。此处的函数可分成三类：
- 公开接口 这些是公开的，并且通常属于检查器函数，不会操作或是写入存储。
- 私有函数，这些是您私用的工具，对其他托盘不可用
- 3呢？

```rust
impl<T: Trait> Module<T> {
    // 添加公开的不可变的和私有的可变函数.
    #[allow(dead_code)]
    fn accumulate_foo(origin: T::Origin, increase_by: T::Balance) -> DispatchResult {
        let _sender = ensure_signed(origin)?;

        let prev = <Foo<T>>::get();
        // 因为Foo具有'default'定义，闭包中的'foo'类型是原始类型而不是Option<>类型.
        // 查看代码，Foo是存储类型 T::Balance
        let result = <Foo<T>>::mutate(|foo| {
            *foo = *foo + increase_by;
            *foo
        });
        assert!(prev + increase_by == result);

        Ok(())
    }
}
```

## 签名扩展
与其他FRAME托盘类似，您的托盘也可以定义一个签名扩展并在交易之前（之后）执行一些预处理（后处理）。签名扩展名可以是任何实现了`SignedExtension`的可解码类型。请参阅该特征定义以获取完整的列表范围。按照惯例，您可以按照以下方法为托盘创建扩展：  
- 如果扩展不包含任何数据，则使用仅带有" marker"的元组结构（编译器需要这个来接受`T：Trait`）就足够了。  
- 否则，创建一个包含外部数据的元组结构。当然，要实现整个结构是可解码的，结构内每个单独的项目必须是可解码的。

请注意，通过实现`additional_signed`，签名扩展可以表明在交易的`_signing payload_ `中必须包含特定的数据。本示例将不涵盖此类扩展，可以另外参见FRAME系统中的`CheckRuntime`的例子。

使用扩展，您可以在每个交易的生命周期中添加一些钩子。要注意默认情况下，会将扩展应用于所有`Call`函数（即所有交易）。 `Call`枚举变量将提供给为SignedExtension的每个函数，因此，您可以根据需要基于托盘或是特定的函数调用需要进行过滤。

同样提供了一些其他信息，例如编码长度，一些静态调度信息，例如权重和交易的发送者（如果已经签名）。

可以添加到已签名扩展中的钩子的完整列表可以参照[此处](https:crates.parity.io/sp_runtime/traits/trait.SignedExtension.html)。

签名扩展将在substrate链中的runtime文件中汇总。所有扩展应该汇总成一个元组，然后传递给runtime中定义的`CheckedExtrinsic`和`UncheckedExtrinsic`类型。在`node/runtime`中查看`pub type SignedExtra =（...）`或者查看" node-template"中的这个例子。

这里是一个简单的检查`set_dummy`的签名扩展，在示例中，它提高了优先级并且打印一些日志。  
另外： 它将丢弃编码后长度大于200字节的交易，没有特定的理由，只是演示签名扩展的能力。

```rust
impl<T: Trait + Send + Sync> SignedExtension for WatchDummy<T> {
    const IDENTIFIER: &'static str = "WatchDummy";
    type AccountId = T::AccountId;
    // Note that this could also be assigned to the top-level call enum. It is passed into the
    // Balances Pallet directly and since `Trait: pallet_balances::Trait`, you could also use `T::Call`.
    // In that case, you would have had access to all call variants and could match on variants from
    // other pallets.
    type Call = Call<T>;
    type AdditionalSigned = ();
    type DispatchInfo = DispatchInfo;
    type Pre = ();

    fn additional_signed(&self) -> sp_std::result::Result<(), TransactionValidityError> { Ok(()) }

    fn validate(
        &self,
        _who: &Self::AccountId,
        call: &Self::Call,
        _info: Self::DispatchInfo,
        len: usize,
    ) -> TransactionValidity {
        // if the transaction is too big, just drop it.
        if len > 200 {
            return InvalidTransaction::ExhaustsResources.into()
        }

        // check for `set_dummy`
        match call {
            Call::set_dummy(..) => {
                sp_runtime::print("set_dummy was received.");

                let mut valid_tx = ValidTransaction::default();
                valid_tx.priority = Bounded::max_value();
                Ok(valid_tx)
            }
            _ => Ok(Default::default()),
        }
    }
}
```
这里的WatchDummy是一个不包含任何数据的结构，因此只需要带一个marker:
```rust
#[derive(Encode, Decode, Clone, Eq, PartialEq)]
pub struct WatchDummy<T: Trait + Send + Sync>(PhantomData<T>);
```