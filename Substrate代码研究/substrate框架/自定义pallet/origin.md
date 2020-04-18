runtime的origin是可分派的函数用来检查这个调用来源的。

# 原始Origins
substrate定义了三个原始的origins，您可以在您的runtime模块中使用。
```rust
pub enum RawOrigin<AccountId> {
    Root,
    Signed(AccountId),
    None,
}

```
* Root: 系统级的来源，这个是最高级的权限。
* Signed: 交易来源，这个是被某个公钥签名过的，包含了签名者的ID。
* None: 缺少来源，这个需要被全体验证者同意或者被包含的某个功能模块验证。

# 自定义的origins
你可以定义你定制化的origin，可以在你的runtime函数中认证检查使用。

# 自定义的origins调用
您可以在您的runtime中创建任意源的调用。例如：
```rust
// Root
proposal.dispatch(system::RawOrigin::Root.into())

// Signed
proposal.dispatch(system::RawOrigin::Signed(who).into())

// None
proposal.dispatch(system::RawOrigin::None.into())
```

您可以查看[sudo模块]来得到这个的一个可行实现。