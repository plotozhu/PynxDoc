
## listener.rs
交易和交易池的事件监听实现。
### Listener
```rust
pub struct Listener<H: hash::Hash + Eq, C: ChainApi> {
    watchers: HashMap<H, watcher::Sender<H, BlockHash<C>>>,
    finality_watchers: LinkedHashMap<BlockHash<C>, Vec<H>>,
}
```
* watchers 交易/区块的事件接收器，由于其key值是哈希，因此watcher可以根据实现的接口，注册到任何交易或是区块事件上。目前的实现只是交易
* finality_watchers 区块的最终确定化事件接收器。
* 
#### finality_watchers理解
`finality_watchers`是一个自动化的区块最终确定化的监视器，用于跟踪区块的最终确定化情况，与之相关的有三个函数
* pruned    ：当一个新的区块产生时，通过此函数通知finality_watchers，注意这个动作将会被根据打包的交易数调用多次，这个将产生几个动作：
    1. 通知交易被包含到了某个block
    2. 如果finality_watchers中没有这个哈希，那么就生成一个
    3. 如果总的finality_watchers的长度超过了MAX_FINALITY_WATCHERS（默认512个），那么把所有在这个之前的哈希对应的区块中的交易都标记为finality_timeout，并且通知对应的watcher
* retracted: 当一个区块在回滚中被撤销时，使用这个函数通知wather
* finalized: 当一个区块被最终确定化时，使用这个函数通知watcher


#### 事件生成
`fn fire<F>(&mut self, hash: &H, fun: F) where F: FnOnce(&mut watcher::Sender<H, BlockHash<C>>) {`
这个是内部的事件生成，然后发送消息给watcher的函数

#### 创建watcher
`pub fn create_watcher(&mut self, hash: H) -> watcher::Watcher<H, BlockHash<C>> {`
创建一个watcher

#### 交易已经广播事件
`pub fn broadcasted(&mut self, hash: &H, peers: Vec<String>) {`
产生一个交易已经广播事件，并且发送给对应的watcher

#### 交易进入就绪态
`pub fn ready(&mut self, tx: &H, old: Option<&H>) {`
产生交易进入就绪态事件，并且发送给对应的watcher

#### 交易进入未来态
`pub fn future(&mut self, tx: &H) {`
产生交易进入未来态事件，并且发送给对应的watcher

#### 交易被丢弃
`pub fn dropped(&mut self, tx: &H, by: Option<&H>) {`
当交易因为限制条件而被丢弃时发生，产生丢弃事件，并且发送给对应的watcher

#### 交易无效
`pub fn invalid(&mut self, tx: &H, warn: bool) {`
交易因为无效而被移除时发生，事件发送给对应的watcher

## pool.rs
首先定义了ChainApi的特征  
然后定义了一个Pool类型，这个类型中只有一个属性，就是validated_pool

