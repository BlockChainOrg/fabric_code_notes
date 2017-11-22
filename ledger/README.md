# Fabric 1.0源代码笔记 之 Ledger（账本）

## 1、Ledger概述

Ledger，即账本数据库。Fabric账本中有四种数据库，idStore（ledgerID数据库）、blkstorage（block文件存储）、statedb（状态数据库）、historydb（历史数据库）。
其中idStore、historydb使用leveldb实现，statedb可选择使用leveldb或couchDB。而blkstorage中index部分使用leveldb实现，实际区块链数据存储使用文件实现。

* idStore，默认目录/var/hyperledger/production/ledgersData/ledgerProvider，更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（1）idStore（ledgerID数据库）](idstore.md)
* blkstorage，默认目录/var/hyperledger/production/ledgersData/chains，更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（2）blkstorage（block文件存储）](blkstorage.md)
* statedb，默认目录/var/hyperledger/production/ledgersData/stateLeveldb，更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（3）statedb（状态数据库）](statedb.md)
* historydb，默认目录/var/hyperledger/production/ledgersData/historyLeveldb，更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（4）historydb（历史数据库）](historydb.md)

## 2、Ledger代码目录结构

Ledger相关代码分布在common/ledger、core/ledger和protos/ledger目录下。目录结构如下：

* common/ledger目录
	* ledger_interface.go，定义了通用接口Ledger、ResultsIterator、以及QueryResult和PrunePolicy（暂时均为空接口）。
	* blkstorage目录，**blkstorage相关接口及实现**。
	* util/leveldbhelper目录，LevelDB数据库操作的封装。
	
* core/ledger目录
	* ledger_interface.go，定义了核心接口PeerLedgerProvider、PeerLedger、ValidatedLedger（暂时未定义）、QueryExecutor、HistoryQueryExecutor和TxSimulator。
	* kvledger目录，目前PeerLedgerProvider、PeerLedger等接口仅有一种实现即：kvledger。
		* kv_ledger_provider.go，实现PeerLedgerProvider接口，即Provider结构体及其方法，以及**idStore结构体及方法**。
		* kv_ledger.go，实现PeerLedger接口，即kvLedger结构体及方法。
		* txmgmt目录，交易管理。
			* statedb目录，**statedb相关接口及实现**。
		* history/historydb目录，**historydb相关接口及实现**。
	* ledgermgmt/ledger_mgmt.go，Ledger管理相关函数实现。
	* ledgerconfig/ledger_config.go，Ledger配置相关函数实现。
	* util目录，Ledger工具相关函数实现。
	

## 3、核心接口定义

PeerLedgerProvider接口定义：提供PeerLedger实例handle。

```go
type PeerLedgerProvider interface {
	Create(genesisBlock *common.Block) (PeerLedger, error) //用给定的创世纪块创建Ledger
	Open(ledgerID string) (PeerLedger, error) //打开已创建的Ledger
	Exists(ledgerID string) (bool, error) //按ledgerID查Ledger是否存在
	List() ([]string, error) //列出现有的ledgerID
	Close() //关闭 PeerLedgerProvider
}
//代码在core/ledger/ledger_interface.go
```

PeerLedger接口定义：
PeerLedger和OrdererLedger的不同之处在于PeerLedger本地维护位掩码，用于区分有效交易和无效交易。

```go
type PeerLedger interface {
	commonledger.Ledger //嵌入common/ledger/Ledger接口
	GetTransactionByID(txID string) (*peer.ProcessedTransaction, error) //按txID获取交易
	GetBlockByHash(blockHash []byte) (*common.Block, error) //按blockHash获取Block
	GetBlockByTxID(txID string) (*common.Block, error) //按txID获取包含交易的Block
	GetTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error) //获取交易记录验证的原因代码
	NewTxSimulator() (TxSimulator, error) //创建交易模拟器，客户端可以创建多个"TxSimulator"并行执行
	NewQueryExecutor() (QueryExecutor, error) //创建查询执行器，客户端可以创建多个'QueryExecutor'并行执行
	NewHistoryQueryExecutor() (HistoryQueryExecutor, error) //创建历史记录查询执行器，客户端可以创建多个'HistoryQueryExecutor'并行执行
	Prune(policy commonledger.PrunePolicy) error //裁剪满足给定策略的块或交易
}
//代码在core/ledger/ledger_interface.go
```

补充PeerLedger接口嵌入的commonledger.Ledger接口定义如下：

```go
type Ledger interface {
	GetBlockchainInfo() (*common.BlockchainInfo, error) //获取blockchain基本信息
	GetBlockByNumber(blockNumber uint64) (*common.Block, error) //按给定高度获取Block，给定math.MaxUint64将获取最新Block
	GetBlocksIterator(startBlockNumber uint64) (ResultsIterator, error) //获取从startBlockNumber开始的迭代器（包含startBlockNumber），迭代器是阻塞迭代，直到ledger中下一个block可用
	Close() //关闭ledger
	Commit(block *common.Block) error //提交新block
}
//代码在common/ledger/ledger_interface.go
```

ValidatedLedger接口暂未定义方法，从PeerLedger筛选出无效交易后，ValidatedLedger表示最终账本。暂时忽略。

QueryExecutor接口定义：用于执行查询。
其中Get*方法用于支持KV-based数据模型，ExecuteQuery方法用于支持更丰富的数据和查询支持。

```go
type QueryExecutor interface {
	GetState(namespace string, key string) ([]byte, error) //按namespace和key获取value，对于chaincode，chaincodeId即为namespace
	GetStateMultipleKeys(namespace string, keys []string) ([][]byte, error) //一次调用获取多个key的值
	//获取迭代器，返回包括startKey、但不包括endKeyd的之间所有值
	GetStateRangeScanIterator(namespace string, startKey string, endKey string) (commonledger.ResultsIterator, error)
	ExecuteQuery(namespace, query string) (commonledger.ResultsIterator, error) //执行查询并返回迭代器，仅用于查询statedb
	Done() //释放QueryExecutor占用的资源
}
//代码在core/ledger/ledger_interface.go
```

HistoryQueryExecutor接口定义：执行历史记录查询。

```go
type HistoryQueryExecutor interface {
	GetHistoryForKey(namespace string, key string) (commonledger.ResultsIterator, error) //按key查历史记录
}
//代码在core/ledger/ledger_interface.go
```

TxSimulator接口定义：在"尽可能"最新状态的一致快照上模拟交易。
其中Set*方法用于支持KV-based数据模型，ExecuteUpdate方法用于支持更丰富的数据和查询支持。

```go
type TxSimulator interface {
	QueryExecutor //嵌入QueryExecutor接口
	SetState(namespace string, key string, value []byte) error //按namespace和key写入value
	DeleteState(namespace string, key string) error //按namespace和key删除
	SetStateMultipleKeys(namespace string, kvs map[string][]byte) error //一次调用设置多个key的值
	ExecuteUpdate(query string) error //ExecuteUpdate用于支持丰富的数据模型
	GetTxSimulationResults() ([]byte, error) //获取模拟交易的结果
}
//代码在core/ledger/ledger_interface.go
```

## 4、kvledger.kvLedger结构体及方法（实现PeerLedger接口）

kvLedger结构体定义：

```go
type kvLedger struct {
	ledgerID   string //ledgerID
	blockStore blkstorage.BlockStore //blkstorage
	txtmgmt    txmgr.TxMgr //txmgr
	historyDB  historydb.HistoryDB //historyDB
}
//代码在core/ledger/kvledger/kv_ledger.go
```

涉及方法如下：

```go
//构造kvLedger
func newKVLedger(ledgerID string, blockStore blkstorage.BlockStore,versionedDB statedb.VersionedDB, historyDB historydb.HistoryDB) (*kvLedger, error)
func (l *kvLedger) recoverDBs() error
func (l *kvLedger) recommitLostBlocks(firstBlockNum uint64, lastBlockNum uint64, recoverables ...recoverable) error
func (l *kvLedger) GetTransactionByID(txID string) (*peer.ProcessedTransaction, error)
func (l *kvLedger) GetBlockchainInfo() (*common.BlockchainInfo, error)
func (l *kvLedger) GetBlockByNumber(blockNumber uint64) (*common.Block, error)
func (l *kvLedger) GetBlocksIterator(startBlockNumber uint64) (commonledger.ResultsIterator, error)
func (l *kvLedger) GetBlockByHash(blockHash []byte) (*common.Block, error)
func (l *kvLedger) GetBlockByTxID(txID string) (*common.Block, error)
func (l *kvLedger) GetTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error)
func (l *kvLedger) Prune(policy commonledger.PrunePolicy) error
func (l *kvLedger) NewTxSimulator() (ledger.TxSimulator, error)
func (l *kvLedger) NewQueryExecutor() (ledger.QueryExecutor, error)
func (l *kvLedger) NewHistoryQueryExecutor() (ledger.HistoryQueryExecutor, error)
func (l *kvLedger) Commit(block *common.Block) error
func (l *kvLedger) Close()
//代码在core/ledger/kvledger/kv_ledger.go
```

## 5、kvledger.Provider结构体及方法（实现PeerLedgerProvider接口）

Provider结构体定义：

```go
type Provider struct {
	idStore            *idStore //idStore
	blockStoreProvider blkstorage.BlockStoreProvider //blkstorage
	vdbProvider        statedb.VersionedDBProvider //statedb
	historydbProvider  historydb.HistoryDBProvider //historydb
}
//代码在core/ledger/kvledger/kv_ledger_provider.go
```

* idStore更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（1）idStore（ledgerID数据库）](idstore.md)
* blkstorage更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（2）blkstorage（block文件存储）](blkstorage.md)
* statedb更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（3）statedb（状态数据库）](statedb.md)
* historydb更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（4）historydb（历史数据库）](historydb.md)

涉及方法如下：

```go
//分别构造idStore、blockStoreProvider、vdbProvider和historydbProvider，并用于构造Provider，并恢复之前未完成创建的Ledger
func NewProvider() (ledger.PeerLedgerProvider, error)
func (provider *Provider) Create(genesisBlock *common.Block) (ledger.PeerLedger, error)
func (provider *Provider) Open(ledgerID string) (ledger.PeerLedger, error)
func (provider *Provider) openInternal(ledgerID string) (ledger.PeerLedger, error)
func (provider *Provider) Exists(ledgerID string) (bool, error)
func (provider *Provider) List() ([]string, error)
func (provider *Provider) Close()
//检查是否有之前未完成创建的Ledger，并恢复
func (provider *Provider) recoverUnderConstructionLedger()
func (provider *Provider) runCleanup(ledgerID string) error //暂时没有实现
func panicOnErr(err error, mgsFormat string, args ...interface{}) //panicOnErr
//代码在core/ledger/kvledger/kv_ledger_provider.go
```