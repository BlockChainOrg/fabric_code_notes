# Fabric 1.0源代码笔记 之 blockfile（区块文件存储）

## 1、blockfile概述

blockfile，即Fabric区块链区块文件存储，默认目录/var/hyperledger/production/ledgersData/chains，含index和chains两个子目录。
其中index为索引目录，采用leveldb实现。而chains为各ledger的区块链文件，子目录以ledgerid为名，使用文件系统实现。
区块文件以blockfile_为前缀，最大大小默认64M。

blockfile，相关代码集中在common/ledger/blkstorage/fsblkstorage目录，目录结构如下：

* blockfile_mgr.go，blockfileMgr和checkpointInfo结构体及方法。
* block_stream.go，blockfileStream、blockStream、blockPlacementInfo结构体及方法。
* blockfile_rw.go，blockfileWriter和blockfileReader结构体及方法（blockfileReader未使用）。
* blockindex.go，index接口定义，index接口实现即blockIndex结构体及方法定义，以及blockIdxInfo、locPointer、fileLocPointer结构体及方法。
* blockfile_helper.go，定义了4个工具函数，constructCheckpointInfoFromBlockFiles、retrieveLastFileSuffix、isBlockFileName、getFileInfoOrPanic。
作用分别为：扫描最新的blockfile并重新构造检查点信息、获取最新的文件后缀、根据文件前缀判断是否为区块文件、获取文件状态信息。
* block_serialization.go，block序列化相关工具函数。
* blocks_itr.go，blocksItr结构体及方法。

## 2、Block结构体定、以及Block序列化

### 2.1、Block相关结构体

Block结构体：

```go
type Block struct {
	Header   *BlockHeader //BlockHeader
	Data     *BlockData //BlockData
	Metadata *BlockMetadata
}

func (m *Block) GetHeader() *BlockHeader //获取BlockHeader，即m.Header
func (m *Block) GetData() *BlockData //获取BlockData，即m.Data
func (m *Block) GetMetadata() *BlockMetadata //m.Metadata
//代码在protos/common/common.pb.go
```

BlockHeader结构体：

```go
type BlockHeader struct {
	Number       uint64 //区块编号
	PreviousHash []byte //前一个区块哈希
	DataHash     []byte //当前区块哈希
}

func (m *BlockHeader) GetNumber() uint64 //获取区块编号，即m.Number
func (m *BlockHeader) GetPreviousHash() []byte //获取前一个区块哈希，即m.PreviousHash
func (m *BlockHeader) GetDataHash() []byte //获取当前区块哈希，即m.DataHash
//代码在protos/common/common.pb.go
```

BlockData结构体：

```go
type BlockData struct {
	Data [][]byte //Data，存储交易信息
}

func (m *BlockData) GetData() [][]byte //获取Data，即m.Data
//代码在protos/common/common.pb.go
```

BlockMetadata结构体：

```go
type BlockMetadata struct {
	Metadata [][]byte //K/V均为[]byte格式
}

func (m *BlockMetadata) GetMetadata() [][]byte //m.Metadata
//代码在protos/common/common.pb.go
```

补充BlockMetadataIndex：

```go
type BlockMetadataIndex int32

const (
	BlockMetadataIndex_SIGNATURES          BlockMetadataIndex = 0
	BlockMetadataIndex_LAST_CONFIG         BlockMetadataIndex = 1
	BlockMetadataIndex_TRANSACTIONS_FILTER BlockMetadataIndex = 2
	BlockMetadataIndex_ORDERER             BlockMetadataIndex = 3
)
```

### 2.2、Block序列化

serializedBlockInfo结构体定义及工具函数：

```go
type serializedBlockInfo struct {
	blockHeader *common.BlockHeader //BlockHeader
	txOffsets   []*txindexInfo //交易索引信息
	metadata    *common.BlockMetadata
}

type txindexInfo struct {
	txID string //交易ID
	loc  *locPointer //文件指针
}

//序列化区块，返回序列化后字节，以及serializedBlockInfo（含BlockHeader和交易索引信息）
func serializeBlock(block *common.Block) ([]byte, *serializedBlockInfo, error)
//反序列化区块，构建Block结构体
func deserializeBlock(serializedBlockBytes []byte) (*common.Block, error)
//反序列化区块，并构造serializedBlockInfo
func extractSerializedBlockInfo(serializedBlockBytes []byte) (*serializedBlockInfo, error)
//序列化中添加BlockHeader，即Number、DataHash和PreviousHash
func addHeaderBytes(blockHeader *common.BlockHeader, buf *proto.Buffer) error
//序列化中添加BlockData，并从BlockData中解析txid，返回交易索引信息数组
func addDataBytes(blockData *common.BlockData, buf *proto.Buffer) ([]*txindexInfo, error)
//序列化中添加Metadata
func addMetadataBytes(blockMetadata *common.BlockMetadata, buf *proto.Buffer) error
//反序列化出BlockHeader
func extractHeader(buf *ledgerutil.Buffer) (*common.BlockHeader, error)
//反序列化出BlockData，并返回交易索引信息数组
func extractData(buf *ledgerutil.Buffer) (*common.BlockData, []*txindexInfo, error)
//反序列化出Metadata
func extractMetadata(buf *ledgerutil.Buffer) (*common.BlockMetadata, error)
//从BlockData中解析出交易ID
func extractTxID(txEnvelopBytes []byte) (string, error)
//代码在common/ledger/blkstorage/fsblkstorage/block_serialization.go
```

## 3、checkpointInfo结构体定义及方法

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

## 4、blockfileStream相关结构体及方法

### 4.1、blockfileStream

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
func (s *blockfileStream) nextBlockBytes() ([]byte, error) //下一个块，调取s.nextBlockBytesAndPlacementInfo()
//下一个块和位置信息
func (s *blockfileStream) nextBlockBytesAndPlacementInfo() ([]byte, *blockPlacementInfo, error) 
func (s *blockfileStream) close() error //关闭blockfileStream
//代码在common/ledger/blkstorage/fsblkstorage/block_stream.go
```

func (s *blockfileStream) nextBlockBytesAndPlacementInfo() ([]byte, *blockPlacementInfo, error) 代码如下：

```go
var lenBytes []byte
var err error
var fileInfo os.FileInfo
moreContentAvailable := true

fileInfo, err = s.file.Stat() //获取文件状态
remainingBytes := fileInfo.Size() - s.currentOffset //文件读取剩余字节
peekBytes := 8
if remainingBytes < int64(peekBytes) { //剩余字节小于8，按实际剩余字节，否则按8
	peekBytes = int(remainingBytes)
	moreContentAvailable = false
}
//存储形式：前n位存储block长度length，之后length位为实际block
lenBytes, err = s.reader.Peek(peekBytes) //Peek 返回缓存的一个切片，该切片引用缓存中前 peekBytes 个字节的数据
length, n := proto.DecodeVarint(lenBytes) //从切片中读取 varint 编码的整数，它返回整数和被消耗的字节数。
	err = s.reader.Discard(n) //丢弃存储block长度length的前n位
	blockBytes := make([]byte, length)
	_, err = io.ReadAtLeast(s.reader, blockBytes, int(length))
	blockPlacementInfo := &blockPlacementInfo{
		fileNum:          s.fileNum,
		blockStartOffset: s.currentOffset,
		blockBytesOffset: s.currentOffset + int64(n)}
	s.currentOffset += int64(n) + int64(length)
	return blockBytes, blockPlacementInfo, nil
//代码在common/ledger/blkstorage/fsblkstorage/block_stream.go
```

补充blockPlacementInfo：块位置信息

```go
type blockPlacementInfo struct {
	fileNum          int //块文件后缀
	blockStartOffset int64 //n+length，n之前
	blockBytesOffset int64 //n+length，length之前
}
//代码在common/ledger/blkstorage/fsblkstorage/block_stream.go
```

## 5、blockfileWriter结构体定义及方法

```go
type blockfileWriter struct {
	filePath string //路径
	file     *os.File //os.File
}

func newBlockfileWriter(filePath string) (*blockfileWriter, error) //构造blockfileWriter，并调用writer.open()
func (w *blockfileWriter) truncateFile(targetSize int) error //截取文件
func (w *blockfileWriter) append(b []byte, sync bool) error //追加文件
func (w *blockfileWriter) open() error //打开文件
func (w *blockfileWriter) close() error //关闭文件
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_rw.go
```

## 6、blockIndex相关结构体及方法

### 6.1、index接口定义

```go
type index interface {
	getLastBlockIndexed() (uint64, error) //获取最后一个块索引（或编号）
	indexBlock(blockIdxInfo *blockIdxInfo) error //索引区块
	getBlockLocByHash(blockHash []byte) (*fileLocPointer, error) //根据区块哈希，获取文件区块指针
	getBlockLocByBlockNum(blockNum uint64) (*fileLocPointer, error) //根据区块编号，获取文件区块指针
	getTxLoc(txID string) (*fileLocPointer, error) //根据交易ID，获取文件交易指针
	getTXLocByBlockNumTranNum(blockNum uint64, tranNum uint64) (*fileLocPointer, error) //根据区块编号和交易编号，获取文件交易指针
	getBlockLocByTxID(txID string) (*fileLocPointer, error)//根据交易ID，获取文件区块指针
	getTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error)//根据交易ID，获取交易验证代码
}
//代码在common/ledger/blkstorage/fsblkstorage/blockindex.go
```

### 6.2、blockIndex结构体

blockIndex结构体定义如下：

```go
type blockIndex struct {
	indexItemsMap map[blkstorage.IndexableAttr]bool //index属性映射
	db            *leveldbhelper.DBHandle //index leveldb操作
}
//代码在common/ledger/blkstorage/fsblkstorage/blockindex.go
```

补充IndexableAttr：

```go
const (
	IndexableAttrBlockNum         = IndexableAttr("BlockNum")
	IndexableAttrBlockHash        = IndexableAttr("BlockHash")
	IndexableAttrTxID             = IndexableAttr("TxID")
	IndexableAttrBlockNumTranNum  = IndexableAttr("BlockNumTranNum")
	IndexableAttrBlockTxID        = IndexableAttr("BlockTxID")
	IndexableAttrTxValidationCode = IndexableAttr("TxValidationCode")
)
//代码在common/ledger/blkstorage/blockstorage.go
```

涉及方法如下：

```go
//构造blockIndex
func newBlockIndex(indexConfig *blkstorage.IndexConfig, db *leveldbhelper.DBHandle) *blockIndex 
//获取最后一个块索引（或编号），取key为"indexCheckpointKey"的值，即为最新的区块编号
func (index *blockIndex) getLastBlockIndexed() (uint64, error) 
func (index *blockIndex) indexBlock(blockIdxInfo *blockIdxInfo) error //索引区块
//根据区块哈希，获取文件区块指针
func (index *blockIndex) getBlockLocByHash(blockHash []byte) (*fileLocPointer, error) 
//根据区块编号，获取文件区块指针
func (index *blockIndex) getBlockLocByBlockNum(blockNum uint64) (*fileLocPointer, error) 
//根据交易ID，获取文件交易指针
func (index *blockIndex) getTxLoc(txID string) (*fileLocPointer, error) 
//根据交易ID，获取文件区块指针
func (index *blockIndex) getBlockLocByTxID(txID string) (*fileLocPointer, error) 
//根据区块编号和交易编号，获取文件交易指针
func (index *blockIndex) getTXLocByBlockNumTranNum(blockNum uint64, tranNum uint64) (*fileLocPointer, error) 
//根据交易ID，获取交易验证代码
func (index *blockIndex) getTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error) 
//代码在common/ledger/blkstorage/fsblkstorage/blockindex.go
```

补充blockIdxInfo结构体定义：块索引信息。

```
type blockIdxInfo struct {
	blockNum  uint64 //区块编号
	blockHash []byte //区块哈希
	flp       *fileLocPointer //文件指针
	txOffsets []*txindexInfo //交易索引信息
	metadata  *common.BlockMetadata 
}
//代码在common/ledger/blkstorage/fsblkstorage/blockindex.go
```

补充fileLocPointer、txindexInfo和common.BlockMetadata：

```go
type locPointer struct { //定义指针
	offset      int //偏移位置
	bytesLength int //字节长度
}
type fileLocPointer struct { //定义文件指针
	fileSuffixNum int //文件后缀
	locPointer //嵌入locPointer
}
//代码在common/ledger/blkstorage/fsblkstorage/blockindex.go

type txindexInfo struct { //交易索引信息
	txID string //交易ID
	loc  *locPointer //文件指针
}
//代码在common/ledger/blkstorage/fsblkstorage/block_serialization.go

type BlockMetadata struct {
	Metadata [][]byte `protobuf:"bytes,1,rep,name=metadata,proto3" json:"metadata,omitempty"`
}
//代码在protos/common/common.pb.go
```

func (index *blockIndex) indexBlock(blockIdxInfo *blockIdxInfo) error代码如下：

```go
flp := blockIdxInfo.flp //文件指针
txOffsets := blockIdxInfo.txOffsets //交易索引信息
txsfltr := ledgerUtil.TxValidationFlags(blockIdxInfo.metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER]) //type TxValidationFlags []uint8
batch := leveldbhelper.NewUpdateBatch() //leveldb批量更新
flpBytes, err := flp.marshal() //文件指针序列化，含文件后缀、偏移位置、字节长度

if _, ok := index.indexItemsMap[blkstorage.IndexableAttrBlockHash]; ok { //使用区块哈希索引文件区块指针
	batch.Put(constructBlockHashKey(blockIdxInfo.blockHash), flpBytes) //区块哈希，blockHash：flpBytes存入leveldb
}

if _, ok := index.indexItemsMap[blkstorage.IndexableAttrBlockNum]; ok { //使用区块编号索引文件区块指针
	batch.Put(constructBlockNumKey(blockIdxInfo.blockNum), flpBytes) //区块编号，blockNum：flpBytes存入leveldb
}

if _, ok := index.indexItemsMap[blkstorage.IndexableAttrTxID]; ok { //使用交易ID索引文件交易指针
	for _, txoffset := range txOffsets {
		txFlp := newFileLocationPointer(flp.fileSuffixNum, flp.offset, txoffset.loc)
		txFlpBytes, marshalErr := txFlp.marshal()
		batch.Put(constructTxIDKey(txoffset.txID), txFlpBytes) //交易ID，txID：txFlpBytes存入leveldb
	}
}

if _, ok := index.indexItemsMap[blkstorage.IndexableAttrBlockNumTranNum]; ok { //使用区块编号和交易编号索引文件交易指针
	for txIterator, txoffset := range txOffsets {
		txFlp := newFileLocationPointer(flp.fileSuffixNum, flp.offset, txoffset.loc)
		txFlpBytes, marshalErr := txFlp.marshal()
		batch.Put(constructBlockNumTranNumKey(blockIdxInfo.blockNum, uint64(txIterator)), txFlpBytes) //区块编号和交易编号，blockNum+txIterator：txFlpBytes
	}
}

if _, ok := index.indexItemsMap[blkstorage.IndexableAttrBlockTxID]; ok { //使用交易ID索引文件区块指针
	for _, txoffset := range txOffsets {
		batch.Put(constructBlockTxIDKey(txoffset.txID), flpBytes) //交易ID，txID：flpBytes
	}
}

if _, ok := index.indexItemsMap[blkstorage.IndexableAttrTxValidationCode]; ok { //使用交易ID索引交易验证代码
	for idx, txoffset := range txOffsets {
		batch.Put(constructTxValidationCodeIDKey(txoffset.txID), []byte{byte(txsfltr.Flag(idx))})
	}
}

batch.Put(indexCheckpointKey, encodeBlockNum(blockIdxInfo.blockNum)) //key为"indexCheckpointKey"的值，即为最新的区块编号
err := index.db.WriteBatch(batch, true) //批量更新
}
//代码在common/ledger/blkstorage/fsblkstorage/blockindex.go
```

## 7、blocksItr结构体及方法

```go
type blocksItr struct {
	mgr                  *blockfileMgr //blockfileMgr
	maxBlockNumAvailable uint64 //最大的区块编号
	blockNumToRetrieve   uint64 //起始区块编号
	stream               *blockStream //blockStream
	closeMarker          bool
	closeMarkerLock      *sync.Mutex
}

func newBlockItr(mgr *blockfileMgr, startBlockNum uint64) *blocksItr //构造blocksItr
func (itr *blocksItr) waitForBlock(blockNum uint64) uint64 
func (itr *blocksItr) initStream() error 
func (itr *blocksItr) shouldClose() bool 
func (itr *blocksItr) Next() (ledger.QueryResult, error) 
func (itr *blocksItr) Close() 
//代码在common/ledger/blkstorage/fsblkstorage/blocks_itr.go
```

## 8、blockfileMgr结构体定义及方法

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
//构建blockfileMgr
func newBlockfileMgr(id string, conf *Conf, indexConfig *blkstorage.IndexConfig, indexStore *leveldbhelper.DBHandle) *blockfileMgr
func syncCPInfoFromFS(rootDir string, cpInfo *checkpointInfo) //从文件系统中更新检查点信息
func deriveBlockfilePath(rootDir string, suffixNum int) string //构造Blockfile路径
func (mgr *blockfileMgr) close() //关闭blockfileWriter
func (mgr *blockfileMgr) moveToNextFile() //转至下个新区块文件，打开新文件、并更新检查点信息
func (mgr *blockfileMgr) addBlock(block *common.Block) error //添加区块，区块写入文件，索引区块
func (mgr *blockfileMgr) syncIndex() error //同步区块索引
func (mgr *blockfileMgr) getBlockchainInfo() *common.BlockchainInfo //获取 BlockchainInfo
func (mgr *blockfileMgr) updateCheckpoint(cpInfo *checkpointInfo) //更新检查点信息
func (mgr *blockfileMgr) updateBlockchainInfo(latestBlockHash []byte, latestBlock *common.Block) //更新BlockchainInfo
//按区块哈希获取块，mgr.index.getBlockLocByHash(blockHash)，mgr.fetchBlock(loc)
func (mgr *blockfileMgr) retrieveBlockByHash(blockHash []byte) (*common.Block, error) 
//按区块编号获取块，mgr.index.getBlockLocByBlockNum(blockNum)，mgr.fetchBlock(loc)
func (mgr *blockfileMgr) retrieveBlockByNumber(blockNum uint64) (*common.Block, error) 
//按交易ID获取块，mgr.index.getBlockLocByTxID(txID)，mgr.fetchBlock(loc)
func (mgr *blockfileMgr) retrieveBlockByTxID(txID string) (*common.Block, error) 
//按交易ID获取交易验证代码，mgr.index.getTxValidationCodeByTxID(txID)
func (mgr *blockfileMgr) retrieveTxValidationCodeByTxID(txID string) (peer.TxValidationCode, error) 
//按区块编号获取BlockHeader：按区块编号从索引中取文件区块指针，按文件指针取区块Bytes，按区块Bytes构建serializedBlockInfo，取serializedBlockInfo.blockHeader
func (mgr *blockfileMgr) retrieveBlockHeaderByNumber(blockNum uint64) (*common.BlockHeader, error)
//构造blocksItr
func (mgr *blockfileMgr) retrieveBlocks(startNum uint64) (*blocksItr, error)
//按交易ID获取交易，mgr.index.getTxLoc(txID)，mgr.fetchTransactionEnvelope(loc)
func (mgr *blockfileMgr) retrieveTransactionByID(txID string) (*common.Envelope, error)
//按区块编号和交易编号获取交易，mgr.index.getTXLocByBlockNumTranNum(blockNum, tranNum)，mgr.fetchTransactionEnvelope(loc)
func (mgr *blockfileMgr) retrieveTransactionByBlockNumTranNum(blockNum uint64, tranNum uint64) (*common.Envelope, error) 
func (mgr *blockfileMgr) fetchBlock(lp *fileLocPointer) (*common.Block, error) //获取下一个块
//获取交易
//type Envelope struct {
//	Payload []byte
//	Signature []byte
//}
func (mgr *blockfileMgr) fetchTransactionEnvelope(lp *fileLocPointer) (*common.Envelope, error) 
//按文件指针获取区块Bytes
func (mgr *blockfileMgr) fetchBlockBytes(lp *fileLocPointer) ([]byte, error)
func (mgr *blockfileMgr) fetchRawBytes(lp *fileLocPointer) ([]byte, error) //按文件指针获取原始字节
func (mgr *blockfileMgr) loadCurrentInfo() (*checkpointInfo, error) //获取存储在index库中最新检查点信息，key为"blkMgrInfo"
func (mgr *blockfileMgr) saveCurrentInfo(i *checkpointInfo, sync bool) error //将最新检查点信息，序列化后存入index库
//扫描给定的块文件并检测文件中的最后完整块，返回最后一个块字节、文件最新偏移量、块数
func scanForLastCompleteBlock(rootDir string, fileNum int, startingOffset int64) ([]byte, int64, int, error) {
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

func newBlockfileMgr(id string, conf *Conf, indexConfig *blkstorage.IndexConfig, indexStore *leveldbhelper.DBHandle) *blockfileMgr实现如下：构建blockfileMgr。

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
err = currentFileWriter.truncateFile(cpInfo.latestFileChunksize) //按最新的区块文件大小截取文件
mgr.index = newBlockIndex(indexConfig, indexStore) 构造blockIndex
mgr.cpInfo = cpInfo
mgr.cpInfoCond = sync.NewCond(&sync.Mutex{})
mgr.syncIndex()

bcInfo := &common.BlockchainInfo{
	Height:            0,
	CurrentBlockHash:  nil,
	PreviousBlockHash: nil}
if !cpInfo.isChainEmpty { //如果不是空链
	lastBlockHeader, err := mgr.retrieveBlockHeaderByNumber(cpInfo.lastBlockNumber) //获取最后一个块的Header
	lastBlockHash := lastBlockHeader.Hash() //最后一个块的哈希
	previousBlockHash := lastBlockHeader.PreviousHash //前一个块的哈希
	bcInfo = &common.BlockchainInfo{ //构造区块链信息
		Height:            cpInfo.lastBlockNumber + 1,
		CurrentBlockHash:  lastBlockHash,
		PreviousBlockHash: previousBlockHash}
}
mgr.bcInfo.Store(bcInfo) //bcInfo赋值给mgr.bcInfo
return mgr
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

func syncCPInfoFromFS(rootDir string, cpInfo *checkpointInfo)代码如下：//从文件系统中更新检查点信息。

```go
filePath := deriveBlockfilePath(rootDir, cpInfo.latestFileChunkSuffixNum) //最新区块文件路径
exists, size, err := util.FileExists(filePath)
_, endOffsetLastBlock, numBlocks, err := scanForLastCompleteBlock( //扫描最后一个完整块
	rootDir, cpInfo.latestFileChunkSuffixNum, int64(cpInfo.latestFileChunksize))
cpInfo.latestFileChunksize = int(endOffsetLastBlock) //最新的区块文件大小
if cpInfo.isChainEmpty { //空链
	cpInfo.lastBlockNumber = uint64(numBlocks - 1) //最新的区块编号
} else {
	cpInfo.lastBlockNumber += uint64(numBlocks)
}
cpInfo.isChainEmpty = false //不再是空链
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```

func (mgr *blockfileMgr) addBlock(block *common.Block) error代码如下：添加区块，区块写入文件，索引区块。

```go
//序列化区块，返回序列化后字节，以及serializedBlockInfo（含BlockHeader和交易索引信息）
blockBytes, info, err := serializeBlock(block)
blockHash := block.Header.Hash() //blockHash
txOffsets := info.txOffsets //交易索引信息
currentOffset := mgr.cpInfo.latestFileChunksize //最新的区块文件大小
blockBytesLen := len(blockBytes)
blockBytesEncodedLen := proto.EncodeVarint(uint64(blockBytesLen)) //blockBytesLen
totalBytesToAppend := blockBytesLen + len(blockBytesEncodedLen) //blockBytesLen + blockBytesEncodedLen

if currentOffset+totalBytesToAppend > mgr.conf.maxBlockfileSize { //超出文件大小限定，创建新文件
	mgr.moveToNextFile()
	currentOffset = 0
}
err = mgr.currentFileWriter.append(blockBytesEncodedLen, false) //追加写入blockBytesLen
if err == nil {
	err = mgr.currentFileWriter.append(blockBytes, true) //追加写入blockBytes
}
if err != nil { //追加写入失败，回滚按原大小截取文件
		truncateErr := mgr.currentFileWriter.truncateFile(mgr.cpInfo.latestFileChunksize)
		return fmt.Errorf("Error while appending block to file: %s", err)
}

currentCPInfo := mgr.cpInfo
newCPInfo := &checkpointInfo{
	latestFileChunkSuffixNum: currentCPInfo.latestFileChunkSuffixNum,
	latestFileChunksize:      currentCPInfo.latestFileChunksize + totalBytesToAppend,
	isChainEmpty:             false,
	lastBlockNumber:          block.Header.Number}
if err = mgr.saveCurrentInfo(newCPInfo, false); err != nil { //更新检查点信息
	truncateErr := mgr.currentFileWriter.truncateFile(currentCPInfo.latestFileChunksize) //更新失败，回滚按原大小截取文件
	return fmt.Errorf("Error while saving current file info to db: %s", err)
}

blockFLP := &fileLocPointer{fileSuffixNum: newCPInfo.latestFileChunkSuffixNum}
blockFLP.offset = currentOffset
for _, txOffset := range txOffsets {
	txOffset.loc.offset += len(blockBytesEncodedLen) //更新文件交易指针
}
mgr.index.indexBlock(&blockIdxInfo{ //索引区块
	blockNum: block.Header.Number, blockHash: blockHash,
	flp: blockFLP, txOffsets: txOffsets, metadata: block.Metadata})

mgr.updateCheckpoint(newCPInfo) //更新检查点信息
mgr.updateBlockchainInfo(blockHash, block) //更新BlockchainInfo
return nil
//代码在common/ledger/blkstorage/fsblkstorage/blockfile_mgr.go
```