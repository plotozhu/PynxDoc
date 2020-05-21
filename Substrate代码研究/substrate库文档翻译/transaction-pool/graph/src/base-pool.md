## base_pool.rs
base_pool.rs是是BasicPool中的pool属性和transaction的实现，在base_pool实现了两个最重要的类：
1. Transaction： 在池中的交易信息
2. BasePool： 注意不是BasicPool，BasicPool在src目录中，主要是对外提供接口，BasePool是实现交易的存储和管理池。

### Transaction
```rust
#[cfg_attr(test, derive(Clone))]
#[derive(PartialEq, Eq, parity_util_mem::MallocSizeOf)]
pub struct Transaction<Hash, Extrinsic> {
    /// 代表了交易的原始的extrinsic
    pub data: Extrinsic,
    /// 交易编码后的大小
    pub bytes: usize,
    /// 交易的唯一哈希
    pub hash: Hash,
    /// 交易的优先级，值越大越高
    pub priority: Priority,
    /// 交易有效的区块
    pub valid_till: Longevity,
    /// 交易依赖的前提Tag
    pub requires: Vec<Tag>,
    /// 这个交易能够提供(所产生的）的tags
    pub provides: Vec<Tag>,
    /// 此交易是否应该被传播（广播到其他节点）//Should that transaction be propagated.
    pub propagate: bool,
    /// 交易的来源（InBlock/Local/External)
    pub source: Source,
}
```
从上述的Transaction定义可以看出，此处定义的交易是有状态的交易，除了原始的交易信息之外，还包括了交易的其他信息。

Transaction其实代表了已经被纳入交易池的交易，因此它实现了InPoolTransaction的特征
* InPoolTransaction特征 
  * data:         返回交易的extrinsic数据
  * hash:         返回交易的哈希值
  * priority:     返回优先级
  * longevity:    返回生命周期（失效的区块高度）
  * requires:     返回依赖条件
  * providers:    返回产生的tags
  * is_propagable:返回是否需要传播

Transaction同时还实现了
* AsRef 特征，可以把Transaction当前一个Extrinsic返回
* Clone 特征，可以实现Transaction的克隆

#### 交易的状态
有效的交易具有两种状态：就绪态和未来态
* 就绪态：交易的所有依赖条件都已经满足，可以被打包进入区块
* 未来态：交易还有一个或多个信赖条件尚未满足，需要等待，还不能被打包进入交易区块。


### BasePool
实现了所有交易的<span id="depend_graph">依赖关系图</span>，并且返回那些已经就绪的交易。 注意，一旦某个函数返回了一些交易，这将意味着重新导入这些交易会失败或者产生一些非期望的结果。在大部情况下，我们需"再激活"它们并且重新计算依赖关系。
```rust
#[derive(Debug)]
#[cfg_attr(not(target_os = "unknown"), derive(parity_util_mem::MallocSizeOf))]
pub struct BasePool<Hash: hash::Hash + Eq, Ex> {
    //是否拒绝未来的交易
    reject_future_transactions: bool,
    //未来的交易
    future: FutureTransactions<Hash, Ex>,
    //已经就绪的交易
    ready: ReadyTransactions<Hash, Ex>,
    /// 保留最近丢弃的tags (最近两次调用所丢弃的).
    ///
    /// 这是为了保证我们不会意外把需要这些tags验证的交易放到future里去。
    recently_pruned: [HashSet<Tag>; RECENTLY_PRUNED_TAGS],
    recently_pruned_index: usize,
}
```
#### 创建
`pub fn new(reject_future_transactions: bool) -> Self {`  
创建一个BasePool

#### 临时允许未来交易
`pub(crate) fn with_futures_enabled<T>(&mut self, closure: impl FnOnce(&mut Self, bool) -> T) -> T {`  
临时使能未来的交易，然后运行closure后，恢复reject_future_transactions，并且将closure的结果返回

#### <span id="import">导入</span>
`pub fn import(
        &mut self,
        tx: Transaction<Hash, Ex>,
    ) -> error::Result<Imported<Hash, Ex>> {`  
把某个交易导入进交易池，注意交易池中包括两个部分：未来的和已就绪。前者包括一些需要依赖尚未被其他交易提供的标签的交易。后者包括所有的条件已经满足的交易，可以被区块包含了。  
导入过程首先使用当前ready和最近被抛弃的tags来对导入的交易运行`WaitingTransaction`，然后判定该交易的状态，如果是ready，进入就绪态，否则进入future态。当然还有一种可能，就是无效的，直接丢弃了。

#### 导入到就绪态
`fn import_to_ready(&mut self, tx: WaitingTransaction<Hash, Ex>) -> error::Result<Imported<Hash, Ex>> {`  
这是非公开的接口，被[导入函数](#span-id%22import%22%e5%af%bc%e5%85%a5span)使用

#### 获取就绪态的交易
`pub fn ready(&self) -> impl Iterator<Item=Arc<Transaction<Hash, Ex>>> `

#### 获取future的交易
`pub fn futures(&self) -> impl Iterator<Item=&Transaction<Hash, Ex>> {`

#### 根据哈希获取交易
`pub fn by_hashes(&self, hashes: &[Hash]) -> Vec<Option<Arc<Transaction<Hash, Ex>>>> {`  
根据提供的哈希返回对应的交易，无论交易处于就绪态还是future态。 注意返回的是一个Vec<Optio<...>>，因此返回值总是与hashes长度一样（每一项要么有交易，要么是个None)

#### 根据哈希获取就绪态的交易
`pub fn ready_by_hash(&self, hash: &Hash) -> Option<Arc<Transaction<Hash, Ex>>> {`
与`by_hashes`的区别就是只获取就绪态下的

#### 交易限制确认
`pub fn enforce_limits(&mut self, ready: &Limit, future: &Limit) -> Vec<Arc<Transaction<Hash, Ex>>> {`  
从交易池的队列中删除最差的交易以及依赖这些最差交易的其他交易。 从技术上说，最差的交易需要通过计算整个pending集才能够得到。此处使用的是最久未被处理的交易。此处的limit可以是两种类型之一：
* 交易个数：限制交易个数的总量
* 编码长度：限制编码后的数据长度

#### 删除交易子树
`pub fn remove_subtree(&mut self, hashes: &[Hash]) -> Vec<Arc<Transaction<Hash, Ex>>> {`  
删除指定哈希所指代的交易以及依赖这些交易的交易。返回被删除的交易，注意：
* 某些交易可能还是有效的，但是他们可能因为是区块的一部分所以被移除了，你可以在后面通过re-import来重新导入
* 如果你想要移除已经被使用的，处于就绪态的交易，并且不想让他们保留在存储池中，请使用`prune_tags`方法
* TODO 需要仔细咀嚼含义

#### 删除所有处于未来态的交易
`pub fn clear_future(&mut self) -> Vec<Arc<Transaction<Hash, Ex>>> {`


#### 修剪tags
`pub fn prune_tags(&mut self, tags: impl IntoIterator<Item=Tag>) -> PruneStatus<Hash, Ex> {`
根据所提供的tags，重新调整交易池中的交易，其实做了三件事：
1. 尝试把所有的依赖这些tags的交易提升到就绪态
2. 在就绪态中删除提供这些tags的交易（自身）
3. 在recently_pruned中记录这些tags
最后返回结果的状态(删除的交易，失败的交易，从future到ready的交易)

#### 交易池状态
`pub fn status(&self) -> PoolStatus {`  
返回交易池的状态 （就续交易的个数，就续交易的编码字节数，未来交易的个数，未来交易的编码字节数）

