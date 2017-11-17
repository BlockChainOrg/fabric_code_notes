# Fabric 1.0源码旅程 之 Ledger（账本）

## 1、Ledger概述

Ledger，即账本数据库。Fabric中有四种账本数据库，idStore（ledgerID数据库）、blkstorage（block数据库）、statedb（状态数据库）、historydb（历史数据库）。
其中statedb可选择使用leveldb或couchDB外，其他三种均使用leveldb实现。

Ledger相关代码分布在core/ledger、common/ledger和protos/ledger目录下。目录结构如下：

* core/ledger目录
	* ledger_interface.go，定义了接口PeerLedgerProvider、PeerLedger、ValidatedLedger、QueryExecutor、HistoryQueryExecutor和TxSimulator。

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


## 10、本文使用到的网络内容

* [fabric源码分析5–kvledger的初始化](http://blog.csdn.net/idsuf698987/article/details/75388868)