substrate中很重要的一个部分就是从节点（node/client到runtime），这个是如何从链下到链上的调用过程，由于过程中使用了宏，所以有很多操作被宏掩盖了。 本文是对这个过程的说明

# substrate runtime（运行时） api
substrate 运行时 api 是节点和运行时之间的关键接口，每个进入运行时的操作都是通过运行时api来完成的。运行时api是不固定的，每个substrate的用户都可以通过[`decl_runtime_apis`](macro.decl_runtime_apis.html)来定义自己的api，并且通过[`impl_runtime_apis`](macro.impl_runtime_apis.html)来实现他们。

所有的substrate的runtime都需要实现[`Core`]的api，这个api提供了每个runtime都需要导出的最基本的功能。

除了宏和[`Core`] api之外，这个模块提供了`Metadata`] runtime api,  [`ApiExt`] 特征,  [`CallApiAt`] 特征 和 [`ConstructRuntimeApi`] 特征。

在这个实现的元数据层次，客户端从客户视角调用生成的API 

## 把特征声明成运行时api
该宏将会创建两个声明，一个在客户侧使用，另外一个在运行时侧使用，运行时侧使用的声明隐藏在自身的Module之后。
客户侧的声明中每个函数多了两个额外的参数，`&self`和`at: &BlockId<Block>`，运行时侧的声明将会与给定的特征匹配（运行时没有这两个参数），还有一个例外：宏给客户侧和运行时侧都添加了一个额外的泛型参数`Block: BlockT`，这个泛型参数对于用户可用。
你应该使用[`impl_runtime_apis!`](macro.impl_runtime_apis.html)宏来实现这些宏。

### 例子

 ```rust
 sp_api::decl_runtime_apis! {
     // Declare the api trait.
     pub trait Balance {
        //  Get the balance.
         fn get_balance() -> u64;
        //  Set the balance.
         fn set_balance(val: u64);
     }

    //  You can declare multiple api traits in one macro call.
    //  In one module you can call the macro at maximum one time.
     pub trait BlockBuilder {
     //     The macro adds an explicit `Block: BlockT` generic parameter for you.
     //     You can use this generic parameter as you would defined it manually.
         fn build_block() -> Block;
     }
 }

 # fn main() {}
 ```
 ## 版本化运行时api特征
 为了支持特征的版本化，宏支持`#[api_version(1)]`属性，该属性支持把任何`u32`的值作为版本号。如果没有提供版本号，每个特征都有一个默认的版本号`1`。我们还支持方法的变化签名（此处的签名更多的被理解为标识），这个变化签名在方法的上部用`#[changed_in(2)]`属性来标识。使用此标签的方法可以通过名称`METHOD_before_version_VERSION`来调用。 这个方法仅支持在wasm中调用，在原生代码中调用将失败（改变了特定的版本！）。这样的方法不需要在运行时中实现，它需一个没有`#[changed_in(_)]`标签的默认实现，该方法将被默认调用。
  ```rust
 sp_api::decl_runtime_apis! {
      Declare the api trait.
     #[api_version(2)]
     pub trait Balance {
         // Get the balance.
         fn get_balance() -> u64;
         // Set balance.
         fn set_balance(val: u64);
         // Set balance, old version.
         //
         // Is callable by `set_balance_before_version_2`.
         #[changed_in(2)]
         fn set_balance(val: u16);
         // In version 2, we added this new function.
         fn increase_balance(val: u64);
     }
 }

 # fn main() {}
 ```

 # 使特征会以运行时api实现的标签

所有使用此宏的特征，需要使用[`decl_runtime_apis!`](macro.decl_runtime_apis.html)来声明，特征的实现应该依照 [`decl_runtime_apis!`](macro.decl_runtime_apis.html)宏的声明，除此之外，泛型`Block`类型需要作为每个运行时的API特征的第一个泛型参数。当特片作为一个运行时api时，需要通过一个路径来引用：比如 `impl my_trait::MyTrait for Runtime`。 宏将会使用此路径来访问运行时侧的特征。
这个宏同样产生一个客户侧访问的、`RuntimeApi`类型的api，这个类型隐藏在`RuntimeApi`被称为`std`的`feature`之后。

为了暴露出所有的实现的api特征，生成了常量`RUNTIME_API_VERSIONS`。该常量可以用来初始化`RuntimeVersion`的字段`apis`。

 ## 例子

 ```rust
 use sp_version::create_runtime_str;
 #
 # use sp_runtime::traits::{GetNodeBlockType, Block as BlockT};
 # use sp_test_primitives::Block;
 #
 #  The declaration of the `Runtime` type and the implementation of the `GetNodeBlockType`
 #  trait are done by the `construct_runtime!` macro in a real runtime.
 # pub struct Runtime {}
 # impl GetNodeBlockType for Runtime {
 #     type NodeBlock = Block;
 # }
 #
 # sp_api::decl_runtime_apis! {
 #      Declare the api trait.
 #     pub trait Balance {
 #          Get the balance.
 #         fn get_balance() -> u64;
 #          Set the balance.
 #         fn set_balance(val: u64);
 #     }
 #     pub trait BlockBuilder {
 #        fn build_block() -> Block;
 #     }
 # }

 /// 所有的运行时api实现必须在一个 macro 里完成！
 sp_api::impl_runtime_apis! {
 #   impl sp_api::Core<Block> for Runtime {
 #       fn version() -> sp_version::RuntimeVersion {
 #           unimplemented!()
 #       }
 #       fn execute_block(_block: Block) {}
 #       fn initialize_block(_header: &<Block as BlockT>::Header) {}
 #   }

     impl self::Balance<Block> for Runtime {
         fn get_balance() -> u64 {
             1
         }
         fn set_balance(_bal: u64) {
             // Store the balance
         }
     }

     impl self::BlockBuilder<Block> for Runtime {
         fn build_block() -> Block {
              unimplemented!("Please implement me!")
         }
     }
 }

 /// Runtime 版本，需要在每个运行时里实现。
 pub const VERSION: sp_version::RuntimeVersion = sp_version::RuntimeVersion {
     spec_name: create_runtime_str!("node"),
     impl_name: create_runtime_str!("test-node"),
     authoring_version: 1,
     spec_version: 1,
     impl_version: 0,
     // Here we are exposing the runtime api versions.
     apis: RUNTIME_API_VERSIONS,
     transaction_version: 1,
 };

 # fn main() {}
 ```

# 总结 *** 非常重要 ***
代码分成了链上(wasm)运行和链下运行部分(native code)。
