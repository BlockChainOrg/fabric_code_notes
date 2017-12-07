# Fabric 1.0源代码笔记 之 Orderer #multichain（多链支持包）

## 1、multichain概述

multichain代码集中在orderer/multichain目录下，目录结构如下：

* manager.go，Manager接口定义及实现。
* chainsupport.go，ChainSupport接口定义及实现。
* systemchain.go，system chain。

## 2、Manager接口定义及实现

### 2.1、Manager接口定义

用于链的创建和访问。

```go
type Manager interface {
	//获取ChainSupport，以及判断链是否存在
	GetChain(chainID string) (ChainSupport, bool)
	//获取系统通道的通道ID
	SystemChannelID() string
	//支持通道创建请求
	NewChannelConfig(envConfigUpdate *cb.Envelope) (configtxapi.Manager, error)
}
//代码在orderer/multichain/manager.go
```

### 2.2、Manager接口实现

Manager接口实现，即multiLedger结构体及方法。

```go
type multiLedger struct {
	chains          map[string]*chainSupport
	consenters      map[string]Consenter
	ledgerFactory   ledger.Factory
	signer          crypto.LocalSigner
	systemChannelID string
	systemChannel   *chainSupport
}

type configResources struct {
	configtxapi.Manager
}

type ledgerResources struct {
	*configResources
	ledger ledger.ReadWriter
}
//代码在orderer/multichain/manager.go
```

涉及方法如下：

```go
func (cr *configResources) SharedConfig() config.Orderer
//获取配置交易Envelope
func getConfigTx(reader ledger.Reader) *cb.Envelope
//构造multiLedger
func NewManagerImpl(ledgerFactory ledger.Factory, consenters map[string]Consenter, signer crypto.LocalSigner) Manager
//获取系统链ID
func (ml *multiLedger) SystemChannelID() string
//按chainID获取ChainSupport
func (ml *multiLedger) GetChain(chainID string) (ChainSupport, bool)
//构造ledgerResources
func (ml *multiLedger) newLedgerResources(configTx *cb.Envelope) *ledgerResources
//创建新链
func (ml *multiLedger) newChain(configtx *cb.Envelope)
//通道或链的个数
func (ml *multiLedger) channelsCount() int
//支持创建新的通道
func (ml *multiLedger) NewChannelConfig(envConfigUpdate *cb.Envelope) (configtxapi.Manager, error)
//代码在orderer/multichain/manager.go
```

func NewManagerImpl(ledgerFactory ledger.Factory, consenters map[string]Consenter, signer crypto.LocalSigner) Manager代码如下：

```go
func NewManagerImpl(ledgerFactory ledger.Factory, consenters map[string]Consenter, signer crypto.LocalSigner) Manager {
	ml := &multiLedger{
		chains:        make(map[string]*chainSupport),
		ledgerFactory: ledgerFactory,
		consenters:    consenters,
		signer:        signer,
	}

	existingChains := ledgerFactory.ChainIDs()
	for _, chainID := range existingChains {
		rl, err := ledgerFactory.GetOrCreate(chainID)
		configTx := getConfigTx(rl)
		ledgerResources := ml.newLedgerResources(configTx)
		chainID := ledgerResources.ChainID()

		if _, ok := ledgerResources.ConsortiumsConfig(); ok { //系统链
			chain := newChainSupport(createSystemChainFilters(ml, ledgerResources), ledgerResources, consenters, signer)
			ml.chains[chainID] = chain
			ml.systemChannelID = chainID
			ml.systemChannel = chain
			defer chain.start()
		} else { //普通链
			chain := newChainSupport(createStandardFilters(ledgerResources), ledgerResources, consenters, signer)
			ml.chains[chainID] = chain
			chain.start()
		}
	}
	return ml
}
//代码在orderer/multichain/manager.go
```

## 3、ChainSupport接口定义及实现

### 3.1、ChainSupport接口定义

```go
type ChainSupport interface {
	PolicyManager() policies.Manager //策略管理
	Reader() ledger.Reader
	Errored() <-chan struct{}
	broadcast.Support
	ConsenterSupport //嵌入ConsenterSupport接口
	Sequence() uint64
	//支持通道更新
	ProposeConfigUpdate(env *cb.Envelope) (*cb.ConfigEnvelope, error)
}

type ConsenterSupport interface {
	crypto.LocalSigner
	BlockCutter() blockcutter.Receiver
	SharedConfig() config.Orderer
	CreateNextBlock(messages []*cb.Envelope) *cb.Block
	WriteBlock(block *cb.Block, committers []filter.Committer, encodedMetadataValue []byte) *cb.Block
	ChainID() string
	Height() uint64
}

type Consenter interface { //定义支持排序机制
	HandleChain(support ConsenterSupport, metadata *cb.Metadata) (Chain, error)
}

type Chain interface {
	//接受消息
	Enqueue(env *cb.Envelope) bool
	Errored() <-chan struct{}
	Start() //开始
	Halt() //挂起
}
//代码在orderer/multichain/chainsupport.go
```

Consenter和Chain更详细内容，参考：[Fabric 1.0源代码笔记 之 consenter（共识插件）](../consenter/README.md)

### 3.2、ChainSupport和ConsenterSupport接口实现

ChainSupport接口实现，即chainSupport结构体及方法。

```go
type chainSupport struct {
	*ledgerResources
	chain         Chain
	cutter        blockcutter.Receiver
	filters       *filter.RuleSet
	signer        crypto.LocalSigner
	lastConfig    uint64
	lastConfigSeq uint64
}
//代码在orderer/multichain/chainsupport.go
```

涉及方法如下：

```go
//构造chainSupport
func newChainSupport(filters *filter.RuleSet,ledgerResources *ledgerResources,consenters map[string]Consenter,signer crypto.LocalSigner,) *chainSupport
func createStandardFilters(ledgerResources *ledgerResources) *filter.RuleSet
func createSystemChainFilters(ml *multiLedger, ledgerResources *ledgerResources) *filter.RuleSet
func (cs *chainSupport) start()
func (cs *chainSupport) NewSignatureHeader() (*cb.SignatureHeader, error)
func (cs *chainSupport) Sign(message []byte) ([]byte, error)
func (cs *chainSupport) Filters() *filter.RuleSet
func (cs *chainSupport) BlockCutter() blockcutter.Receiver
func (cs *chainSupport) Reader() ledger.Reader
func (cs *chainSupport) Enqueue(env *cb.Envelope) bool
func (cs *chainSupport) Errored() <-chan struct{}
//创建块，调取ledger.CreateNextBlock(cs.ledger, messages)
func (cs *chainSupport) CreateNextBlock(messages []*cb.Envelope) *cb.Block
func (cs *chainSupport) addBlockSignature(block *cb.Block)
func (cs *chainSupport) addLastConfigSignature(block *cb.Block)
//写入块
func (cs *chainSupport) WriteBlock(block *cb.Block, committers []filter.Committer, encodedMetadataValue []byte) *cb.Block 
func (cs *chainSupport) Height() uint64
//代码在orderer/multichain/chainsupport.go
```

func (cs *chainSupport) WriteBlock(block *cb.Block, committers []filter.Committer, encodedMetadataValue []byte) *cb.Block 代码如下：

```go
func (cs *chainSupport) WriteBlock(block *cb.Block, committers []filter.Committer, encodedMetadataValue []byte) *cb.Block {
	for _, committer := range committers {
		committer.Commit()
	}
	cs.addBlockSignature(block)
	cs.addLastConfigSignature(block)
	err := cs.ledger.Append(block)//账本追加块
	return block
}
//代码在orderer/multichain/chainsupport.go
```