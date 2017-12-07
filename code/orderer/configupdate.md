# Fabric 1.0源代码笔记 之 Orderer #configupdate（处理通道配置更新）

## 1、configupdate概述

configupdate，用于接收配置交易，并处理通道配置更新。
相关代码在orderer/configupdate目录。

## 2、SupportManager接口定义及实现

### 2.1、SupportManager接口定义

```go
type SupportManager interface {
	GetChain(chainID string) (Support, bool)
	NewChannelConfig(envConfigUpdate *cb.Envelope) (configtxapi.Manager, error)
}
//代码在orderer/configupdate/configupdate.go
```

### 2.2、SupportManager接口实现

SupportManager接口实现，即configUpdateSupport结构体及方法。

```go
type configUpdateSupport struct {
	multichain.Manager //type multiLedger struct
}

func (cus configUpdateSupport) GetChain(chainID string) (configupdate.Support, bool) {
	return cus.Manager.GetChain(chainID)
}
//代码在orderer/server.go
```

multichain.Manager接口及实现multiLedger，见[Fabric 1.0源代码笔记 之 Orderer #multichain（多链支持包）](multichain.md)

## 3、Support接口定义及实现

### 3.1、Support接口定义

```go
type Support interface {
	ProposeConfigUpdate(env *cb.Envelope) (*cb.ConfigEnvelope, error)
}
//代码在orderer/configupdate/configupdate.go
```

### 3.2、Support接口实现

Support接口实现，即configManager结构体及方法。

```go
type configManager struct {
	api.Resources
	callOnUpdate []func(api.Manager)
	initializer  api.Initializer
	current      *configSet
}

func (cm *configManager) processConfig(channelGroup *cb.ConfigGroup) (*configResult, error) //./config.go
func (cm *configManager) commitCallbacks() //./manager.go
func (cm *configManager) ProposeConfigUpdate(configtx *cb.Envelope) (*cb.ConfigEnvelope, error) //./manager.go
func (cm *configManager) proposeConfigUpdate(configtx *cb.Envelope) (*cb.ConfigEnvelope, error) //./manager.go
func (cm *configManager) prepareApply(configEnv *cb.ConfigEnvelope) (*configResult, error) //./manager.go
func (cm *configManager) Validate(configEnv *cb.ConfigEnvelope) error //./manager.go
func (cm *configManager) Apply(configEnv *cb.ConfigEnvelope) error //./manager.go
func (cm *configManager) ChainID() string //./manager.go
func (cm *configManager) Sequence() uint64 //./manager.go
func (cm *configManager) ConfigEnvelope() *cb.ConfigEnvelope //./manager.go
func (cm *configManager) verifyDeltaSet(deltaSet map[string]comparable, signedData []*cb.SignedData) error //./update.go
func (cm *configManager) authorizeUpdate(configUpdateEnv *cb.ConfigUpdateEnvelope) (map[string]comparable, error) //./update.go
func (cm *configManager) policyForItem(item comparable) (policies.Policy, bool) //./update.go
func (cm *configManager) computeUpdateResult(updatedConfig map[string]comparable) map[string]comparable //./update.go
//代码在common/configtx/manager.go
```

## 4、ConfigUpdateProcessor接口定义及实现

### 4.1、ConfigUpdateProcessor接口定义

```go
type ConfigUpdateProcessor interface {
	Process(envConfigUpdate *cb.Envelope) (*cb.Envelope, error)
}
//代码在orderer/common/broadcast/broadcast.go
```

### 4.2、ConfigUpdateProcessor接口实现

ConfigUpdateProcessor接口实现，即Processor结构体及方法。

```go
type Processor struct {
	signer               crypto.LocalSigner
	manager              SupportManager //即type configUpdateSupport struct，或者即multichain.multiLedger
	systemChannelID      string
	systemChannelSupport Support
}

//构造Processor
func New(systemChannelID string, supportManager SupportManager, signer crypto.LocalSigner) *Processor
//获取channelID
func channelID(env *cb.Envelope) (string, error) 
//处理通道配置更新
func (p *Processor) Process(envConfigUpdate *cb.Envelope) (*cb.Envelope, error)
func (p *Processor) existingChannelConfig(envConfigUpdate *cb.Envelope, channelID string, support Support) (*cb.Envelope, error)
func (p *Processor) proposeNewChannelToSystemChannel(newChannelEnvConfig *cb.Envelope) (*cb.Envelope, error)
func (p *Processor) newChannelConfig(channelID string, envConfigUpdate *cb.Envelope) (*cb.Envelope, error)
//代码在orderer/configupdate/configupdate.go
```

#### 4.2.1、func New(systemChannelID string, supportManager SupportManager, signer crypto.LocalSigner) *Processor

```go
func New(systemChannelID string, supportManager SupportManager, signer crypto.LocalSigner) *Processor {
	support, ok := supportManager.GetChain(systemChannelID)
	return &Processor{
		systemChannelID:      systemChannelID,
		manager:              supportManager,
		signer:               signer,
		systemChannelSupport: support,
	}
}
//代码在orderer/configupdate/configupdate.go
```

#### 4.2.2、func (p *Processor) Process(envConfigUpdate *cb.Envelope) (*cb.Envelope, error)

```go
func (p *Processor) Process(envConfigUpdate *cb.Envelope) (*cb.Envelope, error) {
	channelID, err := channelID(envConfigUpdate)
	support, ok := p.manager.GetChain(channelID) //存在
	if ok {
		return p.existingChannelConfig(envConfigUpdate, channelID, support)
	}
	return p.newChannelConfig(channelID, envConfigUpdate) //不存在
}
//代码在orderer/configupdate/configupdate.go
```

#### 4.2.3、func (p *Processor) existingChannelConfig(envConfigUpdate *cb.Envelope, channelID string, support Support) (*cb.Envelope, error)

```go
func (p *Processor) existingChannelConfig(envConfigUpdate *cb.Envelope, channelID string, support Support) (*cb.Envelope, error) {
	configEnvelope, err := support.ProposeConfigUpdate(envConfigUpdate)
	return utils.CreateSignedEnvelope(cb.HeaderType_CONFIG, channelID, p.signer, configEnvelope, msgVersion, epoch)
}
//代码在orderer/configupdate/configupdate.go
```

#### 4.2.4、func (p *Processor) newChannelConfig(channelID string, envConfigUpdate *cb.Envelope) (*cb.Envelope, error)

```go
func (p *Processor) newChannelConfig(channelID string, envConfigUpdate *cb.Envelope) (*cb.Envelope, error) {
	ctxm, err := p.manager.NewChannelConfig(envConfigUpdate) //创建新的通道
	newChannelConfigEnv, err := ctxm.ProposeConfigUpdate(envConfigUpdate) //创建新的通道后处理通道配置
	newChannelEnvConfig, err := utils.CreateSignedEnvelope(cb.HeaderType_CONFIG, channelID, p.signer, newChannelConfigEnv, msgVersion, epoch)
	return p.proposeNewChannelToSystemChannel(newChannelEnvConfig)
}
//代码在orderer/configupdate/configupdate.go
```

## 5、详解configManager结构体

### 5.1、configManager结构体定义及方法

```go
type configManager struct {
	api.Resources
	callOnUpdate []func(api.Manager)
	initializer  api.Initializer
	current      *configSet
}

func validateConfigID(configID string) error
func validateChannelID(channelID string) error
func NewManagerImpl(envConfig *cb.Envelope, initializer api.Initializer, callOnUpdate []func(api.Manager)) (api.Manager, error)
func (cm *configManager) commitCallbacks()
func (cm *configManager) ProposeConfigUpdate(configtx *cb.Envelope) (*cb.ConfigEnvelope, error)
func (cm *configManager) proposeConfigUpdate(configtx *cb.Envelope) (*cb.ConfigEnvelope, error)
func (cm *configManager) prepareApply(configEnv *cb.ConfigEnvelope) (*configResult, error)
func (cm *configManager) Validate(configEnv *cb.ConfigEnvelope) error
func (cm *configManager) Apply(configEnv *cb.ConfigEnvelope) error
func (cm *configManager) ChainID() string
func (cm *configManager) Sequence() uint64
func (cm *configManager) ConfigEnvelope() *cb.ConfigEnvelope

func proposeGroup(result *configResult) error
func processConfig(channelGroup *cb.ConfigGroup, proposer api.Proposer) (*configResult, error)
func (cm *configManager) processConfig(channelGroup *cb.ConfigGroup) (*configResult, error)

func (c *configSet) verifyReadSet(readSet map[string]comparable) error
func ComputeDeltaSet(readSet, writeSet map[string]comparable) map[string]comparable
func validateModPolicy(modPolicy string) error
func (cm *configManager) verifyDeltaSet(deltaSet map[string]comparable, signedData []*cb.SignedData) error
func verifyFullProposedConfig(writeSet, fullProposedConfig map[string]comparable) error
//验证所有修改的配置都有相应的修改策略，返回修改过的配置的映射map[string]comparable
func (cm *configManager) authorizeUpdate(configUpdateEnv *cb.ConfigUpdateEnvelope) (map[string]comparable, error)
func (cm *configManager) policyForItem(item comparable) (policies.Policy, bool)
func (cm *configManager) computeUpdateResult(updatedConfig map[string]comparable) map[string]comparable
//Envelope转换为ConfigUpdateEnvelope
func envelopeToConfigUpdate(configtx *cb.Envelope) (*cb.ConfigUpdateEnvelope, error)
//代码在common/configtx/manager.go
```

### 5.2、func (cm *configManager) ProposeConfigUpdate(configtx *cb.Envelope) (*cb.ConfigEnvelope, error)

```go
func (cm *configManager) ProposeConfigUpdate(configtx *cb.Envelope) (*cb.ConfigEnvelope, error) {
	return cm.proposeConfigUpdate(configtx)
}

func (cm *configManager) proposeConfigUpdate(configtx *cb.Envelope) (*cb.ConfigEnvelope, error) {
	//Envelope转换为ConfigUpdateEnvelope
	configUpdateEnv, err := envelopeToConfigUpdate(configtx)
	//验证所有修改的配置都有相应的修改策略，返回修改过的配置的映射map[string]comparable
	configMap, err := cm.authorizeUpdate(configUpdateEnv)
	channelGroup, err := configMapToConfig(configMap) //ConfigGroup
	//实际调用processConfig(channelGroup, cm.initializer)，并最终调用proposeGroup(configResult)
	result, err := cm.processConfig(channelGroup)
	result.rollback()
	return &cb.ConfigEnvelope{
		Config: &cb.Config{
			Sequence:     cm.current.sequence + 1,
			ChannelGroup: channelGroup,
		},
		LastUpdate: configtx,
	}, nil
}
//代码在common/configtx/manager.go
```

补充ConfigUpdateEnvelope:

```go
type ConfigUpdateEnvelope struct {
	ConfigUpdate []byte //type ConfigUpdate struct
	Signatures   []*ConfigSignature
}

type ConfigUpdate struct {
	ChannelId string
	ReadSet   *ConfigGroup
	WriteSet  *ConfigGroup
}

type ConfigGroup struct {
	Version   uint64
	Groups    map[string]*ConfigGroup
	Values    map[string]*ConfigValue
	Policies  map[string]*ConfigPolicy
	ModPolicy string
}
//代码在protos/common/configtx.pb.go
```

补充ConfigGroup:

```go


### 5.3、func (cm *configManager) authorizeUpdate(configUpdateEnv *cb.ConfigUpdateEnvelope) (map[string]comparable, error)

```go
func (cm *configManager) authorizeUpdate(configUpdateEnv *cb.ConfigUpdateEnvelope) (map[string]comparable, error) {
	//反序列化configUpdateEnv.ConfigUpdate
	configUpdate, err := UnmarshalConfigUpdate(configUpdateEnv.ConfigUpdate)

	readSet, err := MapConfig(configUpdate.ReadSet) //map[string]comparable
	err = cm.current.verifyReadSet(readSet)

	writeSet, err := MapConfig(configUpdate.WriteSet) //map[string]comparable

	//从writeSet中逐一对比readSet，去除没有发生变更的
	deltaSet := ComputeDeltaSet(readSet, writeSet) 
	signedData, err := configUpdateEnv.AsSignedData() //转换为SignedData

	err = cm.verifyDeltaSet(deltaSet, signedData) //校验deltaSet

	fullProposedConfig := cm.computeUpdateResult(deltaSet) //合并为fullProposedConfig
	err := verifyFullProposedConfig(writeSet, fullProposedConfig)
	return fullProposedConfig, nil
}
//代码在common/configtx/update.go
```

补充comparable:

```go
type comparable struct {
	*cb.ConfigGroup
	*cb.ConfigValue
	*cb.ConfigPolicy
	key  string
	path []string
}
//代码在common/configtx/compare.go
```

### 5.4、func (cm *configManager) processConfig(channelGroup *cb.ConfigGroup) (*configResult, error)

```go
func (cm *configManager) processConfig(channelGroup *cb.ConfigGroup) (*configResult, error) {
	configResult, err := processConfig(channelGroup, cm.initializer)
	err = configResult.preCommit()
	return configResult, nil
}
//代码在common/configtx/config.go
```

补充configResult：

```go
type configResult struct {
	tx                   interface{}
	groupName            string
	groupKey             string
	group                *cb.ConfigGroup
	valueHandler         config.ValueProposer
	policyHandler        policies.Proposer
	subResults           []*configResult
	deserializedValues   map[string]proto.Message
	deserializedPolicies map[string]proto.Message
}

func NewConfigResult(config *cb.ConfigGroup, proposer api.Proposer) (ConfigResult, error)
func (cr *configResult) JSON() string
func (cr *configResult) bufferJSON(buffer *bytes.Buffer)
//cr.valueHandler.PreCommit(cr.tx)
func (cr *configResult) preCommit() error
//cr.valueHandler.CommitProposals(cr.tx)
//cr.policyHandler.CommitProposals(cr.tx)
func (cr *configResult) commit()
//cr.valueHandler.RollbackProposals(cr.tx)
//cr.policyHandler.RollbackProposals(cr.tx)
func (cr *configResult) rollback()
func proposeGroup(result *configResult) error
func processConfig(channelGroup *cb.ConfigGroup, proposer api.Proposer) (*configResult, error)
//代码在common/configtx/config.go
```

#### 5.4.1、func processConfig(channelGroup *cb.ConfigGroup, proposer api.Proposer) (*configResult, error)

```go
func processConfig(channelGroup *cb.ConfigGroup, proposer api.Proposer) (*configResult, error) {
	helperGroup := cb.NewConfigGroup()
	helperGroup.Groups[RootGroupKey] = channelGroup

	configResult := &configResult{
		group:         helperGroup,
		valueHandler:  proposer.ValueProposer(),
		policyHandler: proposer.PolicyProposer(),
	}
	err := proposeGroup(configResult)
	return configResult, nil
}
//代码在common/configtx/config.go
```

#### 5.4.2、func proposeGroup(result *configResult) error

```go
func proposeGroup(result *configResult) error {
	subGroups := make([]string, len(result.group.Groups))
	i := 0
	for subGroup := range result.group.Groups {
		subGroups[i] = subGroup
		i++
	}

	valueDeserializer, subValueHandlers, err := result.valueHandler.BeginValueProposals(result.tx, subGroups)
	subPolicyHandlers, err := result.policyHandler.BeginPolicyProposals(result.tx, subGroups)

	for key, value := range result.group.Values {
		msg, err := valueDeserializer.Deserialize(key, value.Value)
		result.deserializedValues[key] = msg
	}

	for key, policy := range result.group.Policies {
		policy, err := result.policyHandler.ProposePolicy(result.tx, key, policy)
		result.deserializedPolicies[key] = policy
	}

	result.subResults = make([]*configResult, 0, len(subGroups))

	for i, subGroup := range subGroups {
		result.subResults = append(result.subResults, &configResult{
			tx:                   result.tx,
			groupKey:             subGroup,
			groupName:            result.groupName + "/" + subGroup,
			group:                result.group.Groups[subGroup],
			valueHandler:         subValueHandlers[i],
			policyHandler:        subPolicyHandlers[i],
			deserializedValues:   make(map[string]proto.Message),
			deserializedPolicies: make(map[string]proto.Message),
		})
		err := proposeGroup(result.subResults[i])
	}
	return nil
}
//代码在common/configtx/config.go
```
