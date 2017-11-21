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
//
func (batch *UpdateBatch) Get(ns string, key string) *VersionedValue
func (batch *UpdateBatch) Put(ns string, key string, value []byte, version *version.Height)
func (batch *UpdateBatch) Delete(ns string, key string, version *version.Height)
func (batch *UpdateBatch) Exists(ns string, key string) bool
func (batch *UpdateBatch) GetUpdatedNamespaces() []string
func (batch *UpdateBatch) GetUpdates(ns string) map[string]*VersionedValue
func (batch *UpdateBatch) GetRangeScanIterator(ns string, startKey string, endKey string) ResultsIterator
func (batch *UpdateBatch) getOrCreateNsUpdates(ns string) *nsUpdates
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go
```