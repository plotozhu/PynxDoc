# Behaviour
[注释翻译]
网络行为，允许定制swarm的行为  
<font color="red"> 本特征被设计成可组合的，可以把多种实现组合成一种，这样可以一次性处理所有的行为 </font>

# 派生 `NetworkBehaviour`

 板箱模块的用户可以通过使用由 `libp2p`导出的宏`#[derive(NetworkBehaviour)]`来实现本特征。该宏生成一个实现该`struct`的代理`trait`，用于代理特征定义的所有的函数调用。结构体所有产生的事件都被代理到`NetworkBehaviourEventProcess`，这个函数应该由使用者实现。  

自定义的`poll`函数是可选的，这个函数必须添加`#[behaviour(poll_method = "poll")]`标签属性，并且会在最终以无参数的形式调用。  

在默认情况下，派生者设置`NetworkBehaviour::OutEvent` 为 `()`，但是这个可以用`#[behaviour(out_event = "AnotherType")]`宏来覆盖。  

如果结构体的某字段没有实现`NetworkBehaviour`，可以在该字段上添加`#[behaviour(ignore)]`宏来禁止产生此字段的代理。

## 特征定义
* ProtocolsHandler
* OutEvent
* new_handler
* addresses_of_peer
* inject_connected
* inject_disconnected
* inject_replaced
* inject_node_event
* inject_addr_reach_failure
* inject_dial_failure

# ProtocolHandler
用于处理与某个远端的连接上的一组协议的处理器。  
当一个类型需要管理与某个远端连接的特定协议的状态时，需要实现本特征。  

## 处理协议
与包含一组协议远端的通信通过以下两种方式之一进行初始化：
1. 通过初始化一个出站子流来进行初始化。要想达到这个目的：[`ProtocolsHandler::poll()`]必须返回一个[`libp2p_core::upgrade::OutboundUpgrade`]实例的[`ProtocolsHandlerEvent::OutboundSubstreamRequest`]，该实例可用于进行协议协商。当成功时，会以最终升级结果为参数调用[`ProtocolsHandler::inject_fully_negotiated_outbound`]
   *备注* 升级一词的来源是因为初始化的时候是以某一个子流协商，最终得到了所有的协议，因此称为升级。 
2. 监听并且接受入站的子流，当在一个连接上建立一个新的入站子流时，[`ProtocolsHandler::listen_protocol`] 将被调用以获得一个[`libp2p_core::upgrade::InboundUpgrade`]实例，该实现用于协议协商。当成功时会以最终升级结果为参数调用[`ProtocolsHandler::inject_fully_negotiated_inbound`]

## 连接保持
`ProtocolsHandler`能够通过[`ProtocolsHandler::connection_keep_alive`]影响低层的连接。也就是说，处理器所实现的协议能够包括终目连接的条件，成功协商后的子流的生命周期完全由处理器来控制。  

本特征的实现者必须注意连接可能在任何时刻被关闭。当一个连接被优雅地关闭，该处理器使用的子流仍旧可以继续读取数据，直到远端也关闭了这个连接。

## 特征定义


## 实现自定义的逻辑
通常的方式是在behaviour中添加一个枚举类型 
```rust
/// Event that can be emitted by the behaviour.
#[derive(Debug)]
pub enum BehaviourOut<TMessage> {
    /// Opened a custom protocol with the remote.
    CustomProtocolOpen {
        /// Identifier of the protocol.
        protocol_id: ProtocolId,
        /// Version of the protocol that has been opened.
        version: u8,
        /// Id of the node we have opened a connection with.
        peer_id: PeerId,
        /// Endpoint used for this custom protocol.
        endpoint: ConnectedPoint,
    },
   ....
}
```
和一个事件队列：
```rust
    /// Queue of events to produce for the outside.
    #[behaviour(ignore)]
    events: Vec<BehaviourOut<TMessage>>,
```
在事件发生时放到这个列表：
```rust
impl<TMessage, TSubstream> NetworkBehaviourEventProcess<IdentifyEvent> for Behaviour<TMessage, TSubstream> {
    fn inject_event(&mut self, event: IdentifyEvent) {
        match event {
            IdentifyEvent::Identified { peer_id, mut info, .. } => {
            ...
            self.events.push(BehaviourOut::Identified { peer_id, info });
            ...
         }
      }
   }
}
```
并且在behaviour中添加一个poll过程来产生消息：
```rust
impl<TMessage, TSubstream> Behaviour<TMessage, TSubstream> {
    fn poll<TEv>(&mut self) -> Async<NetworkBehaviourAction<TEv, BehaviourOut<TMessage>>> {
        if !self.events.is_empty() {
            return Async::Ready(NetworkBehaviourAction::GenerateEvent(self.events.remove(0)))
        }

        Async::NotReady
    }
}
```
你需要在派生类上添加一个属性来使它能够工作：
```rust
/// General behaviour of the network. Combines all protocols together.
#[derive(NetworkBehaviour)]
#[behaviour(out_event = "BehaviourOut<B>", poll_method = "poll")]
pub struct Behaviour<B: BlockT, H: ExHashT> {
    /// All the substrate-specific protocols.
    substrate: Protocol<B, H>,
    /// Periodically pings and identifies the nodes we are connected to, and store information in a
    /// cache.
    debug_info: debug_info::DebugInfoBehaviour,
    /// Discovers nodes of the network.
    discovery: DiscoveryBehaviour,
    /// Block request handling.
    block_requests: protocol::BlockRequests<B>,
    /// Light client request handling.
    light_client_handler: protocol::LightClientHandler<B>,
    /// Queue of events to produce for the outside.
    #[behaviour(ignore)]
    events: Vec<BehaviourOut<B>>,
}
```
参考"派生 `NetworkBehaviour`",可以得到如下结论
<font color="red">在这个声明中，做了以下几件事：
1. 自动为substrate、debug_info、discovery、block_requests和light_client_handler属性代理behaviour行为
2. events属性由于有`#[behaviour(ignore)]`不会代理behaviour行为
3. 事件输出类型为`BehaviourOut<B>`
4. 支持`poll`
5. 在一个结构声明里组合了substrate、debug_info、discovery、block_requests和light_client_handler行为，即一开始的"本特征被设计成可组合的，可以把多种实现组合成一种，这样可以一次性处理所有的行为"
</font>
Polling Swarm将产生这些事件：
```rust
/// Polls for what happened on the network.
    fn poll_swarm(&mut self) -> Poll<Option<ServiceEvent<TMessage>>, IoError> {
        loop {
            match self.swarm.poll() {
                Ok(Async::Ready(Some(BehaviourOut::CustomProtocolOpen { protocol_id, peer_id, version, endpoint }))) => {
               ...
            }
            ...
            }
            ...}
            
```

## 数据收发处理
当上述改造完毕后，数据的收发通过 Ready(NetworkBehaviourAction）来实现,这个里面包括5种类型

```rust
pub enum NetworkBehaviourAction<TInEvent, TOutEvent> {
    GenerateEvent(TOutEvent),
    DialAddress {
        address: Multiaddr,
    },
    DialPeer {
        peer_id: PeerId,
    },
    SendEvent {
        peer_id: PeerId,
        event: TInEvent,
    },
    ReportObservedAddr {
        address: Multiaddr,
    },
}
```
* GenerateEvent(TOutEvent) 生成待处理的数据给上层

* SendEvent 发送事件给对等端
    DialAddress {
    DialAddress {
* DialAddress/DialPeer 拨号
* ReportObserverAddr 新地址到达