# 说明
validated-pool里面保存经过验证的交易，经过验证的交易以下面的方式存储：
```rust
#[derive(Debug)]
pub enum ValidatedTransaction<Hash, Ex, Error> {
    /// Transaction that has been validated successfully.
    Valid(base::Transaction<Hash, Ex>),
    /// Transaction that is invalid.
    Invalid(Hash, Error),
    /// Transaction which validity can't be determined.
    ///
    /// We're notifying watchers about failure, if 'unknown' transaction is submitted.
    Unknown(Hash, Error),
}
```

经过验证的交易有三种类型：
* Valid 合法的
* Invalid 不合法的
* Unknown 未知的，无法确定的

# ValidatedTransaction 特征
##  验证
```rust 
pub fn valid_at(
        at: u64,
        hash: Hash,
        source: TransactionSource,
        data: Ex,
        bytes: usize,
        validity: ValidTransaction,
    ) -> Self {
```
//TODO 只是简单包装了一下，需要回头研究

# ValidatedPool
存放验证过后交易的交易池
```rust /// Pool that deals with validated transactions.
pub struct ValidatedPool<B: ChainApi> {
    api: Arc<B>,
    options: Options,
    listener: RwLock<Listener<ExHash<B>, B>>,
    pool: RwLock<base::BasePool<
        ExHash<B>,
        ExtrinsicFor<B>,
    >>,
    import_notification_sinks: Mutex<Vec<mpsc::UnboundedSender<ExHash<B>>>>,
    rotator: PoolRotator<ExHash<B>>,
}
```   
* api : ChainApi接口
* options:                      交易池的限制策略，当交易数据过多时的处理方法
* listener:                     交易信息监听器
* pool:                         实际存放交易的BasePool类型
* import_notification_sinks：   接收导入消息的接收器
* rotator:  交易处理器，负责对过期的交易进行删除


## 创建交易池
`pub fn new(options: Options, api: Arc<B>) -> Self {`

## 屏蔽某些交易
`pub fn ban(&self, now: &Instant, hashes: impl IntoIterator<Item=ExHash<B>>) {`

## 检查交易的屏蔽状态
`pub fn is_banned(&self, hash: &ExHash<B>) -> bool {`

## 提交一组交易
` pub fn submit<T>(&self, txs: T) -> Vec<Result<ExHash<B>, B::Error>> where
        T: IntoIterator<Item=ValidatedTransactionFor<B>>
    { `

## 提交一个交易
`fn submit_one(&self, tx: ValidatedTransactionFor<B>) -> Result<ExHash<B>, B::Error> {`

## 执行交易池限制
`fn enforce_limits(&self) -> HashSet<ExHash<B>> {`

## 提交并且跟踪该交易
`pub fn submit_and_watch(`

## 重新提交
重新把交易提交给交易池。 删除并且重新将交易提交（重新计算依赖，得到是Ready/Future状态），如果交易池中原来没有，就忽略。
`pub fn resubmit(&self, mut updated_transactions: HashMap<ExHash<B>, ValidatedTransactionFor<B>>) {`

## 读取某些交易提供的tags
`pub fn extrinsics_tags(&self, hashes: &[ExHash<B>]) -> Vec<Option<Vec<Tag>>> {`

## 某个哈希对应的交易是否就绪
`pub fn ready_by_hash(&self, hash: &ExHash<B>) -> Option<TransactionFor<B>> {`

## 依据tags剪裁
依据tags，将交易剪裁（该提升的提升，该丢弃的丢弃）
`pub fn prune_tags(
        &self,
        tags: impl IntoIterator<Item=Tag>,
    ) -> Result<PruneStatus<ExHash<B>, ExtrinsicFor<B>>, B::Error> {`

## 重新提交剪裁过的交易
重新提交经过prune_tags调用所剪裁过的交易（这个应该是在区块被撤回后的动作）
```rust
pub fn resubmit_pruned(
        &self,
        at: &BlockId<B::Block>,
        known_imported_hashes: impl IntoIterator<Item=ExHash<B>> + Clone,
        pruned_hashes: Vec<ExHash<B>>,
        pruned_xts: Vec<ValidatedTransactionFor<B>>,
    ) -> Result<(), B::Error> {
```

## 为剪裁动作产生通知事件

```rust
    pub fn fire_pruned(
        &self,
        at: &BlockId<B::Block>,
        hashes: impl Iterator<Item=ExHash<B>>,
    ) -> Result<(), B::Error> {
```

## 清除过期的交易
从交易池中清除过期的交易。 过期的交易是指超出寿命的交易，注意该函数并不移除那些已经包含在链中的交易，如果您想要移除它们，请参阅[`prune_tags`]()
`pub fn clear_stale(&self, at: &BlockId<B::Block>) -> Result<(), B::Error> {`

## 生成导入通知的事件流：
`pub fn import_notification_stream(&self) -> EventStream<ExHash<B>> {`

## 生成已经广播的事件
`pub fn on_broadcasted(&self, propagated: HashMap<ExHash<B>, Vec<String>>) {`

## 移除无效的交易
从交易池中移除交易和这些交易的子树（依赖这些交易的交易，并且把他们标志为无效的）。传入参数中的交易将会被屏蔽一段时间，以防止他们又立刻被re-import。注意这些依存交易（子树）并不会被屏蔽，因为他们可能会仍然有效，我们可能会重新导入他们。 即交易及依赖这些交易的子树不是强绑定的，交易无效时，其子树交易可能是仍然有效，明显的例子就是多个交易产生一个tag，那么子树只是依赖这个交易需要被先执行而已。  
`pub fn remove_invalid(&self, hashes: &[ExHash<B>]) -> Vec<TransactionFor<B>> {`

## 查询就绪的交易、状态值

## 区块最终决定化消息能知
通知所有的watcher，某个区块已经被最终确定化了：

`pub async fn on_block_finalized(&self, block_hash: BlockHash<B>) -> Result<(), B::Error> { `

## 区块回撤通知
通知所有的listener，某个区块被回撤了
`pub fn on_block_retracted(&self, block_hash: BlockHash<B>) {`