基本状态结构 
```go
type StateManager struct {
	cs *store.ChainStore

	stCache  map[string][]cid.Cid
	compWait map[string]chan struct{}
	stlk     sync.Mutex
}
```
	stCache  map[string][]cid.Cid 对应某个tipset时的状态根和收据的根
    compWait map[string]chan struct{}是获取某个，目标是为了解决TipSetState的重入问题

`StateManager.TipSetState` 计算在给定的tipSet时的状态根，如果cache中有，直接取出，如果没有，使用`StateManager.computeTipSetState`计算，由于`StateManager.computeTipSetState`需要时间，所以使用compWait来实现函数的可重入和免重复计算


