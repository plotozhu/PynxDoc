当 Substrate 运行时模块想要将运行时中的变更或状况通知给外部实体（如用户、区块链浏览器，或 dApps）时，可以触发事件。
你可以定义你的模块会触发哪些事件，在这些事件中包含什么信息，以及什么时候触发这些事件。
# 声明事件
使用 decl_event! 宏创建运行时事件。
```rust 
decl_event!{
    pub enum Event<T> where
        AccountId = <T as system::Trait>::AccountId,
    {
        ValueSet(u64, AccountId),
    }
}
```
枚举类型 Event 需要在运行时的 配置特性 中进行声明。
```rust
pub trait Trait: system::Trait {
    type Event: From<Event<Self>> + Into<<Self as system::Trait>::Event>;
}
```
#  向运行时暴露事件
您的模块中的事件需要暴露给 Substrate 的运行时(/runtime/src/lib.rs)。
首先你需要在你的模块配置特性中实现事件类型：
```rust
// runtime/src/lib.rs
impl template::Trait for Runtime {
    type Event = Event;
}
```
然后您需要将该 Event 类型添加到 construct_runtime! 宏:
```rust
// runtime/src/lib.rs
construct_runtime!(
    pub enum Runtime where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic
    {
        // --snip--
        TemplateModule: template::{Module, Call, Storage, Event<T>},
        //--add-this------------------------------------->^^^^^^^^
    }
);
```
**注意：** 您不一定需要 <T> 参数，取决于您的事件是否使用范型。 在我们的例子中，它的确需要，所有在上面被包括在内。
# 保存事件
Substrate 在 decl_module! 宏中定义了一个默认实现，用于保存事件。
```rust 
decl_module! {
    pub struct Module<T: Trait> for enum Call where origin: T::Origin {
        // Default implementation of `deposit_event`
        fn deposit_event() = default;

        fn set_value(origin, value: u64) {
            let sender = ensure_signed(origin)?;
            // --snip--
            Self::deposit_event(RawEvent::ValueSet(value, sender));
        }
    }
}
```
此函数的默认行为是从 SRML 系统模块调用 deposit_event。  

此函数将事件存在System模块的该区块的运行时存储中。 在一个新区块的开头，System 模块自动删除前一个区块存储的所有事件。

使用默认实现保存的事件将被下游库直接支持，例如 Polkadot-JS api。然而，如果你想以不同方式处理事件，你可以实现你自己的 deposit_event 函数。

# 支持的类型
事件可以发布任何支持 Parity SCALE codec 的类型的数据。  
在您想要使用 Runtime 泛型的情况下，如 AccountId 或 Balances，您需要引入一个 where 从句 来定义上面示例所示的类型。  
# 监听事件
Substrate JSON RPC 并不直接暴露查询事件的端点。 如果您使用了默认实现功能，您可以通过查询 System 模块的存储来查看当前区块的事件列表。 另外， Polkadot-JS api 支持对运行时事件的 WebSocket 订阅。
# 后续步骤
了解更多  
了解更多关于 Substrate 运行时开发中使用的 宏。  
了解更多关于使用 Polkadot JS api 的信息。  
# 示例
These Substrate Recipes offer examples of how runtime events are used:

A pallet that implements standard events

A pallet that does not emit events with generic types
#  参考文档
访问 decl_event! 宏 的参考文档。
访问 decl_module! 宏 的参考文档。
访问 construct_runtime! 宏 的参考文档。