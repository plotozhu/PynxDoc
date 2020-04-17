# 说明
digest是可放置在区块头上的摘要信息，其结构就是一个DigestItem的数据组
```rust
/// Generic header digest.
#[derive(PartialEq, Eq, Clone, Encode, Decode, RuntimeDebug)]
#[cfg_attr(feature = "std", derive(Serialize, Deserialize, parity_util_mem::MallocSizeOf))]
pub struct Digest<Hash: Encode + Decode> {
 A list of logs in the digest.
    pub logs: Vec<DigestItem<Hash>>,
}
```
Digest类提供一些标准的接口：
* logs:             返回当前的记录
* push:             向Digest中境加一条记录
* pop ：            从Digest中弹出
* log ：            通过某个predicate函数来搜索特定的
* convert_first:    使用某个predicate函数来搜索并且得出转换后的值（其实与log的实现过程完全一样）

# DigestItem对象
DigestItem对象用来编码/解码系统的摘要信息，并且互相之间不透明。

DigestItem被分成了6种：
1. `ChangesTrieRoot(Hash),`: 系统的"[更新字典](../../core/src/changes_tries.md)"，包含了给定区块"更新字典"状态根，如果runtime支持创建"更新字典"，这个摘要将会在每个区块上都创建。
2. `PreRuntime(ConsensusEngineId, Vec<u8>)`：runtime前摘要， 这个是共识引擎到runtime的摘要信息，尽管共识引擎可以（并且应该）自行读取数据来避免代码和状态重复。虽然只有在错误情况下runtime才会产生这些数据，但是runtime时并不检查这些数据。  
**注意：** 即使在`on_initialize`调用时，某个期望的'PreRuntime`不存在，runtime也不允许panic或是失败。应该由外部的区块验证器来检查这些。 Runtime API调用将会不使用pre-runtime的摘要来初始化区块，因此当他们不存在的时候，初始化也不会失败。
3. `Consensus(ConsensusEngineId, Vec<u8>)`：从Runtime传递给共识引擎的消息，这个**永远**不应该由任何共识引擎的本地代码生成，但是并不作检查（是否是由本地代码生成）。
4. `Seal(ConsensusEngineId, Vec<u8>)`：封包信息，总是由本地代码生成，runtime从来看不到这个。
5. `ChangesTrieSignal(ChangesTrieSignal)`：包含了从tries管理器到本地代码的状态变化消息的摘要。
6. `Other(Vec<u8>)`

### ChangesTrieSignal

```rust
#[derive(PartialEq, Eq, Clone, Encode, Decode)]
#[cfg_attr(feature = "std", derive(Debug, parity_util_mem::MallocSizeOf))]
pub enum ChangesTrieSignal {
    NewConfiguration(Option<ChangesTrieConfiguration>),
}
```
* NewConfiguration 说明  
 从 **下个区块** 开始，创建新的"更新字典"树、
 产生该信号的区块应该包含"更新字典"，这个字典覆盖了：  
 区块范围为： [BEGIN; current block], 此处的 BEGIN 是 (依次):  
 - LAST_TOP_LEVEL_DIGEST_BLOCK+1 如果顶层的字典树摘要从来没有生成过。 使用当前的配置 **以及** 最近的在LAST_TOP_LEVEL_DIGEST_BLOCK 创建的顶层字典树;
 - LAST_CONFIGURATION_CHANGE_BLOCK+1 如果以前有CT（更新字典树）的配置变化，并且最后的变化发生在区块LAST_CONFIGURATION_CHANGE_BLOCK;
 - 1 其他情况.

**非常重要** 需要配置成每个Epoch应该正好可以容纳一个或多个，即分片后的第一个区块正好应该是某个"更新字典树"的开始的区块。

### OpaqueDigestItemId
包含了原始数据的摘要类型；这个同样命名了可执行的共识引擎ID，可以用来标识感兴趣的一个或多个摘要。
```rust
#[derive(Copy, Clone, Eq, PartialEq, Ord, PartialOrd)]
pub enum OpaqueDigestItemId<'a> {
    /// Type corresponding to DigestItem::PreRuntime.
    PreRuntime(&'a ConsensusEngineId),
    /// Type corresponding to DigestItem::Consensus.
    Consensus(&'a ConsensusEngineId),
    /// Type corresponding to DigestItem::Seal.
    Seal(&'a ConsensusEngineId),
    /// Some other (non-prescribed) type.
    Other,
}
```

### DigestItem的接口
#### `pub fn dref<'a>(&'a self) -> DigestItemRef<'a, Hash> {`
#### `pub fn as_changes_trie_root(&self) -> Option<&Hash> {`
#### `pub fn as_pre_runtime(&self) -> Option<(ConsensusEngineId, &[u8])> {`
### `pub fn as_consensus(&self) -> Option<(ConsensusEngineId, &[u8])> {`

#### `pub fn as_seal(&self) -> Option<(ConsensusEngineId, &[u8])> {`
#### `pub fn as_changes_trie_signal(&self) -> Option<&ChangesTrieSignal> {`
#### `pub fn as_other(&self) -> Option<&[u8]> {`
#### `pub fn try_as_raw(&self, id: OpaqueDigestItemId) -> Option<&[u8]> {`
#### `pub fn try_to<T: Decode>(&self, id: OpaqueDigestItemId) -> Option<T> {`

### 其他
DigestItem同样实现了Encode和Decode特质
### DigestItemRef接口
与DigestItem的接口一致，实现了deref后同样的处理效果
#### `pub fn as_changes_trie_root(&self) -> Option<&'a Hash> {`
