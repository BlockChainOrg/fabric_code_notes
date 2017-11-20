# Fabric 1.0源码旅程 之 Ledger（2）blkstorage（block数据库）

## blkstorage概述

blkstorage相关代码在common/ledger/blkstorage目录，目录结构如下：

* blockstorage.go，定义核心接口BlockStoreProvider和BlockStore。
* fsblkstorage目录，BlockStoreProvider和BlockStore接口实现，即：fsblkstorage.NewProvider和fsblkstorage.fsBlockStore。

## 核心接口定义

BlockStoreProvider接口定义：提供BlockStore句柄。

```go
type BlockStoreProvider interface {
	CreateBlockStore(ledgerid string) (BlockStore, error)
	OpenBlockStore(ledgerid string) (BlockStore, error)
	Exists(ledgerid string) (bool, error)
	List() ([]string, error)
	Close() 
}
//代码在common/ledger/blkstorage/blockstorage.go
```

BlockStore接口定义：

```go
type BlockStore interface {
	AddBlock(block *common.Block) error
	GetBlockchainInfo() (*common.BlockchainInfo, error)
	RetrieveBlocks(startNum uint64) (ledger.ResultsIterator, error)
	RetrieveBlockByHash(blockHash []byte) (*common.Block, error)
	RetrieveBlockByNumber(blockNum uint64) (*common.Block, error) // blockNum of  math.MaxUint64 will return last block
	RetrieveTxByID(txID string) (*common.Envelope, error)
	RetrieveTxByBlockNumTranNum(blockNum uint64, tranNum uint64) (*common.Envelope, error)
	RetrieveBlockByTxID(txID string) (*common.Block, error)
	RetrieveTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error)
	Shutdown()
}
//代码在common/ledger/blkstorage/blockstorage.go
```

## BlockStore接口实现

BlockStore接口实现，即fsBlockStore结构体及方法，BlockStore结构体定义如下：




   117 blockfile_helper.go
   696 blockfile_mgr.go
    94 blockfile_rw.go
   381 blockindex.go
   218 block_serialization.go
   101 blocks_itr.go
   209 block_stream.go
    54 config.go
    93 fs_blockstore.go
    70 fs_blockstore_provider.go

