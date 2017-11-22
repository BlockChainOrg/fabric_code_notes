# Fabric 1.0源代码笔记 之 Ledger（4）historydb（历史数据库）

## 1、historydb概述

historydb代码分布在core/ledger/kvledger/history/historydb目录下，目录结构如下：

* historydb.go，定义核心接口HistoryDBProvider和HistoryDB。
* histmgr_helper.go，historydb工具函数。
* historyleveldb目录，historydb基于leveldb的实现。
	* historyleveldb.go，HistoryDBProvider和HistoryDB接口实现，即HistoryDBProvider和historyDB结构体及方法。
	* historyleveldb_query_executer.go，定义LevelHistoryDBQueryExecutor和historyScanner结构体及方法。

## 2、核心接口定义

HistoryDBProvider接口定义：

```go
type HistoryDBProvider interface {
	GetDBHandle(id string) (HistoryDB, error) //获取HistoryDB
	Close() //关闭所有HistoryDB
}
//代码在core/ledger/kvledger/history/historydb/historydb.go
```

HistoryDB接口定义：

```go
type HistoryDB interface {
	//创建ledger.HistoryQueryExecutor
	NewHistoryQueryExecutor(blockStore blkstorage.BlockStore) (ledger.HistoryQueryExecutor, error)
	Commit(block *common.Block) error
	GetLastSavepoint() (*version.Height, error)
	ShouldRecover(lastAvailableBlock uint64) (bool, uint64, error)
	CommitLostBlock(block *common.Block) error
}
//代码在core/ledger/kvledger/history/historydb/historydb.go
```

补充ledger.HistoryQueryExecutor定义：执行历史记录查询。

```go
type HistoryQueryExecutor interface {
	GetHistoryForKey(namespace string, key string) (commonledger.ResultsIterator, error) //按key查历史记录
}
//代码在core/ledger/ledger_interface.go
```

## 3、historydb工具函数

```go
//构造复合HistoryKey，ns 0x00 key 0x00 blocknum trannum
func ConstructCompositeHistoryKey(ns string, key string, blocknum uint64, trannum uint64) []byte
//构造部分复合HistoryKey，ns 0x00 key 0x00 0xff
func ConstructPartialCompositeHistoryKey(ns string, key string, endkey bool) []byte 
//按分隔符separator，分割bytesToSplit
func SplitCompositeHistoryKey(bytesToSplit []byte, separator []byte) ([]byte, []byte) 
//代码在core/ledger/kvledger/history/historydb/histmgr_helper.go
```

## 4、HistoryDB接口实现

HistoryDB接口实现，即historyDB结构体及方法。historyDB结构体定义如下：

```go
type historyDB struct {
	db     *leveldbhelper.DBHandle //leveldb
	dbName string //dbName
}
//代码在core/ledger/kvledger/history/historydb/historyleveldb/historyleveldb.go
```

涉及方法如下：

```go
//构造historyDB
func newHistoryDB(db *leveldbhelper.DBHandle, dbName string) *historyDB
//do nothing
func (historyDB *historyDB) Open() error
//do nothing
func (historyDB *historyDB) Close()
//
func (historyDB *historyDB) Commit(block *common.Block) error
func (historyDB *historyDB) NewHistoryQueryExecutor(blockStore blkstorage.BlockStore) (ledger.HistoryQueryExecutor, error)
func (historyDB *historyDB) GetLastSavepoint() (*version.Height, error)
func (historyDB *historyDB) ShouldRecover(lastAvailableBlock uint64) (bool, uint64, error)
func (historyDB *historyDB) CommitLostBlock(block *common.Block) error
//代码在core/ledger/kvledger/history/historydb/historyleveldb/historyleveldb.go
```

func (historyDB *historyDB) Commit(block *common.Block) error代码如下：

```go
blockNo := block.Header.Number //区块编号
var tranNo uint64 //交易编号，初始化值为0
dbBatch := leveldbhelper.NewUpdateBatch() //leveldb批量更新

//交易验证代码，type TxValidationFlags []uint8
//交易筛选器
txsFilter := util.TxValidationFlags(block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER])
if len(txsFilter) == 0 {
	txsFilter = util.NewTxValidationFlags(len(block.Data.Data))
	block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER] = txsFilter
}
for _, envBytes := range block.Data.Data {
	if txsFilter.IsInvalid(int(tranNo)) { //检查指定的交易是否有效
		tranNo++
		continue
	}
	//[]byte反序列化为Envelope
	env, err := putils.GetEnvelopeFromBlock(envBytes)
	payload, err := putils.GetPayload(env) //e.Payload反序列化为Payload
	//[]byte反序列化为ChannelHeader
	chdr, err := putils.UnmarshalChannelHeader(payload.Header.ChannelHeader)

	if common.HeaderType(chdr.Type) == common.HeaderType_ENDORSER_TRANSACTION { //背书交易，type HeaderType int32
		respPayload, err := putils.GetActionFromEnvelope(envBytes) //获取ChaincodeAction
		txRWSet := &rwsetutil.TxRwSet{}
		err = txRWSet.FromProtoBytes(respPayload.Results)
		for _, nsRWSet := range txRWSet.NsRwSets {
			ns := nsRWSet.NameSpace
			for _, kvWrite := range nsRWSet.KvRwSet.Writes {
				writeKey := kvWrite.Key
				compositeHistoryKey := historydb.ConstructCompositeHistoryKey(ns, writeKey, blockNo, tranNo)
				dbBatch.Put(compositeHistoryKey, emptyValue)
			}
		}
	} else {
		logger.Debugf("Skipping transaction [%d] since it is not an endorsement transaction\n", tranNo)
	}
	tranNo++
}

height := version.NewHeight(blockNo, tranNo)
dbBatch.Put(savePointKey, height.ToBytes())
err := historyDB.db.WriteBatch(dbBatch, true)
//代码在core/ledger/kvledger/history/historydb/historyleveldb/historyleveldb.go
```
