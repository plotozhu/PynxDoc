amt的数据结构说明见[这里](https://lrita.github.io/2018/07/15/hash-tree-hamp/)
```go

type Root struct {
	Height uint64  //区块高度
	Count  uint64  //?似乎是这个root下的总的Node的数量
	Node   Node    //AMT节点

	bs Blocks      //block store，存储区块数据的k-v数据库对象
}
```

这是一个在某个高度上的根的记录，