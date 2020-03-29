# 箱 libp2p_swarm
[ - ]
网络的上层管理器。

一个Swarm包含了整个网络的状态。libp2p网络的整个行为可以通过swarm来控制。该Swarm结构包含所有到远程的活动和挂起的连接，并管理所有已打开的子流的状态以及所有构建的在这些子流上的升级。

## 初始化swarm
创建一个Swarm需要三样东西：  
1. 本地节点的网络标识，形式为PeerId。
2. 实现Transport特征。这个类型定义了如何根据地址传输到网络上其他节点。更多信息请参见transport模块。
2. 实现NetworkBehaviour特征。这是一个状态机，定义了swarm在连接到节点后的行为动作。
## 网络行为（NetworkBehaviour）
该NetworkBehaviour特征的实现类型指示了swarm的行为方式。这包括支持哪些协议以及应该尝试连接到哪些节点。NetworkBehaviour控制网络上发生的行为动作。可以在多种类型上实现NetworkBehaviour，然后组合为一个行为。

## 协议处理程序
ProtocolsHandler特征定义了每个到远端的连接的行为方式：如何处理传入子协议，支持哪些协议，何时打开一个新的出站子流等

## 再导出
pub use protocols_handler::IntoProtocolsHandler;
pub use protocols_handler::KeepAlive;
pub use protocols_handler::ProtocolsHandler;
pub use protocols_handler::ProtocolsHandlerEvent;
pub use protocols_handler::ProtocolsHandlerUpgrErr;
pub use protocols_handler::SubstreamProtocol;
## 模组
protocol_handler    建立与远程对等方的连接后，ProtocolsHandler协商并处理该连接上的一个或多个特定协议。
toggle    
## 结构
DummyBehaviour    虚拟的实现NetworkBehaviour没有任何作用。
ExpandedSwarm    包含网络状态以及其行为方式。
IntoProtocolsHandlerSelect    IntoProtocolsHandler的实现，将两个协议合为一体。
OneShotHandler    ProtocolsHandler的实现，为每个单独的消息打开一个新的子流。
ProtocolsHandlerSelect    ProtocolsHandler特征的实现，将两个协议合为一体。
SwarmBuilder    
SwarmPollParameters    传递给poll()的，NetworkBehaviour有权访问的参数。
## 枚举
NetworkBehaviourAction    NetworkBehaviour可以Swarm 在其执行上下文中触发的动作。
SwarmEvent    由产生的事件Swarm。
## 特性
NetworkBehaviour    网络的行为,允许定制swarm。
NetworkBehaviourEventProcess    派生NetworkBehaviour特征时，必须对内部行为生成的所有可能的事件类型实施此特征。
PollParameters    传递给poll()的NetworkBehaviour有权访问的参数。
## 类型定义
NegotiatedSubtream    子流指示选择的协议。
swarm    包含网络状态以及其行为方式。