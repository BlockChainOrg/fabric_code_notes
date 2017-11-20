# Fabric 1.0源码旅程 之 blockfile（区块文件存储）

## 1、blockfile概述

blockfile，即Fabric区块链区块文件存储，默认目录/var/hyperledger/production/ledgersData/chains，含index和chains两个子目录。
其中index为索引目录，采用leveldb实现。而chains为各ledger的区块链文件，子目录以ledgerid为名，使用文件系统实现。
区块文件以blockfile_为前缀，最大大小默认64M。

blockfile，相关代码集中在common/ledger/blkstorage/fsblkstorage目录，目录结构如下：

* blockfile_mgr.go，blockfileMgr和checkpointInfo结构体及方法。

## 2、checkpointInfo结构体定义及方法

checkpointInfo，即检查点信息，结构体定义如下：

```go
type checkpointInfo struct {
	latestFileChunkSuffixNum int //最新的区块文件后缀，如blockfile_000000
	latestFileChunksize      int //最新的区块文件大小
	isChainEmpty             bool //是否空链
	lastBlockNumber          uint64 //最新的区块编号
}
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

涉及方法如下：

```go
func (i *checkpointInfo) marshal() ([]byte, error) //checkpointInfo序列化
func (i *checkpointInfo) unmarshal(b []byte) error //checkpointInfo反序列化
func (i *checkpointInfo) String() string //转换为string
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

## 3、blockfileStream相关结构体及方法

### 3.1、blockfileStream

blockfileStream定义如下：

```go
type blockfileStream struct {
	fileNum       int //blockfile文件后缀
	file          *os.File //os.File
	reader        *bufio.Reader //bufio.Reader
	currentOffset int64 //当前偏移量
}
//代码在common/ledger/blkstorage/fsblkstorage/block_stream.go
```

涉及方法如下：

```go
//构造blockfileStream
func newBlockfileStream(rootDir string, fileNum int, startOffset int64) (*blockfileStream, error) 
func (s *blockfileStream) nextBlockBytes() ([]byte, error) //下一个块
//下一个块和位置信息
func (s *blockfileStream) nextBlockBytesAndPlacementInfo() ([]byte, *blockPlacementInfo, error) 
func (s *blockfileStream) close() error //关闭blockfileStream
//代码在common/ledger/blkstorage/fsblkstorage/block_stream.go
```

func (s *blockfileStream) nextBlockBytesAndPlacementInfo() ([]byte, *blockPlacementInfo, error) 代码如下：

```go
//代码在common/ledger/blkstorage/fsblkstorage/block_stream.go
```


## 4、blockfileMgr结构体定义及方法

blockfileMgr结构体定义：

```go
type blockfileMgr struct {
	rootDir           string //ledger文件存储目录，如/var/hyperledger/production/ledgersData/chains/chains/mychannel
	conf              *Conf //即type Conf struct，存放路径和区块文件大小
	db                *leveldbhelper.DBHandle //用于操作index
	index             index //type index interface，其实现为blockIndex结构体
	cpInfo            *checkpointInfo //type checkpointInfo struct
	cpInfoCond        *sync.Cond //定期唤醒锁
	currentFileWriter *blockfileWriter //type blockfileWriter struct
	bcInfo            atomic.Value //原子操作
}
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

涉及方法如下：

```go
//构建blockfileMgr，
func newBlockfileMgr(id string, conf *Conf, indexConfig *blkstorage.IndexConfig, indexStore *leveldbhelper.DBHandle) *blockfileMgr {
func syncCPInfoFromFS(rootDir string, cpInfo *checkpointInfo) {
func deriveBlockfilePath(rootDir string, suffixNum int) string //构造Blockfile路径
func (mgr *blockfileMgr) close() {
func (mgr *blockfileMgr) moveToNextFile() {
func (mgr *blockfileMgr) addBlock(block *common.Block) error {
func (mgr *blockfileMgr) syncIndex() error {
func (mgr *blockfileMgr) getBlockchainInfo() *common.BlockchainInfo {
func (mgr *blockfileMgr) updateCheckpoint(cpInfo *checkpointInfo) {
func (mgr *blockfileMgr) updateBlockchainInfo(latestBlockHash []byte, latestBlock *common.Block) {
func (mgr *blockfileMgr) retrieveBlockByHash(blockHash []byte) (*common.Block, error) {
func (mgr *blockfileMgr) retrieveBlockByNumber(blockNum uint64) (*common.Block, error) {
func (mgr *blockfileMgr) retrieveBlockByTxID(txID string) (*common.Block, error) {
func (mgr *blockfileMgr) retrieveTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error) {
func (mgr *blockfileMgr) retrieveBlockHeaderByNumber(blockNum uint64) (*common.BlockHeader, error) {
func (mgr *blockfileMgr) retrieveBlocks(startNum uint64) (*blocksItr, error) {
func (mgr *blockfileMgr) retrieveTransactionByID(txID string) (*common.Envelope, error) {
func (mgr *blockfileMgr) retrieveTransactionByBlockNumTranNum(blockNum uint64, tranNum uint64) (*common.Envelope, error) {
func (mgr *blockfileMgr) fetchBlock(lp *fileLocPointer) (*common.Block, error) {
func (mgr *blockfileMgr) fetchTransactionEnvelope(lp *fileLocPointer) (*common.Envelope, error) {
func (mgr *blockfileMgr) fetchBlockBytes(lp *fileLocPointer) ([]byte, error) {
func (mgr *blockfileMgr) fetchRawBytes(lp *fileLocPointer) ([]byte, error) {
func (mgr *blockfileMgr) loadCurrentInfo() (*checkpointInfo, error) //获取存储在index库中最新检查点信息，key为"blkMgrInfo"
func (mgr *blockfileMgr) saveCurrentInfo(i *checkpointInfo, sync bool) error //将最新检查点信息，序列化后存入index库
//扫描给定的块文件并检测文件中的最后一个偏移量
func scanForLastCompleteBlock(rootDir string, fileNum int, startingOffset int64) ([]byte, int64, int, error) {
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

func newBlockfileMgr(id string, conf *Conf, indexConfig *blkstorage.IndexConfig, indexStore *leveldbhelper.DBHandle) *blockfileMgr实现如下：
```go
rootDir := conf.getLedgerBlockDir(id) //如/var/hyperledger/production/ledgersData/chains/chains/mychannel
_, err := util.CreateDirIfMissing(rootDir) //检查rootDir是否存在，如不存在则创建
mgr := &blockfileMgr{rootDir: rootDir, conf: conf, db: indexStore} //构造blockfileMgr，包括ledger路径、块文件大小、index目录leveldb句柄
cpInfo, err := mgr.loadCurrentInfo() //获取存储在index库中最新检查点信息，key为"blkMgrInfo"
if cpInfo == nil { //找不到，第一次创建ledger或index被删除
	//扫描最新的blockfile，并重新构造检查点信息
	cpInfo, err = constructCheckpointInfoFromBlockFiles(rootDir)
} else {
	syncCPInfoFromFS(rootDir, cpInfo) //从文件系统中更新检查点信息
}
err = mgr.saveCurrentInfo(cpInfo, true) //将最新检查点信息，序列化后存入index库
currentFileWriter, err := newBlockfileWriter(deriveBlockfilePath(rootDir, cpInfo.latestFileChunkSuffixNum))
err = currentFileWriter.truncateFile(cpInfo.latestFileChunksize)
mgr.index = newBlockIndex(indexConfig, indexStore)
mgr.cpInfo = cpInfo
mgr.cpInfoCond = sync.NewCond(&sync.Mutex{})
mgr.syncIndex()

bcInfo := &common.BlockchainInfo{
	Height:            0,
	CurrentBlockHash:  nil,
	PreviousBlockHash: nil}
if !cpInfo.isChainEmpty {
	lastBlockHeader, err := mgr.retrieveBlockHeaderByNumber(cpInfo.lastBlockNumber)
	lastBlockHash := lastBlockHeader.Hash()
	previousBlockHash := lastBlockHeader.PreviousHash
	bcInfo = &common.BlockchainInfo{
		Height:            cpInfo.lastBlockNumber + 1,
		CurrentBlockHash:  lastBlockHash,
		PreviousBlockHash: previousBlockHash}
}
mgr.bcInfo.Store(bcInfo)
return mgr
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

   117 blockfile_helper.go
    94 blockfile_rw.go
   381 blockindex.go
   218 block_serialization.go
   101 blocks_itr.go
   209 block_stream.go