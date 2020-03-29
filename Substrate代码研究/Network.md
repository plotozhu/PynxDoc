## 理解
此片的network网络指的是区块链网络，而不仅仅是传输网络
## NetworkWorker::new
1. 创建消息接收队列
2. 读取配置
3. 分析bootnode
4. 删除重复的bootnode
5. 初始化 保留的节点
6. 配置peerset
7. 分析本机的帐号和公私钥
8. 创建检查器？
9. 使用Protocol::new创建新的协议
10. 创建swarm，这个是libp2p的 swarm  
  new函数使用Params类型参数来创建，Params是如下的格式
```rust
pub struct Params<B: BlockT, H: ExHashT> {
       /// 角色 (全节点, 轻节点, ...).
       pub roles: Roles,
       /// 这个executor是如何生成后台任务的，如果是`None`，使用默认的
       pub executor: Option<Box<dyn Fn(Pin<Box<dyn Future<Output = ()> + Send>>) + Send>>,
       /// 这个才是网络层的配置
       pub network_config: NetworkConfiguration,
       /// 处理链的客户端
       pub chain: Arc<dyn Client<B>>,
       ///最终确定化的证明
       ///这是一个`Some()`对象，如果有，将使用他来执行最终确定化
       pub finality_proof_provider: Option<Arc<dyn FinalityProofProvider<B>>>,
       /// 这个是用来确认什么样的区块需要执行最终确定化.
       ////这是一个`Some()`对象，如果有，将使用他来决定是否需要最终确定化
       pub finality_proof_request_builder: Option<BoxFinalityProofRequestBuilder<B>>,
       /// `OnDemand` 对象作为一个从客户端获取区块数据的接收器
       /// 如果是`Some`对象,网络工作器将会处理这些请求并且返回给他们，一般来说是轻节点用的
       pub on_demand: Option<Arc<OnDemand<B>>>,
       /// 交易池，网络工作器将从从对象读取交易向网络中广播
       pub transaction_pool: Arc<dyn TransactionPool<H, B>>,
       /// 在连接上使用的协议的名字，每个链都应该不一样
       pub protocol_id: ProtocolId,
       /// 导入需要使用的（区块）队列.
       /// 导入队列用于验证从其他节点收到的区块是否有效
       pub import_queue: Box<dyn ImportQueue<B>>,
       /// 验证收到的新区块消息
       pub block_announce_validator: Box<dyn BlockAnnounceValidator<B> + Send>,
       /// 注册prometheus测量信息
       pub metrics_registry: Option<Registry>,
}

```
这其中的`network_config`是区块链的网络配置
```rust
pub struct NetworkConfiguration {
    ///配置文件的路径
    pub config_path: Option<PathBuf>,
    /// 网络相关的配置文件的路径
    pub net_config_path: Option<PathBuf>,
    /// 监听地址（可能有多个通道，因此有多个地址）
    pub listen_addresses: Vec<Multiaddr>,
    /// 广播的自身公共地址（可用于连接的），如果是空，就自动检测
    pub public_addresses: Vec<Multiaddr>,
    /// List of initial node addresses
    pub boot_nodes: Vec<String>,
    ///节点的网络地址
    pub node_key: NodeKeyConfig,
    /// 最大连入节点数
    pub in_peers: u32,
    /// 最大连出节点数
    pub out_peers: u32,
    /// 保留地址
    pub reserved_nodes: Vec<String>,
    /// The non-reserved peer mode.
    pub non_reserved_mode: NonReservedPeerMode,
    /// 哨兵节点（应该不需要使用）
    pub sentry_nodes: Vec<String>,
    /// 客户端版本号
    pub client_version: String,
    /// 节点名称，通过连接发送，可用于调试
    pub node_name: String,
    ///传输层配置
    pub transport: TransportConfig,
    /// Maximum number of peers to ask the same blocks in parallel.
    pub max_parallel_downloads: u32,
}
```
网络节点对象是通过`Protocol::new`创建的，在调用时，传入了一个peersetConfig的参数，创建一个peerset和peersetHandle，peerset在单独的线程里运行，返回一个peersetHandle用于管理peerset。

## 网络数据收发的流程
`client/service/src/builder.rs`中创建了一个线程，执行`build_network_future`，使用poll_fn闭包中执行`sc_network::NetworkWorker<B, H>`来实现信息网络状态的更新  
KAD网络被用于网络发现，因此在`Discovery`中进行，包装成了一个`DiscoveryBehaviour`, 而在标准的`Behaviour`中，把DiscoveryBehaviour对象的一个实例存储到了内部。在创建Swarm对象的函数`Swarm.Build`中，需要使用此`Behaviour`对象作为参数传入，由此方式实现了KAD网络

目前的实现中，  SendToPeer是否可以实现端点对端点发送？ 
* 分析结果，只发给邻节点，如果邻节点没有连接，直接丢弃返回了

## network::src::protocol::generic_proto::behaviour.rs
这是通用的协议处理，以下是代码内文档的翻译  
Network behaviour 处理与其他节点连接的自定义协议的子流

### 传统与新协议
`GenericProto`有以下的行为方式
- 一旦建立一个连接，我们打开一个子流（在源代码里称为`legacy protocol`）。这个子流的名称由初化时传入的`protocol_id`和`version`决定。如果远端拒绝此子流，我们就关闭这个连接
- 对于每一个已经注册的协议，我们同样需要打开为此协议打开一个子流，即使对方拒绝此子流，同样是可以的（忽略，并不需要关闭连接）。
- 当我们需要发送消息时，我们可以调用`send_packet`来强制向该子流发送，或者使用`write_notification`来提示一个已经注册的协议（子流），如果对端拒绝或是不支持该协议，就使用传统的方式（`send_packet`）发送。

### 如何工作
`GenericProto`角色用来同步以下几个组件
- libp2p的swarm：打开新连接，通知连接断开
- 连接句柄（查阅`handler.rs`)：处理独立的连接
- 连接集管理器（PSM）：请求连接或断开节点
- 扩展API： 给需要了解已经建立连端的外部客户代码

每个连接句柄可以处于4种状态： 使能+打开、使能+关闭、禁用+打开以及禁用+关闭，使能/禁用必须与PSM同步，例如，如果PSM要求断开连接，我们需要在此禁用此连接。打开/关闭需要与扩展API同步。  
然而连接句柄只有当我们实际与某个节点连接后才会存在。这意味着对于每个节点，我们可能有六个状态，断开、拨号中、使能+打开、使能+关闭、禁用+打开以及禁用+关闭，需要特别注意的是拨号中状态必须与PSM中的"连接已建立"对应。换句主说，PSM并不区分节点是正在拨号中还是已经连接。  
还有，存在一个"禁止"系统，如果我们拨号某个节点失败，我们将"禁止"该节点几秒钟。如果PSM在此期间内请求拨号该节点，我们将等到"禁止"时间失效后才会执行，但是PSM仍旧认为已经建立连接了。  
注册这个"禁止"是不完全的，如果一个被"禁止"的节点主动拨号连接我们，我们将接受此连接。即"禁止"系统仅仅是禁止拨出。

## 对象`GenericProto`
`GenericProto::new`需要三个参数，协议名称，协议版本和PSM
`GenericProto::register_notif_protocol`可以用来注册协议的通知，就是向`notif_protocol`数组里添加了(proto_name,engine和handshake_msg)的三元组



# 数据收发流程
![](data-flow.png)

需要使用notification的，使用如上的方式进行数据传输。

1. 首先通过register_notificaitons_protocol把自身注册到notification处理器中




## 如果不使用notification方式