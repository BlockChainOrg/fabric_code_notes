# Fabric 1.0源码旅程 之 Ledger（1）idStore（ledgerID数据库）

## 1、idStore概述

* Fabric支持创建多个Ledger，不同Ledger以ledgerID区分。
* 多个ledgerID及其创世区块存储在idStore数据库中，idStore数据库基于leveldb实现。
* idStore默认使用路径：/var/hyperledger/production/ledgersData/ledgerProvider/。
* idStore库中特殊key "underConstructionLedgerKey"，用于标志最新在建的ledgerID，ledgerID创建成功后或失败时该标志将清除，另外此标志也用于异常时按ledgerID恢复数据。

## 2、idStore结构体定义

leveldbhelper更详细内容，参考：[Fabric 1.0源码旅程 之 LevelDB（KV数据库）](../leveldb/README.md)

```go
type idStore struct {
	db *leveldbhelper.DB
}
//代码在core/ledger/kvledger/kv_ledger_provider.go
```

## 3、idStore方法定义

```go
func openIDStore(path string) *idStore //按path创建并打开leveldb数据库
func (s *idStore) setUnderConstructionFlag(ledgerID string) error //设置ledgerID在建标志，将key为"underConstructionLedgerKey"，value为ledgerID写入库
func (s *idStore) unsetUnderConstructionFlag() error //取消ledgerID在建标志（确认构建失败时），删除key"underConstructionLedgerKey"
func (s *idStore) getUnderConstructionFlag() (string, error) //获取ledgerID在建标志（按ledgerID恢复时），按key"underConstructionLedgerKey"，取ledgerID
func (s *idStore) createLedgerID(ledgerID string, gb *common.Block) error //创建LedgerID，即以ledgerID为key，将创世区块写入库
func (s *idStore) ledgerIDExists(ledgerID string) (bool, error) //查找ledgerID是否存在，即查库中key为ledgerID是否存在
func (s *idStore) getAllLedgerIds() ([]string, error) //获取ledgerID列表
func (s *idStore) close() //关闭idStore leveldb数据库
func (s *idStore) encodeLedgerKey(ledgerID string) []byte //为ledgerID添加前缀即"l"
func (s *idStore) decodeLedgerID(key []byte) string //解除ledgerID前缀
//代码在core/ledger/kvledger/kv_ledger_provider.go
```

func (s *idStore) createLedgerID(ledgerID string, gb *common.Block) error代码如下：
将ledgerID和Block入库，并清除ledgerID在建标志。

```go
func (s *idStore) createLedgerID(ledgerID string, gb *common.Block) error {
	key := s.encodeLedgerKey(ledgerID) //为ledgerID添加前缀即"l"
	var val []byte
	var err error
	if val, err = proto.Marshal(gb); err != nil { //Block序列化
		return err
	}
	if val, err = s.db.Get(key); err != nil {
		return err
	}
	if val != nil {
		return ErrLedgerIDExists //ledgerID已存在
	}
	batch := &leveldb.Batch{}
	batch.Put(key, val) //ledgerID和Block入库
	batch.Delete(underConstructionLedgerKey) //清除ledgerID在建标志
	return s.db.WriteBatch(batch, true) //提交执行
}
//代码在core/ledger/kvledger/kv_ledger_provider.go
```

