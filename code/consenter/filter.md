# Fabric 1.0源代码笔记 之 consenter（共识插件） #filter（过滤器）

## 1、filter概述

filter代码分布在orderer/common/filter、orderer/common/configtxfilter、orderer/common/sizefilter、orderer/common/sigfilter、orderer/multichain目录下。

orderer/common/filter/filter.go，Rule接口定义及emptyRejectRule和acceptRule实现，Committer接口定义及noopCommitter实现，RuleSet结构体及方法。
orderer/common/configtxfilter目录，configFilter结构体（实现Rule接口）及configCommitter结构体（实现Committer接口）。
orderer/common/sizefilter目录，maxBytesRule结构体（实现Rule接口）。
orderer/multichain/chainsupport.go，filter工具函数。
orderer/multichain/systemchain.go，systemChainFilter结构体（实现Rule接口）及systemChainCommitter结构体（实现Committer接口）。

## 2、Rule接口定义及实现

### 2.1、Rule接口定义

```go
type Action int
const (
	Accept = iota
	Reject
	Forward
)

type Rule interface { //定义一个过滤器函数, 它接受、拒绝或转发 (到下一条规则) 一个信封
	Apply(message *ab.Envelope) (Action, Committer)
}
//代码在orderer/common/filter/filter.go
```

### 2.2、emptyRejectRule（校验是否为空过滤器）

```go
type emptyRejectRule struct{}
var EmptyRejectRule = Rule(emptyRejectRule{})

func (a emptyRejectRule) Apply(message *ab.Envelope) (Action, Committer) {
	if message.Payload == nil {
		return Reject, nil
	}
	return Forward, nil
}
//代码在orderer/common/filter/filter.go
```

### 2.3、acceptRule（接受过滤器）

```go
type acceptRule struct{}
var AcceptRule = Rule(acceptRule{})

func (a acceptRule) Apply(message *ab.Envelope) (Action, Committer) {
	return Accept, NoopCommitter
}
//代码在orderer/common/filter/filter.go
```

### 2.4、configFilter（配置交易合法性过滤器）

```go
type configFilter struct {
	configManager api.Manager
}

func NewFilter(manager api.Manager) filter.Rule //构造configFilter
//配置交易过滤器
func (cf *configFilter) Apply(message *cb.Envelope) (filter.Action, filter.Committer) {
	msgData, err := utils.UnmarshalPayload(message.Payload) //获取Payload
	chdr, err := utils.UnmarshalChannelHeader(msgData.Header.ChannelHeader) //获取ChannelHeader
	if chdr.Type != int32(cb.HeaderType_CONFIG) { //配置交易
		return filter.Forward, nil
	}
	configEnvelope, err := configtx.UnmarshalConfigEnvelope(msgData.Data) //获取configEnvelope
	err = cf.configManager.Validate(configEnvelope) //校验configEnvelope
	return filter.Accept, &configCommitter{
		manager:        cf.configManager,
		configEnvelope: configEnvelope,
	}
}
//代码在orderer/common/configtxfilter/filter.go
```

### 2.5、sizefilter（交易大小过滤器）

```go
type maxBytesRule struct {
	support Support
}

func MaxBytesRule(support Support) filter.Rule //构造maxBytesRule
func (r *maxBytesRule) Apply(message *cb.Envelope) (filter.Action, filter.Committer) {
	maxBytes := r.support.BatchSize().AbsoluteMaxBytes
	if size := messageByteSize(message); size > maxBytes {
		return filter.Reject, nil
	}
	return filter.Forward, nil
}
//代码在orderer/common/sizefilter/sizefilter.go
```

### 2.6、sigFilter（签名数据校验过滤器）

```go
type sigFilter struct {
	policySource  string
	policyManager policies.Manager
}

func New(policySource string, policyManager policies.Manager) filter.Rule //构造sigFilter
func (sf *sigFilter) Apply(message *cb.Envelope) (filter.Action, filter.Committer) {
	signedData, err := message.AsSignedData() //构造SignedData
	policy, ok := sf.policyManager.GetPolicy(sf.policySource) //获取策略
	err = policy.Evaluate(signedData) //校验策略
	if err == nil {
		return filter.Forward, nil
	}
	return filter.Reject, nil
}
//代码在orderer/common/sigfilter/sigfilter.go
```

### 2.7、systemChainFilter（系统链过滤器）

```go
type systemChainFilter struct {
	cc      chainCreator
	support limitedSupport
}

func newSystemChainFilter(ls limitedSupport, cc chainCreator) filter.Rule //构造systemChainFilter
func (scf *systemChainFilter) Apply(env *cb.Envelope) (filter.Action, filter.Committer) {
	msgData := &cb.Payload{}
	err := proto.Unmarshal(env.Payload, msgData) //获取Payload
	chdr, err := utils.UnmarshalChannelHeader(msgData.Header.ChannelHeader)
	if chdr.Type != int32(cb.HeaderType_ORDERER_TRANSACTION) { //ORDERER_TRANSACTION
		return filter.Forward, nil
	}
	maxChannels := scf.support.SharedConfig().MaxChannelsCount()
	if maxChannels > 0 {
		if uint64(scf.cc.channelsCount()) > maxChannels {
			return filter.Reject, nil
		}
	}

	configTx := &cb.Envelope{}
	err = proto.Unmarshal(msgData.Data, configTx)
	err = scf.authorizeAndInspect(configTx)
	return filter.Accept, &systemChainCommitter{
		filter:   scf,
		configTx: configTx,
	}
}
//代码在orderer/multichain/systemchain.go
```

## 3、Committer接口定义及实现

### 3.1、Committer接口定义

```go
type Committer interface {
	Commit() //提交
	Isolated() bool //判断交易是孤立的块，或与其他交易混合的块
}
//代码在orderer/common/filter/filter.go
```

### 3.2、noopCommitter

```go
type noopCommitter struct{}
var NoopCommitter = Committer(noopCommitter{})

func (nc noopCommitter) Commit()        {}
func (nc noopCommitter) Isolated() bool { return false }
//代码在orderer/common/filter/filter.go
```

### 3.3、configCommitter

```go
type configCommitter struct {
	manager        api.Manager
	configEnvelope *cb.ConfigEnvelope
}

func (cc *configCommitter) Commit() {
	err := cc.manager.Apply(cc.configEnvelope)
}

func (cc *configCommitter) Isolated() bool {
	return true
}
//代码在orderer/common/configtxfilter/filter.go
```

### 3.4、systemChainCommitter

```go
type systemChainCommitter struct {
	filter   *systemChainFilter
	configTx *cb.Envelope
}

func (scc *systemChainCommitter) Isolated() bool {
	return true
}

func (scc *systemChainCommitter) Commit() {
	scc.filter.cc.newChain(scc.configTx)
}
//代码在orderer/multichain/systemchain.go
```

### 4、RuleSet结构体及方法

```go
type RuleSet struct {
	rules []Rule
}

func NewRuleSet(rules []Rule) *RuleSet //构造RuleSet
func (rs *RuleSet) Apply(message *ab.Envelope) (Committer, error) {
	for _, rule := range rs.rules {
		action, committer := rule.Apply(message)
		switch action {
		case Accept: //接受
			return committer, nil
		case Reject: //拒绝
			return nil, fmt.Errorf("Rejected by rule: %T", rule)
		default:
		}
	}
	return nil, fmt.Errorf("No matching filter found")
}
//代码在orderer/common/filter/filter.go
```

### 5、filter工具函数

```go
//为普通 (非系统) 链创建过滤器集
func createStandardFilters(ledgerResources *ledgerResources) *filter.RuleSet {
	return filter.NewRuleSet([]filter.Rule{
		filter.EmptyRejectRule, //EmptyRejectRule
		sizefilter.MaxBytesRule(ledgerResources.SharedConfig()), //sizefilter
		sigfilter.New(policies.ChannelWriters, ledgerResources.PolicyManager()), //sigfilter
		configtxfilter.NewFilter(ledgerResources), //configtxfilter
		filter.AcceptRule, //AcceptRule
	})

}

//为系统链创建过滤器集
func createSystemChainFilters(ml *multiLedger, ledgerResources *ledgerResources) *filter.RuleSet {
	return filter.NewRuleSet([]filter.Rule{
		filter.EmptyRejectRule, //EmptyRejectRule
		sizefilter.MaxBytesRule(ledgerResources.SharedConfig()), //sizefilter
		sigfilter.New(policies.ChannelWriters, ledgerResources.PolicyManager()), //sigfilter
		newSystemChainFilter(ledgerResources, ml),
		configtxfilter.NewFilter(ledgerResources), //configtxfilter
		filter.AcceptRule, //AcceptRule
	})
}
//代码在orderer/multichain/chainsupport.go
```