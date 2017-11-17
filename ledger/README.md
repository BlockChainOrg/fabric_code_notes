# Fabric 1.0源码旅程 之 Ledger（账本）

## 1、Ledger概述

Ledger，即账本数据库。Fabric中有四种账本数据库，idStore（ledgerID数据库）、blkstorage（block数据库）、statedb（状态数据库）、historydb（历史数据库）。
其中statedb可选择使用leveldb或couchDB外，其他三种均使用leveldb实现。

idStore更详细内容，参考：[Fabric 1.0源码旅程 之 Ledger（1）idStore（ledgerID数据库）](idstore.md)

Ledger相关代码分布在common/ledger、core/ledger和protos/ledger目录下。目录结构如下：

* common/ledger目录
	* ledger_interface.go，定义了通用接口Ledger、ResultsIterator、以及QueryResult和PrunePolicy（暂时均为空接口）。
	* blkstorage目录
		* blockstorage.go，定义了通用接口 BlockStoreProvider和BlockStore。
		* fsblkstorage目录，实现了BlockStoreProvider和BlockStore接口。
* core/ledger目录
	* ledger_interface.go，定义了核心接口PeerLedgerProvider、PeerLedger、ValidatedLedger（暂时未定义）、QueryExecutor、HistoryQueryExecutor和TxSimulator。
	* kvledger目录，目前PeerLedgerProvider、PeerLedger等接口仅有一种实现即：kvledger。
		* kv_ledger_provider.go，实现PeerLedgerProvider接口，即Provider结构体及其方法，以及idStore结构体及方法。
		* kv_ledger.go，实现PeerLedger接口，即kvLedger结构体及方法。
		* txmgmt目录，交易管理。
	* ledgermgmt/ledger_mgmt.go，Ledger管理相关函数实现。
	* ledgerconfig/ledger_config.go，Ledger配置相关函数实现。
	* util目录，Ledger工具相关函数实现。
	

## 2、核心接口定义

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

## 3、kvledger.Provider结构体及方法（实现PeerLedgerProvider接口）

Provider结构体定义：

```go
type Provider struct {
	idStore            *idStore
	blockStoreProvider blkstorage.BlockStoreProvider
	vdbProvider        statedb.VersionedDBProvider
	historydbProvider  historydb.HistoryDBProvider
}
//代码在core/ledger/kvledger/kv_ledger_provider.go
```

idStore结构体定义：

type idStore struct {
	db *leveldbhelper.DB
}

## 10、本文使用到的网络内容

* [fabric源码分析5–kvledger的初始化](http://blog.csdn.net/idsuf698987/article/details/75388868)