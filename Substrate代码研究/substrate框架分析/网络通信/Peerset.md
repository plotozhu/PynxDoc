
# Peerset 分组策略

## 可配置优先组
1. 不占用连接槽(slot)
2. 启动是主动建立连接
3. 可将最近优先组持久化保存在本地文件系统，节点重启后快速恢复连接

## In Peers
 通过reserved_only:true，并讲同组节点加入priority_groups["reserved"],以此限制入栈peer与本地节点同组。
 
 
## 连接上级节点
将上级节点（中继链节点）加入priority_groups 不占用连接槽(slot).


## 连接槽分配策略

1. 主动连接优先组
