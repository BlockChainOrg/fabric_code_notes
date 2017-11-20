# Fabric 1.0源码旅程 之 Ledger（2）blkstorage（block数据库）

## blkstorage概述

blkstorage，默认目录/var/hyperledger/production/ledgersData/chains，含index和chains两个子目录。
其中index为索引目录，采用leveldb实现。而chains为各ledger的区块链文件，子目录以ledgerid为名，使用文件系统实现。

blkstorage相关代码在common/ledger/blkstorage目录，目录结构如下：

* blockstorage.go，定义核心接口BlockStoreProvider和BlockStore。
* fsblkstorage目录，BlockStoreProvider和BlockStore接口实现，即：FsBlockstoreProvider和fsBlockStore。
	* config.go，结构体Conf，blockStorage路径和块文件大小（默认最大64M）。
	* fs_blockstore.go，BlockStore接口实现，即fsBlockStore，主要为封装blockfileMgr。
	* fs_blockstore_provider.go，BlockStoreProvider接口实现，即FsBlockstoreProvider。
	
blockfile更详细内容，参考：[Fabric 1.0源码旅程 之 blockfile（区块文件存储）](../blockfile/README.md)。

## 1、核心接口定义

BlockStoreProvider接口定义：提供BlockStore句柄。

```go
type BlockStoreProvider interface {
	CreateBlockStore(ledgerid string) (BlockStore, error) //创建并打开BlockStore
	OpenBlockStore(ledgerid string) (BlockStore, error) //创建并打开BlockStore
	Exists(ledgerid string) (bool, error) //ledgerid的Blockstore目录是否存在
	List() ([]string, error) //获取已存在的ledgerid列表
	Close() //关闭BlockStore
}
//代码在common/ledger/blkstorage/blockstorage.go
```

BlockStore接口定义：

```go
type BlockStore interface {
	AddBlock(block *common.Block) error //添加块
	GetBlockchainInfo() (*common.BlockchainInfo, error) //获取区块链当前信息
	RetrieveBlocks(startNum uint64) (ledger.ResultsIterator, error) //获取区块链迭代器，可以循环遍历区块
	RetrieveBlockByHash(blockHash []byte) (*common.Block, error) //根据区块哈希获取块
	RetrieveBlockByNumber(blockNum uint64) (*common.Block, error) //根据区块链高度获取块
	RetrieveTxByID(txID string) (*common.Envelope, error) //根据交易ID获取交易
	RetrieveTxByBlockNumTranNum(blockNum uint64, tranNum uint64) (*common.Envelope, error) //根据区块链高度和tranNum获取交易
	RetrieveBlockByTxID(txID string) (*common.Block, error) //根据交易ID获取块
	RetrieveTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error) //根据交易ID获取交易验证代码
	Shutdown() //关闭BlockStore
}
//代码在common/ledger/blkstorage/blockstorage.go
```

## 2、Conf

Conf定义如下：

```go
type Conf struct {
	blockStorageDir  string //blockStorage路径
	maxBlockfileSize int //块文件大小（默认最大64M）
}

func NewConf(blockStorageDir string, maxBlockfileSize int) *Conf //构造Conf
func (conf *Conf) getIndexDir() string //获取index路径，即/var/hyperledger/production/ledgersData/chains/index
func (conf *Conf) getChainsDir() string //获取chains路径，即/var/hyperledger/production/ledgersData/chains/chains
func (conf *Conf) getLedgerBlockDir(ledgerid string) string //获取Ledger Block，如/var/hyperledger/production/ledgersData/chains/chains/mychannel
//代码在common/ledger/blkstorage/fsblkstorage/config.go
```

## 3、BlockStore接口实现

BlockStore接口基于文件系统实现，即fsBlockStore结构体及方法，BlockStore结构体定义如下：

```go
type fsBlockStore struct {
	id      string //即ledgerid
	conf    *Conf //type Conf struct
	fileMgr *blockfileMgr //区块文件存储
}
//代码在common/ledger/blkstorage/fsblkstorage/fs_blockstore.go
```

涉及方法如下：

```go
//构造fsBlockStore
func newFsBlockStore(id string, conf *Conf, indexConfig *blkstorage.IndexConfig, dbHandle *leveldbhelper.DBHandle) *fsBlockStore
//添加块，store.fileMgr.addBlock(block)
func (store *fsBlockStore) AddBlock(block *common.Block) error
//获取区块链当前信息，store.fileMgr.getBlockchainInfo()
func (store *fsBlockStore) GetBlockchainInfo() (*common.BlockchainInfo, error)
//获取区块链迭代器，可以循环遍历区块，store.fileMgr.retrieveBlocks(startNum)
func (store *fsBlockStore) RetrieveBlocks(startNum uint64) (ledger.ResultsIterator, error)
//根据区块哈希获取块，store.fileMgr.retrieveBlockByHash(blockHash)
func (store *fsBlockStore) RetrieveBlockByHash(blockHash []byte) (*common.Block, error)
//根据区块链高度获取块，store.fileMgr.retrieveBlockByNumber(blockNum)
func (store *fsBlockStore) RetrieveBlockByNumber(blockNum uint64) (*common.Block, error)
//根据交易ID获取交易，store.fileMgr.retrieveTransactionByID(txID)
func (store *fsBlockStore) RetrieveTxByID(txID string) (*common.Envelope, error) 
//根据区块链高度和tranNum获取交易，store.fileMgr.retrieveTransactionByBlockNumTranNum(blockNum, tranNum)
func (store *fsBlockStore) RetrieveTxByBlockNumTranNum(blockNum uint64, tranNum uint64) (*common.Envelope, error)
//根据交易ID获取块，store.fileMgr.retrieveBlockByTxID(txID)
func (store *fsBlockStore) RetrieveBlockByTxID(txID string) (*common.Block, error) 
//根据交易ID获取交易验证代码，store.fileMgr.retrieveTxValidationCodeByTxID(txID)
func (store *fsBlockStore) RetrieveTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error) 
//关闭BlockStore，store.fileMgr.close()
func (store *fsBlockStore) Shutdown() 
//代码在common/ledger/blkstorage/fsblkstorage/fs_blockstore.go
```

## 4、BlockStoreProvider接口实现

BlockStoreProvider接口实现，即NewProvider结构体及方法。NewProvider结构体定义如下：

```go
type FsBlockstoreProvider struct {
	conf            *Conf
	indexConfig     *blkstorage.IndexConfig
	leveldbProvider *leveldbhelper.Provider //用于操作index
}
//代码在common/ledger/blkstorage/fsblkstorage/fs_blockstore_provider.go
```

涉及方法：

```go
//构造FsBlockstoreProvider
func NewProvider(conf *Conf, indexConfig *blkstorage.IndexConfig) blkstorage.BlockStoreProvider 
//创建并打开BlockStore，同p.OpenBlockStore(ledgerid)
func (p *FsBlockstoreProvider) CreateBlockStore(ledgerid string) (blkstorage.BlockStore, error) 
//创建并打开BlockStore，调取newFsBlockStore(ledgerid, p.conf, p.indexConfig, indexStoreHandle)，即构造fsBlockStore
func (p *FsBlockstoreProvider) OpenBlockStore(ledgerid string) (blkstorage.BlockStore, error) 
//ledgerid的Blockstore目录是否存在，如/var/hyperledger/production/ledgersData/chains/chains/mychannel
func (p *FsBlockstoreProvider) Exists(ledgerid string) (bool, error) 
//获取已存在的ledgerid列表，util.ListSubdirs(p.conf.getChainsDir())
func (p *FsBlockstoreProvider) List() ([]string, error) 
//关闭BlockStore，目前仅限关闭p.leveldbProvider.Close()
func (p *FsBlockstoreProvider) Close() 
//代码在common/ledger/blkstorage/fsblkstorage/fs_blockstore_provider.go
```