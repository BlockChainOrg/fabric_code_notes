# Fabric 1.0源代码笔记 之 Orderer #ledger（Orderer Ledger）

## 1、Orderer Ledger概述

Orderer Ledger代码分布在orderer/ledger目录下，目录结构如下：

* orderer/ledger目录：
	* ledger.go，Factory、Iterator、Reader、Writer、ReadWriter等接口定义。
	* util.go，Orderer Ledger工具函数，及NotFoundErrorIterator结构体定义。
	* file目录，file类型ledger实现。
	* json目录，json类型ledger实现。
	* ram目录，内存类型ledger实现。
	
## 2、接口定义

Factory接口定义：

```go
type Factory interface {
	//按chainID获取已存在的ledger，如不存在则创建
	GetOrCreate(chainID string) (ReadWriter, error)
	//获取ChainID列表
	ChainIDs() []string
	//关闭并释放所有资源
	Close()
}
//代码在orderer/ledger/ledger.go
```

ReadWriter接口定义：

```go
type Reader interface {
	//按起始块号获取迭代器
	Iterator(startType *ab.SeekPosition) (Iterator, uint64)
	//获取ledger高度（即块数）
	Height() uint64
}

type Writer interface {
	//ledger向追加新块
	Append(block *cb.Block) error
}

type ReadWriter interface {
	Reader //嵌入Reader
	Writer //嵌入Writer
}

type Iterator interface {
	//获取下一个可用的块
	Next() (*cb.Block, cb.Status)
	//获取可用的通道
	ReadyChan() <-chan struct{}
}
//代码在orderer/ledger/ledger.go
```

## 3、file类型ledger实现

### 3.1、fileLedgerFactory结构体及方法（实现Factory接口）

```go
type fileLedgerFactory struct {
	blkstorageProvider blkstorage.BlockStoreProvider //blkstorage
	ledgers            map[string]ledger.ReadWriter //多链
	mutex              sync.Mutex
}

//从ledgers中查找，如找到则返回，否则创建Ledger（即blkstorage）并构造fileLedger
func (flf *fileLedgerFactory) GetOrCreate(chainID string) (ledger.ReadWriter, error)
//获取已存在的Ledger列表，调取flf.blkstorageProvider.List()
func (flf *fileLedgerFactory) ChainIDs() []string
//关闭并释放资源flf.blkstorageProvider.Close()
func (flf *fileLedgerFactory) Close()
//构造fileLedgerFactory
func New(directory string) ledger.Factory
//代码在orderer/ledger/file/factory.go
```

blkstorage更详细内容，可参考：[Fabric 1.0源代码笔记 之 Ledger #blkstorage（block文件存储）](../ledger/blkstorage.md)

### 3.2、fileLedger结构体及方法（实现ReadWriter接口）

```go
type fileLedger struct {
	blockStore blkstorage.BlockStore //blkstorage
	signal     chan struct{}
}

//按起始块号获取迭代器
func (fl *fileLedger) Iterator(startPosition *ab.SeekPosition) (ledger.Iterator, uint64)
//获取ledger高度（即块数）
func (fl *fileLedger) Height() uint64
//ledger向追加新块
func (fl *fileLedger) Append(block *cb.Block) error
//代码在orderer/ledger/file/impl.go
```

### 3.3、fileLedgerIterator结构体及方法（实现Iterator接口）

```go
type fileLedgerIterator struct {
	ledger      *fileLedger
	blockNumber uint64 //当前已迭代的块号
}
//获取下一个可用的块，如果没有可用的块则阻止
func (i *fileLedgerIterator) Next() (*cb.Block, cb.Status)
//获取可用的通道，如果块不可用返回signal，否则返回closedChan
func (i *fileLedgerIterator) ReadyChan() <-chan struct{}
//代码在orderer/ledger/file/impl.go
```

## 4、Orderer Ledger工具函数

```go
//创建块
func CreateNextBlock(rl Reader, messages []*cb.Envelope) *cb.Block
func GetBlock(rl Reader, index uint64) *cb.Block
//地址在orderer/ledger/util.go
```

func CreateNextBlock(rl Reader, messages []*cb.Envelope) *cb.Block代码如下：

```go
func CreateNextBlock(rl Reader, messages []*cb.Envelope) *cb.Block {
	var nextBlockNumber uint64
	var previousBlockHash []byte

	if rl.Height() > 0 {
		it, _ := rl.Iterator(&ab.SeekPosition{
			Type: &ab.SeekPosition_Newest{
				&ab.SeekNewest{},
			},
		})
		<-it.ReadyChan()
		block, status := it.Next() //获取前一个最新的块
		nextBlockNumber = block.Header.Number + 1
		previousBlockHash = block.Header.Hash() //前一个最新的块的哈希
	}

	data := &cb.BlockData{
		Data: make([][]byte, len(messages)),
	}

	var err error
	for i, msg := range messages {
		data.Data[i], err = proto.Marshal(msg) //逐一填充数据
	}

	block := cb.NewBlock(nextBlockNumber, previousBlockHash)
	block.Header.DataHash = data.Hash()
	block.Data = data

	return block
}
//地址在orderer/ledger/util.go
```