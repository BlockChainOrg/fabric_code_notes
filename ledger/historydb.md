# Fabric 1.0源代码笔记 之 Ledger #historydb（历史数据库）

## 1、historydb概述

historydb，用于存储所有块读写集中写集的内容。
代码分布在core/ledger/kvledger/history/historydb目录下，目录结构如下：

* historydb.go，定义核心接口HistoryDBProvider和HistoryDB。
* histmgr_helper.go，historydb工具函数。
* historyleveldb目录，historydb基于leveldb的实现。
	* historyleveldb.go，HistoryDBProvider和HistoryDB接口实现，即historyleveldb.HistoryDBProvider和historyleveldb.historyDB结构体及方法。
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
	//构造 LevelHistoryDBQueryExecutor
	NewHistoryQueryExecutor(blockStore blkstorage.BlockStore) (ledger.HistoryQueryExecutor, error)
	//提交Block入historyDB
	Commit(block *common.Block) error
	//获取savePointKey，即version.Height
	GetLastSavepoint() (*version.Height, error)
	//是否应该恢复，比较lastAvailableBlock和Savepoint
	ShouldRecover(lastAvailableBlock uint64) (bool, uint64, error)
	//提交丢失的块
	CommitLostBlock(block *common.Block) error
}
//代码在core/ledger/kvledger/history/historydb/historydb.go
```

补充ledger.HistoryQueryExecutor接口定义：执行历史记录查询。

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

HistoryDB接口实现，即historyleveldb.historyDB结构体及方法。historyDB结构体定义如下：

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
//提交Block入historyDB，将读写集中写集入库，并更新savePointKey
func (historyDB *historyDB) Commit(block *common.Block) error
//构造 LevelHistoryDBQueryExecutor
func (historyDB *historyDB) NewHistoryQueryExecutor(blockStore blkstorage.BlockStore) (ledger.HistoryQueryExecutor, error)
获取savePointKey，即version.Height
func (historyDB *historyDB) GetLastSavepoint() (*version.Height, error)
//是否应该恢复，比较lastAvailableBlock和Savepoint
func (historyDB *historyDB) ShouldRecover(lastAvailableBlock uint64) (bool, uint64, error)
//提交丢失的块，即调用historyDB.Commit(block)
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
		err = txRWSet.FromProtoBytes(respPayload.Results) //[]byte反序列化后构造NsRwSet，加入txRWSet.NsRwSets
		for _, nsRWSet := range txRWSet.NsRwSets {
			ns := nsRWSet.NameSpace
			for _, kvWrite := range nsRWSet.KvRwSet.Writes {
				writeKey := kvWrite.Key
				//txRWSet中写集入库
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

Tx（Transaction 交易）相关更详细内容，参考：[Fabric 1.0源代码笔记 之 Tx（Transaction 交易）](../tx/README.md)

## 5、HistoryDBProvider接口实现

HistoryDBProvider接口实现，即historyleveldb.HistoryDBProvider结构体和方法。

```go
type HistoryDBProvider struct {
	dbProvider *leveldbhelper.Provider
}

//构造HistoryDBProvider
func NewHistoryDBProvider() *HistoryDBProvider
//获取HistoryDB
func (provider *HistoryDBProvider) GetDBHandle(dbName string) (historydb.HistoryDB, error)
//关闭所有HistoryDB句柄，调取provider.dbProvider.Close()
func (provider *HistoryDBProvider) Close()
//代码在core/ledger/kvledger/history/historydb/historyleveldb/historyleveldb.go
```

## 6、LevelHistoryDBQueryExecutor和historyScanner结构体及方法

LevelHistoryDBQueryExecutor结构体及方法：实现ledger.HistoryQueryExecutor接口。

```go
type LevelHistoryDBQueryExecutor struct {
	historyDB  *historyDB
	blockStore blkstorage.BlockStore //用于传递给historyScanner
}
//按key查historyDB，调用q.historyDB.db.GetIterator(compositeStartKey, compositeEndKey)
func (q *LevelHistoryDBQueryExecutor) GetHistoryForKey(namespace string, key string) (commonledger.ResultsIterator, error) 
//代码在core/ledger/kvledger/history/historydb/historyleveldb/historyleveldb_query_executer.go
```

historyScanner结构体及方法：实现ledger.ResultsIterator接口。

```go
type historyScanner struct {
	compositePartialKey []byte //ns 0x00 key 0x00
	namespace           string
	key                 string
	dbItr               iterator.Iterator //leveldb迭代器
	blockStore          blkstorage.BlockStore
}

//构造historyScanner
func newHistoryScanner(compositePartialKey []byte, namespace string, key string, dbItr iterator.Iterator, blockStore blkstorage.BlockStore) *historyScanner
//按迭代器中key取blockNum和tranNum，再按blockNum和tranNum从blockStore中取Envelope，然后从Envelope的txRWSet.NsRwSets中按key查找并构造queryresult.KeyModification
func (scanner *historyScanner) Next() (commonledger.QueryResult, error)
func (scanner *historyScanner) Close() //scanner.dbItr.Release()
从Envelope的txRWSet.NsRwSets中按key查找并构造queryresult.KeyModification
func getKeyModificationFromTran(tranEnvelope *common.Envelope, namespace string, key string) (commonledger.QueryResult, error)
//代码在core/ledger/kvledger/history/historydb/historyleveldb/historyleveldb_query_executer.go
```

补充queryresult.KeyModification：

```go
type KeyModification struct {
	TxId      string //交易ID，ChannelHeader.TxId
	Value     []byte //读写集中Value，KVWrite.Value
	Timestamp *google_protobuf.Timestamp //ChannelHeader.Timestamp
	IsDelete  bool //KVWrite.IsDelete
}
//代码在protos/ledger/queryresult/kv_query_result.pb.go
```
