# Fabric 1.0源码旅程 之 LevelDB（KV数据库）

## 1、LevelDB概述

LevelDB是Google开源的持久化KV单机数据库，具有很高的随机写，顺序读/写性能，但是随机读的性能很一般，也就是说，LevelDB很适合应用在查询较少，而写很多的场景。

LevelDB的特点：
* key和value都是任意长度的字节数组；
* entry（即一条K-V记录）默认是按照key的字典顺序存储的，当然开发者也可以重载这个排序函数；
* 提供的基本操作接口：Put()、Delete()、Get()、Batch()；
* 支持批量操作以原子操作进行；
* 可以创建数据全景的snapshot(快照)，并允许在快照中查找数据；
* 可以通过前向（或后向）迭代器遍历数据（迭代器会隐含的创建一个snapshot）；
* 自动使用Snappy压缩数据；
* 可移植性；

Fabric中使用了goleveldb包，即https://github.com/syndtr/goleveldb/。

goleveldb的基本操作：

* 打开数据库，db, err:=leveldb.OpenFile("./db", nil)。作用就是在当前目录下创建一个db文件夹作为数据库的目录。
* 存储键值，db.Put([]byte("key1"),[]byte("value1"),nil)。作用就是在数据库中存储键值对 key1-value1。leveldb数据库中对键值的操作都是byte格式化的数据。
* 获取键值对，data,_ := db.Get([]byte("key1"),nil)，获取key1对应的值。
* 遍历数据库，iter := db.NewIterator(nil, nil)，for iter.Next(){ fmt.Printf("key=%s,value=%s\n",iter.Key(),iter.Value()) }，iter.Release()。作用就是建立迭代器iter，然后依次遍历数据库中所有的数据并打印键和值，最后释放迭代器iter。
* 关闭数据库，db.Close()。

Fabric中LevelDB代码，分布在common/ledger/util/leveldbhelper目录，目录结构如下：

* leveldb_provider.go，定义了结构体Provider、Provider、UpdateBatch、Iterator及其方法。
* leveldb_helper.go，定义了DB结构体及方法。

## 2、DB结构体及方法

DB结构体定义：对实际数据存储的包装。

```go
type Conf struct {
	DBPath string //路径
}

type DB struct {
	conf    *Conf //配置
	db      *leveldb.DB //leveldb.DB对象
	dbState dbState //type dbState int32
	mux     sync.Mutex //锁

	readOpts        *opt.ReadOptions
	writeOptsNoSync *opt.WriteOptions
	writeOptsSync   *opt.WriteOptions
}
//代码在common/ledger/util/leveldbhelper/leveldb_helper.go
```

涉及如下方法：对goleveldb包做了封装。

```go
func CreateDB(conf *Conf) *DB //创建DB实例
func (dbInst *DB) Open() //leveldb.OpenFile，创建并打开leveldb数据库（如目录不存在则创建）
func (dbInst *DB) Close() //db.Close()
func (dbInst *DB) Get(key []byte) ([]byte, error) //db.Get
func (dbInst *DB) Put(key []byte, value []byte, sync bool) error //db.Put
func (dbInst *DB) Delete(key []byte, sync bool) error //db.Delete
func (dbInst *DB) GetIterator(startKey []byte, endKey []byte) iterator.Iterator //db.NewIterator，创建迭代器
func (dbInst *DB) WriteBatch(batch *leveldb.Batch, sync bool) error //db.Write，批量写入
//代码在common/ledger/util/leveldbhelper/leveldb_helper.go
```

## 3、DBHandle结构体及方法

DBHandle结构体定义：封装DB，目的为给key添加dbName前缀，添加和拆除前缀通过constructLevelKey(h.dbName, key)和retrieveAppKey()实现。

```go
type DBHandle struct {
	dbName string //DB名称
	db     *DB //type DB struct
}
//代码在common/ledger/util/leveldbhelper/leveldb_provider.go
```

涉及如下方法：

```go
func (h *DBHandle) Get(key []byte) ([]byte, error) //h.db.Get
func (h *DBHandle) Put(key []byte, value []byte, sync bool) error //h.db.Put
func (h *DBHandle) Delete(key []byte, sync bool) error //h.db.Delete
func (h *DBHandle) WriteBatch(batch *UpdateBatch, sync bool) error //h.db.WriteBatch
func (h *DBHandle) GetIterator(startKey []byte, endKey []byte) *Iterator //h.db.GetIterator
//代码在common/ledger/util/leveldbhelper/leveldb_provider.go
```

补充UpdateBatch结构体及方法：

```go
type UpdateBatch struct {
	KVs map[string][]byte
}
func NewUpdateBatch() *UpdateBatch //构造UpdateBatch
func (batch *UpdateBatch) Put(key []byte, value []byte) //batch.KVs[string(key)] = value
func (batch *UpdateBatch) Delete(key []byte) //batch.KVs[string(key)] = nil
//代码在common/ledger/util/leveldbhelper/leveldb_provider.go
```

补充Iterator结构体及方法：封装github.com/syndtr/goleveldb/leveldb/iterator。

```go
type Iterator struct {
	iterator.Iterator
}
func (itr *Iterator) Key() []byte //itr.Iterator.Key()拆除dbName
func constructLevelKey(dbName string, key []byte) []byte //为key添加dbName
func retrieveAppKey(levelKey []byte) []byte //为key拆除dbName
//代码在common/ledger/util/leveldbhelper/leveldb_provider.go
```

## 4、Provider结构体及方法

Provider结构体定义：将单个物理LevelDB，虚拟为多个逻辑LevelDB

```go
type Provider struct {
	db        *DB
	dbHandles map[string]*DBHandle
	mux       sync.Mutex
}
//代码在common/ledger/util/leveldbhelper/leveldb_provider.go
```

涉及方法如下：

```go
func NewProvider(conf *Conf) *Provider {//创建并打开db，构造Provider
	db := CreateDB(conf)
	db.Open()
	return &Provider{db, make(map[string]*DBHandle), sync.Mutex{}}
}

//获取名称为dbName的leveldb句柄
func (p *Provider) GetDBHandle(dbName string) *DBHandle {
	p.mux.Lock()
	defer p.mux.Unlock()
	dbHandle := p.dbHandles[dbName]
	if dbHandle == nil {
		dbHandle = &DBHandle{dbName, p.db}
		p.dbHandles[dbName] = dbHandle
	}
	return dbHandle
}

//关闭leveldb
func (p *Provider) Close() {
	p.db.Close()
}
//代码在common/ledger/util/leveldbhelper/leveldb_provider.go
```

## 5、本文使用到的网络内容

* [LevelDB详解](http://blog.csdn.net/linuxheik/article/details/52768223)
* [Golang 之 key-value LevelDB](http://www.cnblogs.com/qufo/p/5701237.html)
* [fabric源码解析5——kvledger初始化](http://blog.csdn.net/idsuf698987/article/details/75388868)

