# Fabric 1.0源代码笔记 之 configtx（配置交易） #ChannelConfig（通道配置）

## 1、ChannelConfig概述

ChannelConfig代码分布在common/config目录下。目录结构如下：

* api.go，核心接口定义，如Org、ApplicationOrg、Channel、Orderer、Application、Consortium、Consortiums、ValueProposer接口定义。
* 
* root.go，Root结构体及方法。
* channel.go，ChannelGroup结构体及方法。
* orderer.go，OrdererGroup结构体及方法。
* application.go，ApplicationGroup结构体及方法。

## 2、核心接口定义

```go
type Org interface { //组织接口
	Name() string //组织名称
	MSPID() string //组织MSPID
}

type ApplicationOrg interface { //应用组织接口
	Org //嵌入Org
	AnchorPeers() []*pb.AnchorPeer //锚节点
}

type Channel interface { //通道配置接口
	HashingAlgorithm() func(input []byte) []byte //哈希算法
	BlockDataHashingStructureWidth() uint32 //指定计算 BlockDataHash 时使用的 Merkle 树的宽度
	OrdererAddresses() []string //Orderer地址
}

type Application interface { //应用配置接口
	Organizations() map[string]ApplicationOrg //应用组织map
}

type Consortiums interface { //联盟配置map接口
	Consortiums() map[string]Consortium //Consortium map
}

type Consortium interface { //联盟配置接口
	ChannelCreationPolicy() *cb.Policy //通道创建策略
}

type Orderer interface { //Orderer配置接口
	ConsensusType() string //共识类型
	BatchSize() *ab.BatchSize //块中的最大消息数
	BatchTimeout() time.Duration //创建批处理之前等待的时间量
	MaxChannelsCount() uint64 //最大通道数
	KafkaBrokers() []string //Kafka地址
	Organizations() map[string]Org //Orderer组织
}

type ValueProposer interface {
	BeginValueProposals(tx interface{}, groups []string) (ValueDeserializer, []ValueProposer, error) //配置Proposal前
	RollbackProposals(tx interface{}) //回滚配置Proposal
	PreCommit(tx interface{}) error //提交前
	CommitProposals(tx interface{}) //提交
}
//代码在common/config/api.go
```

## 2、Root结构体及方法

```go
type Root struct {
	channel          *ChannelGroup
	mspConfigHandler *msp.MSPConfigHandler
}

func NewRoot(mspConfigHandler *msp.MSPConfigHandler) *Root //构造Root
//启动新的配置Proposal，r.mspConfigHandler.BeginConfig(tx)
func (r *Root) BeginValueProposals(tx interface{}, groups []string) (ValueDeserializer, []ValueProposer, error) 
//回滚配置Proposal，r.mspConfigHandler.RollbackProposals(tx)
func (r *Root) RollbackProposals(tx interface{})
//提交前校验配置，r.mspConfigHandler.PreCommit(tx)
func (r *Root) PreCommit(tx interface{}) error
//提交配置Proposal，r.mspConfigHandler.CommitProposals(tx)
func (r *Root) CommitProposals(tx interface{})
//获取r.channel
func (r *Root) Channel() *ChannelGroup
//获取r.channel.OrdererConfig()
func (r *Root) Orderer() *OrdererGroup
//获取r.channel.ApplicationConfig()
func (r *Root) Application() *ApplicationGroup
//获取r.channel.ConsortiumsConfig()
func (r *Root) Consortiums() *ConsortiumsGroup {
//代码在common/config/root.go
```

## 3、ChannelGroup结构体及方法

### 3.1、ChannelGroup结构体及方法

```go
type ChannelGroup struct {
	*ChannelConfig //嵌入ChannelConfig
	*Proposer //嵌入Proposer
	mspConfigHandler *msp.MSPConfigHandler
}

type ChannelConfig struct {
	*standardValues
	protos *ChannelProtos
	hashingAlgorithm func(input []byte) []byte
	appConfig         *ApplicationGroup
	ordererConfig     *OrdererGroup
	consortiumsConfig *ConsortiumsGroup
}

type ChannelProtos struct {
	HashingAlgorithm          *cb.HashingAlgorithm
	BlockDataHashingStructure *cb.BlockDataHashingStructure
	OrdererAddresses          *cb.OrdererAddresses
	Consortium                *cb.Consortium
}

构造ChannelGroup，以及构造ChannelConfig和Proposer
func NewChannelGroup(mspConfigHandler *msp.MSPConfigHandler) *ChannelGroup
func (cg *ChannelGroup) Allocate() Values //构造channelConfigSetter
//获取cg.ChannelConfig.ordererConfig
func (cg *ChannelGroup) OrdererConfig() *OrdererGroup
//获取cg.ChannelConfig.appConfig
func (cg *ChannelGroup) ApplicationConfig() *ApplicationGroup
//获取cg.ChannelConfig.consortiumsConfig
func (cg *ChannelGroup) ConsortiumsConfig() *ConsortiumsGroup
func (cg *ChannelGroup) NewGroup(group string) (ValueProposer, error)

//构造ChannelConfig，NewStandardValues(cc.protos)
func NewChannelConfig() *ChannelConfig
//获取cc.hashingAlgorithm
func (cc *ChannelConfig) HashingAlgorithm() func(input []byte) []byte
//获取cc.protos.BlockDataHashingStructure.Width
func (cc *ChannelConfig) BlockDataHashingStructureWidth() uint32
//获取cc.protos.OrdererAddresses.Addresses
func (cc *ChannelConfig) OrdererAddresses() []string
//获取cc.protos.Consortium.Name
func (cc *ChannelConfig) ConsortiumName() string
func (cc *ChannelConfig) Validate(tx interface{}, groups map[string]ValueProposer) error
func (cc *ChannelConfig) validateHashingAlgorithm() error
func (cc *ChannelConfig) validateBlockDataHashingStructure() error
func (cc *ChannelConfig) validateOrdererAddresses() error
//代码在common/config/channel.go
```

补充cb.HashingAlgorithm、cb.BlockDataHashingStructure、cb.OrdererAddresses、cb.Consortium定义：

```go
//哈希算法
type HashingAlgorithm struct {
	Name string
}

//块数据哈希结构
type BlockDataHashingStructure struct {
	Width uint32 //指定计算 BlockDataHash 时使用的 Merkle 树的宽度
}

//Orderer地址
type OrdererAddresses struct {
	Addresses []string
}

type Consortium struct {
	Name string
}
//代码在protos/common/configuration.pb.go
```

### 3.2、Proposer结构体及方法

```go
type Proposer struct {
	vh          Handler
	pending     map[interface{}]*config
	current     *config
	pendingLock sync.RWMutex
}
func NewProposer(vh Handler) *Proposer
func (p *Proposer) BeginValueProposals(tx interface{}, groups []string) (ValueDeserializer, []ValueProposer, error)
func (p *Proposer) PreCommit(tx interface{}) error
func (p *Proposer) RollbackProposals(tx interface{})
func (p *Proposer) CommitProposals(tx interface{})
//代码在common/config/proposer.go
```

### 3.3、OrdererGroup结构体及方法

```go
type OrdererGroup struct {
	*Proposer
	*OrdererConfig
	mspConfig *msp.MSPConfigHandler
}

type OrdererConfig struct {
	*standardValues
	protos       *OrdererProtos
	ordererGroup *OrdererGroup
	orgs         map[string]Org
	batchTimeout time.Duration
}

type OrdererProtos struct {
	ConsensusType       *ab.ConsensusType
	BatchSize           *ab.BatchSize
	BatchTimeout        *ab.BatchTimeout
	KafkaBrokers        *ab.KafkaBrokers
	ChannelRestrictions *ab.ChannelRestrictions
}

//构造OrdererGroup，以及Proposer
func NewOrdererGroup(mspConfig *msp.MSPConfigHandler) *OrdererGroup
func (og *OrdererGroup) NewGroup(name string) (ValueProposer, error)
func (og *OrdererGroup) Allocate() Values

//构造OrdererConfig
func NewOrdererConfig(og *OrdererGroup) *OrdererConfig
//oc.ordererGroup.OrdererConfig = oc
func (oc *OrdererConfig) Commit()
//获取oc.protos.ConsensusType.Type
func (oc *OrdererConfig) ConsensusType() string
//获取oc.protos.BatchSize
func (oc *OrdererConfig) BatchSize() *ab.BatchSize
//获取oc.batchTimeout
func (oc *OrdererConfig) BatchTimeout() time.Duration
//获取oc.protos.KafkaBrokers.Brokers
func (oc *OrdererConfig) KafkaBrokers() []string
//获取oc.protos.ChannelRestrictions.MaxCount
func (oc *OrdererConfig) MaxChannelsCount() uint64
//获取oc.orgs
func (oc *OrdererConfig) Organizations() map[string]Org
func (oc *OrdererConfig) Validate(tx interface{}, groups map[string]ValueProposer) error
func (oc *OrdererConfig) validateConsensusType() error
func (oc *OrdererConfig) validateBatchSize() error
func (oc *OrdererConfig) validateBatchTimeout() error
func (oc *OrdererConfig) validateKafkaBrokers() error
func brokerEntrySeemsValid(broker string) bool
//代码在common/config/orderer.go
```

### 3.4、ApplicationGroup结构体及方法

```go
//代码在common/config/application.go
```