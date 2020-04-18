链可用的资源有限。 资源包括内存使用量、 存储 I/O 、 计算、 交易/区块大小和状态数据库大小。 有几种机制可以管理对资源的访问，并防止链中的各个组件对任何资源的消耗过多。 权重是用于管理验证区块所花费时间的机制。 一般来说，这用来限制存储 I/O 和计算。  
备注：权重不用于限制对其他资源的访问，例如存储本身或内存占用。 为此必须使用其他机制。  
区块可能包含的权重数是有限的，并且可选的权重消耗（即不需要作为区块初始化或最终确定阶段的一部分而部署的权重，也不需要用于强制性固有交易的权重）通常会通过经济措施受到限制，或简单来说，就是通过交易费用。 The fee implications of the weight system are covered in the Transaction Fees document.
# 权重基础知识
权重代表了区块链验证区块必须的 有限 时间。 这包括计算周期和存储 I/O 。 自定义实现可以使用复杂的数据结构来表示权重。 Substrate只是简单地使用数值来代表权重。
权重计算应始终：
* 在调用之前可计算。 区块创建者在实际决定是否接受某个交易之前应该能够检查其权重。
* 本身消耗很少的资源。 如果计算交易的权重与执行交易消耗类似的资源，这样是没有意义的。 因此，权重计算应该比执行交易更轻量级。
* 无需访问链上状态即可确定使用的资源。 权重擅长表示 固定 的测量或仅基于可调用函数参数的测量，而无需昂贵的 I/O 。 当成本取决于链上状态时，权重并不是那么有用。  

如果可调用函数的权重在很大程度上取决于链上的状态，则有两种选择：
*  针对可调用函数，给出可能消耗的权重上限。 如果强制上限与可调用函数权量的最小可能值的差额很小，则可以假设它始终处于权重的上限，而无需访问链上状态。 但是，如果差异太大，那么即使进行很少的交易，其经济成本也可能会太大，这将破坏激励措施，并导致低效率的链上吞吐。
*  要求将有效权重（或可用于有效计算权重的前提）作为参数传递到可调用函数中。 消耗的权量应基于这些参数，但也应包括在调用时验证它们所花费的时间。 必须进行验证以确保权重参数准确地对应于链上状态，如果不正确，则操作应友好地报错。

System模块负责在执行区块时汇总权重，确保不会超过限制。 Transaction Payment模块负责解析这些权重并根据它扣除相关的费用。 权重计算功能是runtime的一部分，因此可以根据需要进行升级。
# 区块权重和长度限制
除了影响手续费外，权重系统的主要目的是防止区块被填充太多的交易，以至于需要过长的时间来执行这些交易。 当在区块内处理交易时，System模块累计区块的总长度 (已编码交易总量，以字节为单位) 以及区块的总权重。 只要其中一个超过限制，该区块就不再接受任何交易。 这些限制在MaximumBlockLength和MaximumBlockWeight中定义。  
关于这些限制的一个重要说明是，其中一部分保留给了 Operaional可调用函数。 该规则适用于以上两个限制，并且比率可以在 AvailableBlockRatio 中找到。  
例如，如果区块长度限制为1 Mb，并且比率设置为80％，则所有普通交易可以填充该区块的前800 Kb，而最后200 Kb只能由操作operational类填充。  
There is also a Mandatory dispatch class that can be used to ensure an extrinsic is always included in a block regardless of its impact on block weight. Please refer to the Transaction Fees document to learn more about the different dispatch classes and when to use them.
# 后续步骤
进一步学习
Substrate Recipes 包含 自定义 权重 和自定义 WeightToFee 的示例。
srml-example模块。
# 示例
参阅示例在 自定义的runtime函数中 添加交易权重。
# 参考文档
查看 SRML Transaction Payment模块。
查找关于权重的信息，包括weights.rs中的 SimpleDispatchInfo 枚举。