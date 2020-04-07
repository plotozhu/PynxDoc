# 概述
Transp2p模块里包含四个部分：如下图所示：
![](transp2p.png)
从图上可以看到,Transp2p内需要使用RouteTab和SwarmKAD两个对象的实例，完成了RouteManage/Transpp两个特质，并且需要实现NetworkBehaviour特性。

# SwarmKAD
# RouteTab

# Transpp特质实现
## 数据定义
### 消息队列
消息队列用于存放需要Transp2p处理的消息
```rust
    events: Vec<mpsc::UnboundedSender<CustomEventOut<B>>>,
```
### 数据缓存
数据缓存是哈希/Vec<u8>的映射，用于存放已知的哈希值及其对应的数据，目前第一个版本使用LruCache，未来是否需要使用LruCache+db进行持久化存储？
```rust
    data_cache:Mutex(HashMap<hash,Vec<u8>>)
```
这个是由Transp2p创建后，在new的时候传入进来的。
### 读取队列
读取队列用于记录当前的正在发送请求获取对应数据的哈希值，同时还记录了所有可能具有该数据的节点（从其他节点推送了同样的数据请求）
```rust
    retrieving_cache:HashMap<hash,Vec<RelayDataReq>>
```

### 已汇报队列
最近的已经向上一层应用汇报过的数据请求，这是为了避免多次处理
```rust
    processed_cache:Hashet<(target,hash))>
```
这里使用的是target,hash，因此需要考虑以下两点：
1. 如果要求中继的数据是直接数据，也需要生成一下哈希用于记录
2. 如果有多个节点请求往同一目标节点发送数据，只有第一次的会被转发，后续的会被丢弃
* 在这种处理模式下，如果第一次数据到达，转发没有成功，第二次再到达时，即使转发是可能成功，也不会再做了。
### 事件定义
```rust
#[derive(Debug)]
pub enum TransppEventIn<B:Hash> {
    RelayDataEvt(RelayDataReq<B>),
    PullDataEvt(PullDataReq<B>),
    PullDataRespEvt(PullDataResp<B>),
    RelayDataArrive(RelayDataReq<B>),
    None,
}
```

事件的数据结构
```rust 
#[derive(Debug, Eq, PartialEq,Clone,Encode,Decode)]
struct RelayDataReq<B:Hash>{
    pub    routeInfo:FindRouteReq,
    pub    hash:B::Out,
    pub    data:Vec<u8>,
    pub    sign_packet:Vec<u8>, 
}
#[derive(Debug, Eq, PartialEq,Clone,Encode,Decode)]
pub struct PullDataReq<B:Hash> {
    hash: B::Out,
}
#[derive(Debug, Eq, PartialEq,Clone,Encode,Decode)]
pub struct PullDataResp<B:Hash> {
    hash: B::Out,
    data: Option<Vec<u8>>,
}
``` 
## 函数定义
### 新建
#### 响应动作
创建一个对象返回

### 请求发送中继数据
#### 说明
#### 函数签名
#### 响应动作
如果data比较小

### 网络事件到达
#### 响应动作
匹配事件消息类型，调用相应的函数进行处理：
 
### poll
#### 响应动作
如果events中有事件，Poll::Ready()此事件，否则返回Poll::Pending

### on_relay_data

#### 响应动作
1. 检查target/hash对，如果已经在reported_cache中，就结束
2. 如果data有数据，构建RelayDataArrive（relay_req）放到events中
3. 在reported_cache中添加该target/hash记录
4. 如果在retrieving_cache中有该哈希，将relay_req添加到retrieving_cache的Vec队列中，结束
5. 如果在retrieving_cache中没有该哈希，创建一个哈希和`Vec<relay_req>`的记录，然后
6. 创建一个`PullDataEvt（PullDataReq{peerId,relay_req.hash>}`的事件，放到events中

### on_pull_data
#### 响应动作
1. 如果data_cache中有，