# 说明
future.rs和ready.rs分别定义了未来态和就绪态交易队列


* future.rs ：这个文件实现了两个类，`WaitingTransaction`和`FutureTransactions`，前者对Transaction的依赖tag进行管理，后者对前者进行管理。
* ready.rs : 这个文件定义了两个类，`ReadyTx`和`ReadyTransactions`，前者描述了一个已经在就绪的交易，后者是对前面的管理
 
## WaitingTransaction
记录了一个交易本身，该交易进入就绪态所缺失的tags，以及交易的生成时间（区块时间）
```rust
#[cfg_attr(not(target_os = "unknown"), derive(parity_util_mem::MallocSizeOf))]
/// Transaction with partially satisfied dependencies.
pub struct WaitingTransaction<Hash, Ex> {
    /// Transaction details.
    pub transaction: Arc<Transaction<Hash, Ex>>,
    /// Tags that are required and have not been satisfied yet by other transactions in the pool.
    pub missing_tags: HashSet<Tag>,
    /// Time of import to the Future Queue.
    pub imported_at: Instant,
}
```
## ReadyTx
```rust
#[derive(Debug, parity_util_mem::MallocSizeOf)]
pub struct ReadyTx<Hash, Ex> {
    /// 交易的引用
    pub transaction: TransactionRef<Hash, Ex>,
    /// 这个交易将会可以解锁多少个其他的交易
    pub unlocks: Vec<Hash>,
    /// How many required tags are provided inherently
    ///
    /// 有些交易有可能已经从队列中剪裁了，
    /// 这个值用来设计要从前面多少个开始设置
    pub requires_offset: usize,
}
```
#### 创建
```rust
pub fn new(
        transaction: Transaction<Hash, Ex>,
        provided: &HashMap<Tag, Hash>,
        recently_pruned: &[HashSet<Tag>],
    ) -> Self {
```
* transaction： 交易本身
* provided: 目前已经提供的tags
* recently_pruned: 最近被丢弃（用于检查交易过期导入的交易）

#### 新tag到达
`pub fn satisfy_tag(&mut self, tag: &Tag) {`    
标识某个tag已经到达

#### 检查该交易是否就绪
`pub fn is_ready(&self) -> bool {`   
当missin_tags为空时，该交易就是就绪的 

### FutureTransactions
管理了一组WaitingTransaction以及他们依赖的tags
```rust
#[derive(Debug)]
#[cfg_attr(not(target_os = "unknown"), derive(parity_util_mem::MallocSizeOf))]
pub struct FutureTransactions<Hash: hash::Hash + Eq, Ex> {
    /// 所有的还没有提供的tags以及他们对应的交易表
    wanted_tags: HashMap<Tag, HashSet<Hash>>,
    /// 交易及依赖该交易的其他交易
    waiting: HashMap<Hash, WaitingTransaction<Hash, Ex>>,
}

```

#### 导入交易
`pub fn import(&mut self, tx: WaitingTransaction<Hash, Ex>) {`  
导入一个交易到Future队列，只有那些依赖tag还没有完全满足的交易才会进入到Future队列。一旦依赖的tag被满足，这个交易应该立刻从队列中移除，并且转移到Ready队列

#### 检查交易是否在队列中
`pub fn contains(&self, hash: &Hash) -> bool {`

#### 根据哈希获得对应的交易
`pub fn by_hashes(&self, hashes: &[Hash]) -> Vec<Option<Arc<Transaction<Hash, Ex>>>> {`
注意返回的是Vec<Option<...>>

#### tags满足
`pub fn satisfy_tags<T: AsRef<Tag>>(&mut self, tags: impl IntoIterator<Item=T>) -> Vec<WaitingTransaction<Hash, Ex>> {`
通知这些tags已经被提供了，返回（并且已经从队列中删除）成为就绪态的交易

#### 删除交易
`pub fn remove(&mut self, hashes: &[Hash]) -> Vec<Arc<Transaction<Hash, Ex>>> {`  
从队列中删除交易

#### 折叠/合并
`pub fn fold<R, F: FnMut(Option<R>, &WaitingTransaction<Hash, Ex>) -> Option<R>>(&mut self, f: F) -> Option<R> {`
使用f函数来对所有的交易计算并且得到最终结果。

#### 查询所有交易
`pub fn all(&self) -> impl Iterator<Item=&Transaction<Hash, Ex>> {`
返回队列中的所有交易的迭代器

#### 清除所有交易
`pub fn clear(&mut self) -> Vec<Arc<Transaction<Hash, Ex>>> {`

#### 交易个数
`pub fn len(&self) -> usize {`
返回交易的个数

#### 交易编码长度
`pub fn bytes(&self) -> usize {`
返回队列中所有交易的编码后的长度
