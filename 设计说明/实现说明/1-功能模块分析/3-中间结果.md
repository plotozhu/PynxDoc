# 说明
Executive模块是通过整合各个pallet执行block的调度模块。

# 跨分片中间结果的处理方式
中间结果有两种处理方式；
* 一种使用类似事件的方式进行，即提供接口给pallet，然后pallet中需要跨分片执行的中间结果，使用此接口记录需要跨分片交易的信息。  
* 另外一种方案使用返回值，我们设计一个跨分片中间类型，Call如果需要有跨分片执行的数据，就返回一个此中间类型；   
无论使用哪一种方式，Executive都需要在on-finalize时，收集这些记录。然后生成结果放到区块体里，然后将对应的哈希填充到区块头。由于第一种方式需要使用和修改现有的宏系统，因此使用第二种方式。 
在目前的Callable的系统中，Applable中返回的`DispatchOutcome`是 `Result<(), DispatchError>`形式，因此我们需要做以下和工作；
1. 定义一个跨分片交易的类`Intermediate`
2. 更新`DispatchOutcome`的形式为`Result<Option<Intermediate>, DispatchError>`
1. 所有的Extrinics执行后，收集`DispatchOutcome`的结果,然后根据此结果分类打包
2. 分类打包后，数据本地存储，生成哈希(CID),在P2P系统中存储<CID，哈希>对。

# 详细设计

## 数据结构
### Intermediate特质
Intermediate必须支持Decode和Encode
#### 获取执行目标
获取执行目标即（AccountID)
* 签名

这个函数可以用于对多个Intermediate对象进行分组合并。

应该也是一个Appliable的特质，这样可以使用标准的方式进行执行

## 执行过程
