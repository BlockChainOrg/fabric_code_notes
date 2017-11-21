# Fabric 1.0源代码笔记 之 Ledger（3）statedb（状态数据库）

## 1、statedb概述

statedb，或VersionedDB，即状态数据库，存储了交易（transaction）日志中所有键的最新值，也称世界状态（world state）。
可选择基于leveldb或cauchdb实现。

statedb，代码分布在core/ledger/kvledger/txmgmt/statedb目录下，目录结构如下：
* statedb.go，定义了核心接口VersionedDBProvider、VersionedDB、ResultsIterator和QueryResult，以及UpdateBatch和nsIterator结构体及方法。
* util.go，包括工具函数EncodeValue和DecodeValue的实现。
* stateleveldb目录，VersionedDBProvider和VersionedDB接口的leveldb版本实现，即stateleveldb.VersionedDBProvider和stateleveldb.versionedDB结构体及方法。
* statecouchdb目录，VersionedDBProvider和VersionedDB接口的couchdb版本实现，即statecouchdb.VersionedDBProvider和statecouchdb.VersionedDB结构体及方法。

## 2、核心接口定义

VersionedDBProvider接口定义：

```go
type VersionedDBProvider interface {
	GetDBHandle(id string) (VersionedDB, error) //获取VersionedDB句柄
	Close() //关闭所有 VersionedDB 实例
}
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go
```

VersionedDB接口定义：

```go
type VersionedDB interface {
	//获取给定命名空间和键的值
	GetState(namespace string, key string) (*VersionedValue, error)
	//在单个调用中获取多个键的值
	GetStateMultipleKeys(namespace string, keys []string) ([]*VersionedValue, error)
	//返回一个迭代器, 其中包含给定键范围之间的所有键值（包括startKey，不包括endKey）
	GetStateRangeScanIterator(namespace string, startKey string, endKey string) (ResultsIterator, error)
	//执行给定的查询并返回迭代器
	ExecuteQuery(namespace, query string) (ResultsIterator, error)
	//批处理应用
	ApplyUpdates(batch *UpdateBatch, height *version.Height) error
	//返回statedb一致的最高事务的高度
	GetLatestSavePoint() (*version.Height, error)
	//测试数据库是否支持这个key（leveldb支持任何字节, 而couchdb只支持utf-8字符串）
	ValidateKey(key string) error
	//打开db
	Open() error
	//关闭db
	Close()
}
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go
```

ResultsIterator和QueryResult接口定义：

```go
type ResultsIterator interface {
	Next() (QueryResult, error)
	Close()
}

type QueryResult interface{}
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go
```

补充CompositeKey、VersionedValue和VersionedKV结构体：

```go
type CompositeKey struct {
	Namespace string //命名空间
	Key       string //Key
}

type VersionedValue struct {
	Value   []byte //Value
	Version *version.Height //版本
}

type VersionedKV struct {
	CompositeKey //嵌入CompositeKey
	VersionedValue //嵌入VersionedValue
}
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go
```

nsUpdates结构体及方法：

```go
type nsUpdates struct {
	m map[string]*VersionedValue //string为Key
}

func newNsUpdates() *nsUpdates//构造nsUpdates
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go
```

UpdateBatch结构体及方法：

```go
type UpdateBatch struct {
	updates map[string]*nsUpdates //string为Namespace
}

//构造UpdateBatch
func NewUpdateBatch() *UpdateBatch
//按namespace和key获取Value
func (batch *UpdateBatch) Get(ns string, key string) *VersionedValue
//按namespace和key添加Value
func (batch *UpdateBatch) Put(ns string, key string, value []byte, version *version.Height)
//按namespace和key删除Value，即置为nil
func (batch *UpdateBatch) Delete(ns string, key string, version *version.Height)
//按namespace和key查找是否存在
func (batch *UpdateBatch) Exists(ns string, key string) bool
//获取更新的namespace列表
func (batch *UpdateBatch) GetUpdatedNamespaces() []string
//按namespace获取nsUpdates
func (batch *UpdateBatch) GetUpdates(ns string) map[string]*VersionedValue
//构造nsIterator
func (batch *UpdateBatch) GetRangeScanIterator(ns string, startKey string, endKey string) ResultsIterator
//按namespace获取或创建nsUpdates
func (batch *UpdateBatch) getOrCreateNsUpdates(ns string) *nsUpdates
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go
```

nsIterator结构体及方法：

```go
type nsIterator struct {
	ns         string //namespace
	nsUpdates  *nsUpdates //batch.updates[ns]
	sortedKeys []string //nsUpdates.m中key排序
	nextIndex  int //startKey
	lastIndex  int //endKey
}

//构造nsIterator
func newNsIterator(ns string, startKey string, endKey string, batch *UpdateBatch) *nsIterator
func (itr *nsIterator) Next() (QueryResult, error) //按itr.nextIndex获取VersionedKV
func (itr *nsIterator) Close() // do nothing
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go
```

## 3、statedb基于leveldb实现

### 3.1、VersionedDB接口实现

VersionedDB接口实现，即versionedDB结构体，定义如下：

```go
type versionedDB struct {
	db     *leveldbhelper.DBHandle //leveldb
	dbName string //dbName
}
//代码在core/ledger/kvledger/txmgmt/statedb/stateleveldb/stateleveldb.go
```

涉及方法如下：

```go
//构造versionedDB
func newVersionedDB(db *leveldbhelper.DBHandle, dbName string) *versionedDB
func (vdb *versionedDB) Open() error // do nothing
func (vdb *versionedDB) Close() // do nothing
func (vdb *versionedDB) ValidateKey(key string) error // do nothing
//按namespace和key获取Value
func (vdb *versionedDB) GetState(namespace string, key string) (*statedb.VersionedValue, error)
//在单个调用中获取多个键的值
func (vdb *versionedDB) GetStateMultipleKeys(namespace string, keys []string) ([]*statedb.VersionedValue, error)
//返回一个迭代器, 其中包含给定键范围之间的所有键值（包括startKey，不包括endKey）
func (vdb *versionedDB) GetStateRangeScanIterator(namespace string, startKey string, endKey string) (statedb.ResultsIterator, error)
//leveldb不支持ExecuteQuery方法
func (vdb *versionedDB) ExecuteQuery(namespace, query string) (statedb.ResultsIterator, error)
//批处理应用
func (vdb *versionedDB) ApplyUpdates(batch *statedb.UpdateBatch, height *version.Height) error
//返回statedb一致的最高事务的高度
func (vdb *versionedDB) GetLatestSavePoint() (*version.Height, error)
//拼接ns和key，ns []byte{0x00} key
func constructCompositeKey(ns string, key string) []byte
//分割ns和key，分割符[]byte{0x00}
func splitCompositeKey(compositeKey []byte) (string, string)
//代码在core/ledger/kvledger/txmgmt/statedb/stateleveldb/stateleveldb.go
```

func (vdb *versionedDB) ApplyUpdates(batch *statedb.UpdateBatch, height *version.Height) error代码如下：

```go
dbBatch := leveldbhelper.NewUpdateBatch()
namespaces := batch.GetUpdatedNamespaces() //获取更新的namespace列表
for _, ns := range namespaces {
	updates := batch.GetUpdates(ns) //按namespace获取nsUpdates
	for k, vv := range updates {
		compositeKey := constructCompositeKey(ns, k) //拼接ns和key
		if vv.Value == nil {
			dbBatch.Delete(compositeKey)
		} else {
			dbBatch.Put(compositeKey, statedb.EncodeValue(vv.Value, vv.Version))
		}
	}
}
//statedb一致的最高事务的高度
dbBatch.Put(savePointKey, height.ToBytes()) //var savePointKey = []byte{0x00}
err := vdb.db.WriteBatch(dbBatch, true)
//代码在core/ledger/kvledger/txmgmt/statedb/stateleveldb/stateleveldb.go
```

### 3.2、ResultsIterator接口实现

ResultsIterator接口实现，即kvScanner结构体及方法。

```go
type kvScanner struct {
	namespace string
	dbItr     iterator.Iterator
}

//构造kvScanner
func newKVScanner(namespace string, dbItr iterator.Iterator) *kvScanner
//迭代获取statedb.VersionedKV
func (scanner *kvScanner) Next() (statedb.QueryResult, error)
func (scanner *kvScanner) Close() //释放迭代器
//代码在core/ledger/kvledger/txmgmt/statedb/stateleveldb/stateleveldb.go
```

### 3.3、VersionedDBProvider接口实现

VersionedDBProvider接口实现，即VersionedDBProvider结构体及方法。

```go
type VersionedDBProvider struct {
	dbProvider *leveldbhelper.Provider
}

func NewVersionedDBProvider() *VersionedDBProvider //构造VersionedDBProvider
//获取statedb.VersionedDB
func (provider *VersionedDBProvider) GetDBHandle(dbName string) (statedb.VersionedDB, error)
func (provider *VersionedDBProvider) Close() //关闭statedb.VersionedDB
//代码在core/ledger/kvledger/txmgmt/statedb/stateleveldb/stateleveldb.go
```

## 4、statedb基于cauchdb实现

暂略，待补充。