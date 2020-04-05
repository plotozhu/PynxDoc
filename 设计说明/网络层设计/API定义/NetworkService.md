# API 概要
## Transpp
| 序号 | 名称                 | 函数签名                          |
|---------|---------------------|----------------------------------|
| 1   | 发送数据             | `send_message(targets:Vec<PeerId>, alpha:usize, ttl:usize, data:Vec<u8>)`        |
| 2   | 设置接收事件流         | `register_event_stream( subname:&str)->  mpsc::UnboundedReceiver<TransppEvent>` |

## Groupset

| 序号 | 名称                 | 函数签名                          |
|-----|---------------------|----------------------------------|
| 1   | 加入分组            | `join_group(group_name:&str, min_peers_required:usize)`        |
| 2   | 获取分组节点         | `get_group_peers(group_name:&str)-> Vec<PeerId>` |
| 3   | 退出分组         | `leave_group(group_name:&str)` |
| 4   | 跟踪分组         | `trace_group(group_name:&str)` |
| 5   | 停止跟踪分组         | `untrace_group(group_name:&str)` |
| 6   | 向分组发送数据         | `group_message(group_name:&str,data:Vec<u8>)->Result<usize,Error>` |
| 7   | 设置接收事件流          | `register_event_stream()-> mpsc::UnboundedReceiver<GroupSetEvent>` |

## DelaySend

| 序号 | 名称                 | 函数签名                          |
|-----|---------------------|----------------------------------|
| 1   | 延时组播            | `delay_send_groups( group_name:Vec<&str>, data_id:Vec<u8>, delay:Duration, weight:usize,data:Vec<u8>) ->Reult<State,Error>` |

# 详细描述
NetworkService是提供给就应用层的，包含了在框架中的Transpp和GroupSet两个部分
## Transpp
### 发送数据
#### 功能描述
发送数据是指向某个特定的PeerId发送消息，这个PeerId可能是邻节点，也可能不是邻节点。如果不是邻节点，会先进行一次路由发现，然后再发送数据。此处并没有发送成功与否的回应，应用在需要时，可以添加发送/回应/超时重发的机制来实现可靠的数据传输。  
#### 函数签名
`send_message(targets:Vec<PeerId>,alpha:usize,ttl:usize,data:Vec<u8>)`
#### 参数
* **targets**: 发送的目标节点，可以有多个或是1个
* **alpha**： 扇出系数，在每级路由上，请求多少个节点
* **ttl**： 最多发送多少层
* **data**: 需要发送的数据，是经过编码的
#### 返回值
无
#### 附加说明
   1. 最坏情况下，需要支付ttl<sup>alpha</aplha>倍的费用
   2. 如果ttl和alpha都为0，那么会根据路由表自动选择最长的路径为ttl，并且将alpha设置为1
   3. 如果没有相应的路由，会自动路由发现一次。


### 设置接收事件流
#### 功能描述
模块通过事件机制向上层汇报信息，需要管理事件的上层应用，需要注册一个事件接收流，当事件发生时，模块向该流推送相应的事件消息
#### 函数签名
`register_event_stream(subname:&str)-> mpsc::UnboundedReceiver<TransppEvent>`
#### 参数
   subname:一个字符串，用于不用的应用接收不同的流，只有属于该流的事件才会被推送。
#### 返回值
返回一个mpsc::UnboundedReceiver<TransppEvent>，需要使用future/poll机制来实现数据读取。


## GroupSet
GroupSet通过KAD网络中的Start/Get Provider来实现

### <span id="join_group">加入分组</span>
#### 功能描述
GroupSet分组是自由设置的功能，通过DHT网络中的StartProvider进行，因为一旦提供Provider，需要将该信息定时向外扩散，为了避免恶意节点发送无效的信息，我们对同一个节点同时可以加入分组的数量作限定，这个限定值在配置中实现，目前的设置是10。
#### 函数签名
`join_group(group_name:&str,min_peers_required:usize)`
#### 参数
*   **group_name** 分组名字
*   **min_peers_required** 希望最少找到这么多分组节点
#### 返回值
   无
#### 说明
如果分组节点数少于`min_peers_required`，就会启动一个定时器，定时查询读取节点个数

### 获取分组节点
#### 功能描述
区取分组节点用来获取当前缓存的属于该分组的节点
#### 函数签名
`get_group_peers(group_name:&str)->Vec<PeerId>`
#### 参数
* **group_name**: 分组名称
#### 返回值
* **`Vec<PeerId>`**，网络中属于该分组的节点
#### 说明
查询分组节点是经过DHT网络进行的，这个通过定时器进行，见[加入分组](#span-id%22joingroup%22%e5%8a%a0%e5%85%a5%e5%88%86%e7%bb%84span)，而此处读取到的仅仅是查询缓存在内部的

### 退出分组
#### 功能描述
当节点需要离开某个分组时，调用该函数退出，该函数实现两个功能：
1. 停止对外广播自身在该分组内
2. 断开与该分组内的节点的连接
#### 函数签名
`leave_group(group_name:&str)`
#### 参数
* **group_name**需要离开的分组名称
#### 返回值
无

### 跟踪分组
由于读取分组信息是异步的，这样如果节点不在分组里，当节点想要给分组发送数据的时候，本地的分组信息可能是空的，需要发现/回应后再次发送数据，这样的话，数据发送的延时会增加，失败率会升高。可以使用监视分组来解决这个问题，节点一旦开始监视分组，就会定时重复查询该分组的provider信息，这样节点就可以根据到这个分组内的节点信息，当需要发送数据时，直接向该分组的节点发送即可。与加入分组最大的差别是节点本身并不加入此分组。
#### 功能描述
节点开始定时查询该分组的提供者的节点信息：
1. 启动定时器查询分组的提供者节点信息
2. 缓存这些节点信息
#### 函数签名
`trace_group(group_name:&str)`
#### 参数
* **group_name**要跟踪的分组名称
#### 返回值
无

### 停止跟踪分组
#### 功能描述
当节点不再需要向某个分组发送数据时，调用该函数停止跟踪分组节点，该函数实现两个功能：
1. 停止查询分组的定时器
2. 删除缓存的分组信息
#### 函数签名
`untrace_group(group_name:&str)`
#### 参数
* **group_name**需要停止观察的分组名称
#### 返回值
无

### 向分组广播数据
#### 功能描述
通过此功能，应用层可以在不知道分组中节点信息的情况下，向分组进行广播。注意，调用此功能时，
1. 要么节点已经加入分组
2. 要么节点已经跟踪分组
否则会返回unknown_group的错误。
#### 函数签名
`group_message(group_name:&str,data:Vec<u8>)->Result<usize,Error>`
#### 参数
* **group_name**： 分组名称

* **data**： 需要传输的数据
#### 返回值
* **usize**: 用于发送结果的数据帧指示,是一个从0开始的序列号
#### 说明
由[libp2p-kad说明](../../../Substrate代码研究/libp2p代码研究/../libp2p研究/libp2p-kad.md)可知，系统中将缓存0-K个属于该分组的节点，当调用此函数时，如果系统中缓存的节点个数小于alpha个，那么将返回错误，否则返回一个确定的值，这个值可以被上层用于处理数据回应之类的。
   

### 设置接收事件流
#### 功能描述
模块通过事件机制向上层汇报信息，需要管理事件的上层应用，需要注册一个事件接收流，当事件发生时，模块向该流推送相应的事件消息
#### 函数签名
`register_event_stream()-> mpsc::UnboundedReceiver<GroupSetEvent>`
#### 参数
   无
#### 返回值
返回一个mpsc::UnboundedReceiver<GroupSetEvent>，需要使用future/poll机制来实现数据读取。

## DelaySend
### 延时w分组广播
延时转发应用的链上消息，链上的节点总是分组的，因上延时转发只能按分组发送
#### 函数签名
`delay_send_groups(group_name:Vec<&str>,data_id:Vec<u8>,delay:Duration,weight:usize,data:Vec<u8>)->Reult<State,Error>`
#### 参数
   **group_name**:所需要发送的分组，可以是多个
   **data_id** :数据标识  
   **delay**:需要延时的时间  
   **weight**:权重  
   **data**:需要发送的数据  
#### 返回值
   Accepted,Replaced,Discard：分别代表:数据被接受，数据被接受并且替换了原有的数据，本数据被抛弃
#### 说明
数据标识由应用确定，DelaySend模块根据数据标识来确定某个数据是否需要根据权重被替换

## 不常用的API
### 请求进行路由发现
#### 功能描述
请求模块进行路由发现，如果路由表中已经有该节点的路由信息了，就直接返回，也可以强制要求重新进行路由发现。
#### 函数签名
`find_route(target:PeerId,alpha usize,ttl:usize,force:bool)->Result<bool,Error>`
#### 参数
* **target**: 需要发现的目标地址
* **alpha**: 每一层次的扇出系数
* **ttl**: 最多层次（time_to_live)
* **force**: 是否强制刷新，强制刷新是即使路由里有，也请求一次
#### 返回值
 **一个布尔项**, 指示是否路由表中是否有对应的信息了
#### 说明
* alpha的取值0-3，如果是0，表示是系统默认值（一般是3）
* ttl取值1-20，如果是1，就是在邻节点中查找
* 如果参数不正确，返回相应的错误信息
  
### 获取路由表项
获取路由表中的路由信息，注意如果路由表中没有该节点的路由信息，就直接返回。如果需要明确的路由信息，请使用"请求路由发现"，然后再监听路由回应事件。
#### 函数签名
`get_route_item(target:PeerId)->Option<Vec<RouteItemCodec>>`
#### 参数
* **target**: 需要获取路由表项的目标地址
#### 返回值
 **`Option<Vec<RouteItemCodec>>`** 一组路由表项