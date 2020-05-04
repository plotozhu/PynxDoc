OffChain Worker(离线工作机）是提供了离线的操作，在每个区块到达时就执行offchain动作，更值得研究的是系统为OffChain Worker提供了两个链下的接口，一个是http请求，一个是存储。 理解了Offchain就能够理解如何向链上函数提供链下的接口。

# Frame/system/src/offchain.rs
## 概览
这个文件提供了离线工作机所经常使用的调用，包括以下几个：
- 提交一个原始未签名的交易
- 提交一个具有签名负载的未签名交易
- 提交一个签名交易
  
## 用法
本模块的更具体的用法，请参考[`example-offchain-worker`](../../pallet_example_offchain_worker/index.html)

 ### 交易签名

 要想使用签名，应该实现以下几个特征：

 - [`AppCrypto`](./trait.AppCrypto.html): 定模块的签名辅助函数所需的应用密钥。
 - [`CreateSignedTransaction`](./trait.CreateSignedTransaction.html): 构建交易的方法

 #### 提交一个具有签名负载的未签名交易

 开始，实现了`SignedPayload`特征的负载实例应该已经定义好。更多信息请查看[`PricePayload`](../../pallet_example_offchain_worker/struct.PricePayload.html)

 需要所定义的负载可以被签名并且提交到链上。
 

 #### 提交签名后的交易

 可以用[`Signer`](./struct.Signer.html) 来签名和验证交易


## SubmitTransaction对象
 Provides the ability to directly submit signed and unsigned
 transaction onchain.

 For submitting unsigned transactions, `submit_unsigned_transaction`
 utility function can be used. However, this struct is used by `Signer`
 to submit a signed transactions providing the signature along with the call.

## Signer对象
 Provides an implementation for signing transaction payloads.

 Keys used for signing are defined when instantiating the signer object.
 Signing can be done using:

 - All supported keys in the keystore
 - Any of the supported keys in the keystore
 - An intersection of in-keystore keys and the list of provided keys

 The signer is then able to:
 - Submit a unsigned transaction with a signed payload
 - Submit a signed transaction

## AppCrypto对象
 A type binding runtime-level `Public/Signature` pair with crypto wrapped by `RuntimeAppPublic`.

 Implementations of this trait should specify the app-specific public/signature types.
 This is merely a wrapper around an existing `RuntimeAppPublic` type, but with
 extra non-application-specific crypto type that is being wrapped (e.g. `sr25519`, `ed25519`).
 This is needed to later on convert into runtime-specific `Public` key, which might support
 multiple different crypto.
 The point of this trait is to be able to easily convert between `RuntimeAppPublic`, the wrapped
 (generic = non application-specific) crypto types and the `Public` type required by the runtime.

 TODO [#5662] Potentially use `IsWrappedBy` types, or find some other way to make it easy to
 obtain unwrapped crypto (and wrap it back).

	Example (pseudo-)implementation:
 ```ignore
	// im-online specific crypto
 type RuntimeAppPublic = ImOnline(sr25519::Public);
 // wrapped "raw" crypto
 type GenericPublic = sr25519::Public;
 type GenericSignature = sr25519::Signature;

 // runtime-specific public key
 type Public = MultiSigner: From<sr25519::Public>;
 type Signature = MulitSignature: From<sr25519::Signature>;
 ```

 # client/offchain/src/lib.rs
 Substrate离线工作器

 离线工作器是特殊的运行时函数，在每个区块被导入的时候执行。在执行时，它具有异步提交外源（交易），作为未签名的交易传播给其他节点的能力。

 离线工作器可以用来执行不适合在正常的区块处理过程中使用执行的重计算任务。它可以是不需要共适的任务，也可以是某些基于链上数据所构建的共识形式，如：
 1. 对于不正确计算结果的挑战过程
 2. 对于结果的主要投票过程
 3. 等等


# frame/example-offchain-worker/src/lib.rs
## 离线工作器例程模块