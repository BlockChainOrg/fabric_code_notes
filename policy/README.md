# Fabric 1.0源代码笔记 之 policy（背书策略）

## 1、policy概述

policy代码分布在core/policy、core/policyprovider、common/policies目录下。目录结构如下：

* core/policy/policy.go，PolicyChecker接口定义及实现、PolicyCheckerFactory接口定义。
* core/policyprovider/provider.go，PolicyChecker工厂默认实现。
* common/policies目录，ChannelPolicyManagerGetter接口及实现。

## 2、PolicyChecker工厂

### 2.1、PolicyCheckerFactory接口定义

```go
type PolicyCheckerFactory interface {
	NewPolicyChecker() PolicyChecker //构造PolicyChecker实例
}

var pcFactory PolicyCheckerFactory //全局变量定义及赋值函数
func RegisterPolicyCheckerFactory(f PolicyCheckerFactory) {
	pcFactory = f
}
//代码在core/policy/policy.go
```

### 2.2、PolicyCheckerFactory接口默认实现

```go
type defaultFactory struct{}

//构造policy.PolicyChecker
func (f *defaultFactory) NewPolicyChecker() policy.PolicyChecker {
	return policy.NewPolicyChecker(
		peer.NewChannelPolicyManagerGetter(), //&channelPolicyManagerGetter{}
		mgmt.GetLocalMSP(),
		mgmt.NewLocalMSPPrincipalGetter(),
	)
}

//获取policy.PolicyChecker，即调用policy.GetPolicyChecker()
func GetPolicyChecker() policy.PolicyChecker

func init() { //初始化全局变量pcFactory
	policy.RegisterPolicyCheckerFactory(&defaultFactory{})
}
```

## 3、PolicyChecker接口定义及实现

### 3.1、PolicyChecker接口定义

```go
type PolicyChecker interface {
	CheckPolicy(channelID, policyName string, signedProp *pb.SignedProposal) error
	CheckPolicyBySignedData(channelID, policyName string, sd []*common.SignedData) error
	CheckPolicyNoChannel(policyName string, signedProp *pb.SignedProposal) error
}
//代码在core/policy/policy.go
```

### 3.2、PolicyChecker接口实现

PolicyChecker接口实现，即policyChecker结构体及方法。

```go
type policyChecker struct {
	channelPolicyManagerGetter policies.ChannelPolicyManagerGetter //通道策略管理器
	localMSP                   msp.IdentityDeserializer //身份
	principalGetter            mgmt.MSPPrincipalGetter //委托人
}

//构造policyChecker
func NewPolicyChecker(channelPolicyManagerGetter policies.ChannelPolicyManagerGetter, localMSP msp.IdentityDeserializer, principalGetter mgmt.MSPPrincipalGetter) PolicyChecker
//检查签名提案是否符合通道策略
func (p *policyChecker) CheckPolicy(channelID, policyName string, signedProp *pb.SignedProposal) error
func (p *policyChecker) CheckPolicyNoChannel(policyName string, signedProp *pb.SignedProposal) error
//检查签名数据是否符合通道策略，获取策略并调取policy.Evaluate(sd)
func (p *policyChecker) CheckPolicyBySignedData(channelID, policyName string, sd []*common.SignedData) error
func GetPolicyChecker() PolicyChecker //pcFactory.NewPolicyChecker()
//代码在core/policy/policy.go
```

func (p *policyChecker) CheckPolicy(channelID, policyName string, signedProp *pb.SignedProposal) error代码如下：

```go
func (p *policyChecker) CheckPolicy(channelID, policyName string, signedProp *pb.SignedProposal) error {
	if channelID == "" { //channelID为空，调取CheckPolicyNoChannel()
		return p.CheckPolicyNoChannel(policyName, signedProp)
	}
	
	policyManager, _ := p.channelPolicyManagerGetter.Manager(channelID)
	proposal, err := utils.GetProposal(signedProp.ProposalBytes) //获取proposal
	header, err := utils.GetHeader(proposal.Header)
	shdr, err := utils.GetSignatureHeader(header.SignatureHeader) //SignatureHeader
	sd := []*common.SignedData{&common.SignedData{
		Data:      signedProp.ProposalBytes,
		Identity:  shdr.Creator,
		Signature: signedProp.Signature,
	}}
	return p.CheckPolicyBySignedData(channelID, policyName, sd)
}
//代码在core/policy/policy.go
```

## 4、ChannelPolicyManagerGetter接口及实现

### 4.1、ChannelPolicyManagerGetter接口定义

```go
type ChannelPolicyManagerGetter interface {
	Manager(channelID string) (Manager, bool)
}
//代码在common/policies/policy.go
```

### 4.2、ChannelPolicyManagerGetter接口实现

ChannelPolicyManagerGetter接口实现，即ManagerImpl结构体及方法。

```go
type ManagerImpl struct {
	parent        *ManagerImpl
	basePath      string
	fqPrefix      string
	providers     map[int32]Provider //type Provider interface
	config        *policyConfig //type policyConfig struct
	pendingConfig map[interface{}]*policyConfig //type policyConfig struct
	pendingLock   sync.RWMutex
	SuppressSanityLogMessages bool
}

type Provider interface {
	NewPolicy(data []byte) (Policy, proto.Message, error)
}

type policyConfig struct {
	policies map[string]Policy //type Policy interface
	managers map[string]*ManagerImpl
	imps     []*implicitMetaPolicy
}

type Policy interface {
	//对给定的签名数据，按规则检验确认是否符合约定的条件
	Evaluate(signatureSet []*cb.SignedData) error
}

//构造ManagerImpl
func NewManagerImpl(basePath string, providers map[int32]Provider) *ManagerImpl
//获取pm.basePath
func (pm *ManagerImpl) BasePath() string
//获取pm.config.policies，即map[string]Policy中Key列表
func (pm *ManagerImpl) PolicyNames() []string
//获取指定路径的子管理器
func (pm *ManagerImpl) Manager(path []string) (Manager, bool)
//获取pm.config.policies[relpath]
//获取Policy
func (pm *ManagerImpl) GetPolicy(id string) (Policy, bool)
func (pm *ManagerImpl) BeginPolicyProposals(tx interface{}, groups []string) ([]Proposer, error)
func (pm *ManagerImpl) RollbackProposals(tx interface{})
func (pm *ManagerImpl) PreCommit(tx interface{}) error
func (pm *ManagerImpl) CommitProposals(tx interface{})
func (pm *ManagerImpl) ProposePolicy(tx interface{}, key string, configPolicy *cb.ConfigPolicy) (proto.Message, error)
//代码在common/policies/policy.go
```

```go
type implicitMetaPolicy struct {
	conf        *cb.ImplicitMetaPolicy
	threshold   int
	subPolicies []Policy
}
//代码在common/policies/implicitmeta.go
```