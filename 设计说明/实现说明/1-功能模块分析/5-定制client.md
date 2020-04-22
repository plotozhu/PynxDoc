# 说明

需要根据角色定制不同的client，然后将这些client作为网络参数中的一部分，用来生成NetworkWorker。 在现有的版本中，import_queue/ finality_proof_provider/ transaction_pool等参数作为了network_params，而是事实，Networker将network_params中的参数作为参数创建了Protocol。因此，我们又回到这里[protocol处理](./4-protocol处理.md)。
创建三个protocol还是一个？
chain应该是一个还是3个？
transaction_pool应该是一个还是3个？

# 方案决定
合理的方案是三个client分别管理三个transaction_pool，一个protocol，将原先单独处理的部分，block_announce_validator,能够改造成与client的实例有关的。

Client对链的历史数据读取、从哈希到blockNumber或者反之，这些功能是在Backend里面实现，Backend是一个数据库和基于此数据库提供的链操作，当区块信息插入数据时，以一些特定的值作为KEY进行插入。在Pnyx中，有三个种类型的Client对象，为了避免冲突，每一种Client对象使用独立的Backend和数据库

# Client定制

参考TFullClient和TLightClient

我们有三种类型的Client:
1. Beacon Client: 信标链节点
2. Relay Client： 中继链节点
3. Shard Client： 分片链节点  
这三种类型的Client的差别在于所需要处理的数据以及数据处理方案的不同，我们在Client中实现所有的数据处理方案，然后通过标识位（角色）来决定处理哪一种数据，按照哪一种方式执行，通过这种方式，可以将三种实现合成到一个实现里。因此我们还是在TFullClient上添加功能。并不另外增加对象。

与现有的执行方案一致，  Client最终是通过Callable和Appliable 来执行所有的操作，因此需要做以下两个处理：
1. 定义新的Callable和Appliable
2. 根据不同的类型，在生成Client时配置不同的Callable
一个client具有四种泛型属性模板
* Backend 后端数据存储类型，这个主要包含了链的处理，有链的存储、导入、区块头的管理等。
* E      Executor 执行类型
* BlockT 区块的类型
* RuntimeApi: Runtime提供的API接口



## 定制runtime
参考[node-template]()中的node-template，我们可以构建不同的pallet，然后通过construct_runtime!来创建一个特定的Runtime，然后再在ServiceBuilder中，使用不同的Runtime来构建特定的client。 因此我们将会有三种Runtime:
1. Beacon Runtime
2. Relay Runtime
3. Shard Runtime

现有的生成不同的Client的方案是在定义ServiceBuilder时，使用不同的Runtime:
```rust 
12let builder = sc_service::ServiceBuilder::new_full::<
            node_template_runtime::opaque::Block, node_template_runtime::RuntimeApi, crate::service::Executor
        >($config)?
```