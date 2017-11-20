# Fabric 1.0源码旅程 之 blockfile（区块文件存储）

## 1、blockfile概述

blockfile，即Fabric区块链区块文件存储。

blockfile，相关代码集中在common/ledger/blkstorage/fsblkstorage目录，目录结构如下：

* blockfile_mgr.go，blockfileMgr和checkpointInfo结构体及方法。

## 2、blockfileMgr结构体定义及方法

blockfileMgr结构体定义：

```go
type blockfileMgr struct {
	rootDir           string
	conf              *Conf
	db                *leveldbhelper.DBHandle
	index             index
	cpInfo            *checkpointInfo
	cpInfoCond        *sync.Cond
	currentFileWriter *blockfileWriter
	bcInfo            atomic.Value
}
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

涉及方法如下：




   117 blockfile_helper.go
   696 blockfile_mgr.go
    94 blockfile_rw.go
   381 blockindex.go
   218 block_serialization.go
   101 blocks_itr.go
   209 block_stream.go
    54 config.go