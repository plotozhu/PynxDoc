
BasicPool创建了一个RevalidateQueue，这个队列创建了一个后台的RevalidationWorker。
<span id="basic_pool">BasicPool用法之一：</span>
1. 实现`MaintainedTransactionPool`,监控链的NewBlock和Finalized两个事件，NewBlock事件时，调用：
   1. ready_poll处理需要在本区块处理的交易
   2. `revalidation_queue.revalidate_later`清除不需要的交易
## 基本类
### `BasicPool`
基本的交易池,里面包括五个属性字段：
```rust
{
    pool: Arc<sc_transaction_graph::Pool<PoolApi>>,
    api: Arc<PoolApi>,
    revalidation_strategy: Arc<Mutex<RevalidationStrategy<NumberFor<Block>>>>,
    revalidation_queue: Arc<revalidation::RevalidationQueue<PoolApi>>,
    ready_poll: Arc<Mutex<ReadyPoll<ReadyIteratorFor<PoolApi>, Block>>>,
} 
```
pool：实际存储交易的交易池
api： 一个PoolApi对象,用于对外供api接口
revalidation_strategy： 重新验证的策略
revalidation_queue：重新验证的队列（详情见[Revalidation](#repla)。
ready_poll：一个可以被多个线程调用的`ReadyPoll`对象

BasicPool提供了如下的接口
#### 创建

```rust 
pub fn new(
        options: sc_transaction_graph::Options,
        pool_api: Arc<PoolApi>,
    ) -> (Self, Option<Pin<Box<dyn Future<Output=()> + Send>>>) {
```
创建一个BasicPool，其中:  
* <span id="options" > `options` </span>是Pool的配置，包括三种：
  * `ready` 限制ready队列
  * `future` 限制future队列
  * `reject_future_transactions`拒绝future交易  
* `pool_api`是一个PoolApi的对象     

new实际使用了[使用验证方案来创建](#span-id%22validate%22%e4%bd%bf%e7%94%a8%e9%aa%8c%e8%af%81%e6%96%b9%e6%a1%88%e6%9d%a5%e5%88%9b%e5%bb%baspan)创建了实际的对象
#### <span id="validate">使用验证方案来创建</span>

```rust 
    pub fn with_revalidation_type(
        options: sc_transaction_graph::Options,
        pool_api: Arc<PoolApi>,
        revalidation_type: RevalidationType,
    ) -> (Self, Option<Pin<Box<dyn Future<Output=()> + Send>>>) {
```
* `options`和`pool_api`的说明见[new中的options](#options)
* `revalidation_type`有两种，`Light`和`Full`

如果是Light，那么使用`new`创建一个`RevalidationQueue`，如果是Full，那么使用`new_background`创建一个`RevalidationQueue`和`background`线程，这个线程每200ms执行一次，详细执行见[revalidation](#span-id%22revalidation%22-revalidatetionspan)。
然后使用返回的`RevalidationQueue`和输入的参数来创建BasicPool对象。
#### 查询pool对象
```rust
    pub fn pool(&self) -> &Arc<sc_transaction_graph::Pool<PoolApi>> {
        &self.pool
    }
```

### `RevalidationQueue`
实现再激活功能的队列，具有三个属性：
```rust
pub struct RevalidationQueue<Api: ChainApi> {
    pool: Arc<Pool<Api>>,
    api: Arc<Api>,
    background: Option<mpsc::UnboundedSender<WorkerPayload<Api>>>,
}
```
* pool 交易池
* api  提供交易处理的api
* background 如果启动了后台线程，这个就是与后台线程通信的接口

#### 创建
创建有三种方法：
1. `new(api: Arc<Api>, pool: Arc<Pool<Api>>) -> Self {` 创建了一个默认的，没有自动工作后台线程的队列
2. `pub fn new_with_interval<R: intervalier::IntoStream>(
        api: Arc<Api>,
        pool: Arc<Pool<Api>>,
        interval: R,
    ) -> (Self, Pin<Box<dyn Future<Output=()> + Send>>)` 创建一个具有后台线程，并且指定每interval与后台交互一次的队列
3. `pub fn new_background(api: Arc<Api>, pool: Arc<Pool<Api>>) ->
        (Self, Pin<Box<dyn Future<Output=()> + Send>>)` 创建一个具有后台线程，每默认时间（200ms)与后台交互一次的队列
#### 延时激活
异步的，对一组交易实现再激活的函数，如果：
1. 队列有后台再激活线程，直接返回（再
2. 队列如果没有再激少后台线程，等到实际工作执行后再返回
`fn revalidate_later(&self, at: NumberFor<Api>, transactions: Vec<ExHash<Api>>) {`
注意，参数中的transactions是交易的哈希值，这意味着这个函数是对交易池内现有的交易进行再激活处理
### `RevalidationWorker`
异步验证交易的的工作线程，实现了可以直接或是放在后台进程中使用的的future
```rust
struct RevalidationWorker<Api: ChainApi> {
    api: Arc<Api>,
    pool: Arc<Pool<Api>>,
    best_block: NumberFor<Api>,
    block_ordered: BTreeMap<NumberFor<Api>, HashSet<ExHash<Api>>>,
    members: HashMap<ExHash<Api>, NumberFor<Api>>,
}
```
* api是chainapi的对象
* pool是交易池
* best_block是当前最好的区块
* block_ordered：根据每个区块高度放置的交易信息
* members：根据交易哈希查询应该在哪个高度有效

## 辅助类
### TransactionPool特质
BasicPool实现了TransactionPool特质。
#### submit_at
```rust
fn submit_at(
        &self,
        at: &BlockId<Self::Block>,
        source: TransactionSource,
        xts: Vec<TransactionFor<Self>>,
    ) -> PoolFuture<Vec<Result<TxHash<Self>, Self::Error>>, Self::Error> {
```

####  submit_one
```rust
fn submit_one(
        &self,
        at: &BlockId<Self::Block>,
        source: TransactionSource,
        xt: TransactionFor<Self>,
    ) -> PoolFuture<TxHash<Self>, Self::Error> {
```
#### submit_and_watch 

### `ReadyPoll`
`ReadyPoll`记录了在某个区块上需要查询数据的`poller`，其结构如下：
```rust 
struct ReadyPoll<T, Block: BlockT> {
    updated_at: NumberFor<Block>,
    pollers: Vec<(NumberFor<Block>, oneshot::Sender<T>)>,
}
```
* `update_at`是记录当前对象在哪个区块上被执行了，执行是指trigger函数被调用了；
* `pollers`记录了在哪个区块上需要执行，然后执行后的数据接收者。

`ReadyPoll`对外提供三个接口：
1. `trigger(&mut self, number: NumberFor<Block>, iterator_factory: impl Fn() -> T)`:这是一个执行函数，在某个区块上可以通过执行这个函数来对需要在number区块以前的所有的poller都执行一次清理。
2. `add(&mut self, number: NumberFor<Block>) -> oneshot::Receiver<T>` 向ReadyPoll对象添加一个poller
3. `updated_at(&self) -> NumberFor<Block> `这个ReadPoll的最新执行的区块高度

### `RevalidationType`
### <span id="revalidation"> `revalidatetion`</span>
`revalidation.rs`文件里是与验证有关的特质和结构类型：
#### `WorkerPayload`
这个表征了工作线程的工作任务
```rust
struct WorkerPayload<Api: ChainApi> {
    at: NumberFor<Api>,
    transactions: Vec<ExHash<Api>>,
}
```
* `at`是需要在哪个区块上进行执行
* transactions是在该区块上需要重新validation的交易



#### 创建
#### 批量再验证
根据链的情况，对pool中所有的交易进行验证，无效的移除，有用的重新放进交易池
```rust
async fn batch_revalidate<Api: ChainApi>(
    pool: Arc<Pool<Api>>,
    api: Arc<Api>,
    at: NumberFor<Api>,
    batch: impl IntoIterator<Item=ExHash<Api>>,
) 
```
* pool是交易池
* api是提供验证接口的api
* at 是区块高度
* batch 是一组待再验证的交易哈希


## 实现的特质
BasicPool实现了两个最主要的特质：
* `TransactionPool` 最基本的交易池的接口实现
  * `submit_at`  提交一组在某个区块后生效的的交易
  * `submit_one` 提交一个在某个区块后生效的的交易
  * `submit_and_watch` 提交一个在某个区块后生效的的交易，并且返回一个监视器
  * `remove_invalid` 清除当前交易池内无效的交易，返回这些被清除的交易
  * `status`  返回交易池的状态
  * `import_notification_stream` 注册一个交易被导入到交易池时的通知事件
  * `hash_of`  根据交易查询对应的哈希
  * `on_broadcasted` 通知交易池，某些交易已经被传播（交易池中需要更新交易的状态，是否被传播过也是一种痒）
  * `ready_transaction`  某交易是否已经就绪，如果未就续，返回None
  * `ready_at` 这个将返回一个future,在某个区块时有效的交易
  * `ready` 返回当前区块时，已经就绪的交易
* `MaintainedTransactionPool`这个特质自动处理区块的两个事件，NewBlock区块和Finalized事件，在事件到达时，对交易池作一些处理（见[BasicPool用法之一](#basicpool))的说明

