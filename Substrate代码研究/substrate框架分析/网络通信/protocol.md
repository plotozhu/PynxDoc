# 同步过程
同步的核心由三个部分组成：  

* pending_request,表示有哪些节点是可能有数据需要同步的  
* block_requests()函数：检查所有的pending_request的节点，看看是否有哪些需要同步的,生成peer_request请求
* update_peer_request，把上述的block_requests产生的结果放置到对应的节点中去

## pending_request
其中pending_requests是一组可能需要同步的连接，在这几个函数中被更新：
`on_block_data` `new_peer` `set_sync_fork_request` `on_block_justification` `on_block_finality_proof` `finish_block_announce_validation`

# 优化方案
在原来的方案中，一旦超时或是错误，就立刻断开连接。　而在新方案中，这是有问题的：

1) 连接可能具有多个，其中的一个超时了？
2) 如果某个类型的只有这一个节点怎么办？