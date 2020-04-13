# 概要说明
Service将系统中的Client/Network/Transaction pool/后端存储/keystore等各个子组件聚合。

ServiceBuilder在创建对象后，还可以通过with_xxx函数补充配置信息。  
Service中通过ServiceBuilder.build创建各个模块，并且配置模埠的依赖关系。其步骤如下：
1. 创建session_keys
2. 创建关键任务失败的消息通道
3. 创建import_queue、chain_info、chain_spec和版本
4. 向telemetry汇报信息
5. 为execution_extensions注册transaction_poll了
6. 创建transaction_pool_adapter
7. 生成protocol_id
8. 使用client创建一个block_validator
9. 用roles,executor,network_config,chain,finality_proof_provider, finality_proof_request_builder,on_demand, transaction_pool, import_queue,protocol_id,block_announce_validator, metrics_registry作为网络参数来创建一个NetworkWorker
10. 从NetworkWorker取出NetworkService这个对象，可以向这个对象发送网络操作命令
11. 创建一个network_status_sinks，用来接收网络消息事件
12. 创建一个offchain_storage并且使用这个对象和client创建offchain_workers
13. 启动所有的后台任务（background_task）
14. 利用client中的创建import_stream和finality_stream，这个是实现NewBlock和FinalizedBlock消息的通知，并且用futures::stream::select来创建一个独立的任务线程？（这个的作用未明）
15. 创建一个线程，对transaction_pool中的每个哈希在导入的时候，使用网络层进行传播propagate_extrinsic
16. 配置Prometheus统计工具
17. 创建状态通道，并且以每5秒钟一次，将数据信息推向此状态通道，然后把数据通过telemetry进行统计
18. 创建状态通道，以30秒每次的频率network_status_sinks
19. 创建rpc使用的通道，以及rpc_handler
20. 创建一个线程，通过build_network_future来对网络服务进行

# 分析
## session_key
验证者使用会话密钥来签署与共识相关的消息。SessionKeys是在运行时中具体化的通用可索引类型。

您可以声明任意数量的会话密钥。例如，默认的"基板"节点使用四个。其他链可能有更多或更少，具体取决于链希望其验证程序执行的操作。

在实践中，验证器将所有会话公共密钥合并为一个对象，使用"控制器"帐户对公共密钥集进行签名，并提交事务以在链上注册密钥。该链上注册将验证者节点与拥有资金的帐户链接起来。这样，该帐户可以根据节点的行为获得奖励或大幅削减。

运行时声明将包含哪些会话密钥（runtime/src/lib.rs）：

## TransactionPoolAdapter
TransactionPoolAdapter实现pool和client的关联，client的主要作用是在adapter的函数中，获取当前的block_number，用作verified验证时使用。 个人理解是指这个在当前最近的block_number之后，这个transaction是否有效，此处的有效的概念最简单的就是nonce是否大于在这个区块后，该帐户存储的最近发送的交易的nonce值。

有了这个判定后，transactionPoolAdater就可以对外提供交易导入、交易广播、交易到达等行为了，所有的交易经过验证后才会进行其他的处理。