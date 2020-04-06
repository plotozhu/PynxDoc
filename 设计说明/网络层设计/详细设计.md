# 消息化调用
为了防止服务提供的函数在多个线程中被调用产生的同步问题，系统采用消息机制实现，网络结构如下：
![](./networker.png)
在创建实际应用时，通过NetworkWorker::new()创建一个实例networker，这个类将创建NetworkService和network_service(一个libp2p swarm)对象。**名字为什么这么绕口和相似？substrate干的！**。NetworkerService实例可以通过.service()函数返回出去。  
客户端使用返回得到的NetworkService实例提供的接口对network_service进行操作，而这个network_service就是NetworkBehaviour对象，也就是Transp2p/DelaySend/GroupSet实例。  
由上图可知，在操作时，通过Tx/Rx进行解耦，这样所以的network_service操作调用都将在poll时解析命令然后在poll中来调用相应函数实现。这样就不需要考虑这些函数的可重入性。对结构内的数据进行操作时，也无需使用同步。  
当然，这种方案带来了一个麻烦，就是所有的函数接口必须NetworkService中定义成事件进行收发。另外会不会因为单线程导致堵塞。考虑到network_service(NetworkBehaviour)的服务特性（只是提供服务接口），使用这种方案是值得的。在以下的设计中，Transp2p位于network_service(Behaviour)中，而GroupSet和DelaySend位于NetworkService中。注意，由于NetworkService中的接口函数可能会在多个线程中被调用，因此GroupSet/DelaySend的实例必须使用Mutex进行同步管理。

我们在以下的设计中，首先分开设计各个服务模块的对外接口及内部过程。最后统一设计消息化的事件。

# Transp2p
Transp2p有三个部分组成,routetab--路由表，swarmkad--具有饱和度计算的KAD表，以及transpp本身。
Transpp实现了NetworkBehaviour，并且通过Legency模式的CustomMsg实现节点间的通信。
Transpp通过push/pull的方式进行数据传输，即发送大数据时，即点首先推送数据的哈希，对端收到哈希时，检查是否有该哈希对应的数据，如果没有则请求相应的数据，否则就直接使用本地已经存在的数据。

详细的传输设计方案见[传输框架详细设计](./传输框架详细设计.md)
# GroupSet
GroupSet实现分组管理功能,由于KAD服务，读取/设置Provider的过程是异步的事件过程，因此我们接收异步的消息回应并且更新本地缓存。KAD服务本身会管理Provider信息的自动重复发送(见[libp2p-kad](../../Substrate代码研究/libp2p研究/libp2p-kad.md))，因此我们只需在加入时启动provider/退出时停止provider。
由于自身入的组和跟踪的组的差别仅仅为是否启动provider，因此我们在数据结构上可以统一管理：
## 数据结构说明
### 组缓存信息项
```rust
    struct GroupInfo {
        joined:bool,  //是否加入该组
        providers:Vec<PeerId>, //提供该服务的节点
    }
```
### 组缓存
```rust
    group_info_map:HashMap<&str,GroupInfo>
```
group_info_map是一个有关str到GroupInfo的映射表，这个映射表在Provider事件到达时被更新，然后被客户端代码所读取。  
如果GroupSet在Behaviour(network_service)层，数据从Behaviour事件导出，存入到roup_info_map中时，需要复制生成一次数据。再由于客户端代码通过service是的事件来获取数据，因此数据需要再通过复制机制在线程间传输，这会大大降低效率。 因此将GroupSet直接放置到NetworkService对象中。

### 定时器
```rust
    timer:futures_timer::Interval,
```
这是一个定时器，每隔5分钟启动一次，启动时根据group_info_map的key值，发送一遍provider读取请求。结果会通过异步地通过事件返回，并且在poll中被处理。

## 过程说明
### 加入分组    
join_group(group_name:&str, min_peers_required:usize)
#### 执行动作
1. 如果group_info_map中不存在，用空的provider和join:true创建一个GroupInfo对象，否则
2. 
3. 如果在group_info_map中存在，
   1. 设置joined为true
4. 调用startprovider通知全网自身加入这个组了。

### 获取分组节点    
get_group_peers(group_name:&str)-> Vec<PeerId>
#### 执行动作
1. 从group_info_map中读取Providers
   
### 退出分组    
 leave_group(group_name:&str)
#### 执行动作
1. 调用stop_provider通知全网自身离开这个组了

### 跟踪分组    
 trace_group(group_name:&str)
#### 执行动作
1. 如果group_info_map中不存在，用空的provider和join:false创建一个GroupInfo对象，否则
1. 如果在group_info_map中存在,则忽略。

### 停止跟踪分组    
untrace_group(group_name:&str)
### 执行动作
从group_info_map中删除该项

### 向分组发送数据    
group_message(group_names:Vec<&str>,data:Vec<u8>)->Result<usize,Error>
### 执行动作
1. 根据group_names查询分组内的节点，作为发送的目标节点
2. 合并这些目标节点
3. 向这些目标节点发送数据
### 设置接收事件流    
register_event_stream()-> mpsc::UnboundedReceiver<GroupSetEvent>

### poll
1. 根据收到的事件，决定是否更新group_info_map
2. 如果定时器到达：
   1. 根据group_info_map里的所有的key，发送一次get_provider命令

# DelaySend
## 数据结构说明：
### 待发送的数据项
```rust
struct PendDataItem {
    weight:u64,
    data:Vec<u8>,
    timeout:futures_timer::Delay,
}
```
在这个数据项中，
* weight是权重
* data是编码后的待发送的数据
* timeout:是要发送/或是取消的时间

### 待发送队列 pending_data_queue;
在类结构中有一个以data_id为键值的待发送数据映射表，当有数据插入时，根据data_id匹配数据是否重复，这个数据发送映射表中有一个超时时间。
```rust

pending_data_queue:HashMap<BlockT::Hash,PendData>
```
data_id是由应用层决定的数据标识，它是一个哈希值，在DelaySend模块中，data_id一样的数据被认为是可互相覆盖的（一般来说是高山权重的覆盖低权重的），如何生成data_id是应用层管理的，在Pynx中，使用的是 Hash(（数据类型）2字节+（分片ID）2字节 + （区块高度）8字节)来表征唯一的ID值
### 已经发送队列 sent_data_cache
在运行过程，可能会遇到这样一种情况，同一个data_id的数据，当高权重的数据被发送完毕后，又收到了低权重的数据，这时候，pending_data_queue中的数据已经被移除，因此模块无法知晓这个数据已经被处理过，有可能会再转发一次该低权重的数据。为了解决这个问题，我们增加一个sent_data_cache的数据：
```rust
    sent_data_cache:LruCache<Hash,weight>
```
这是一个LRU（最长时间不用）的缓存，当数据被发送后，移动到此对象中。当有新的数据到达时，通过此对象检查是否有更高权重的数据被发送过，如果有，就拒绝/忽略新来的数据。
考虑另外一种情况：低权重的数据被发送后，信息移动到了sent_data_cache队列中；然后又来了一个高权重的数据，经过比较需要发送，因此会在pending_data_queue又添加一项，然而这时候sent_data_cache对象中的数据还没有过期。在这种情况下，sent_data_cache/pending_data_queue中可能会存在同一个data_id。为了解决这个问题，代码需要在pending_data_queue中加入数据时，从sent_data_cache中删除对应的项。
## 过程说明
### delay_send_groups
#### 说明
这个是提供给客户端的调用函数，当客户端有数据要进行延时组播发送时，调用此函数
#### 过程
1. 检查data_id是否在sent_data_cache中存在，
   1. 如果存在，比较weight，如果新数据的weight小于原有的，返回并且告诉调用者这个操作被忽略
2. 检查pending_data_queue中是否存在该data_id,
   1. 如果存在，比较weigt,如果weight小于原有的，返回并且告诉调用者这个操作被忽略
3. 生成一个新的PendDataItem，添加到pending_data_queue中
4. 从sent_data_cache中删除对应的data_id项

### do_send_groups
这个是内部定时器到达时，进行的操作：
1. 调用GroupSet进行数据发送
2. 从pending_data_queue中移除
3. 在sent_data_cache中添加<data_id,weight>
### poll
poll是future调用的，需要做以下的工作：
1. 遍历pending_data_queue所有的项，
2. 如果项里的timeout到达，就对该项执行do_send_groups

# 补充设计说明
NetworkService中需要两个实例：
```rust
    group_set:Mutex<GroupSet>,
    delay_sender:Mutex<DelaySend>,
```
这两个实例必须用Mutex包裹起来。

## SeedGroup有源组
![](./seedgroup_routine.png)