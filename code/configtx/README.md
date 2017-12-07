# Fabric 1.0源代码笔记 之 configtx（配置交易）

## 1、configtx概述

configtx代码分布在common/configtx目录，目录结构如下：

* api目录，核心接口定义，如Manager、Resources、Transactional、PolicyHandler、Proposer、Initializer。
* initializer.go，Resources和Initializer接口实现。
* template.go，Template接口定义及实现。
* config.go，ConfigResult接口定义及实现。
* manager.go/update.go，Manager接口实现。
* util.go，configtx工具函数。

## 2、Template接口定义及实现

### 2.1、Template接口定义

```go
type Template interface {
	Envelope(chainID string) (*cb.ConfigUpdateEnvelope, error)
}
//代码在common/configtx/template.go
```

ConfigUpdateEnvelope定义：

```go
type ConfigUpdateEnvelope struct {
	ConfigUpdate []byte //type ConfigUpdate struct
	Signatures   []*ConfigSignature //type ConfigSignature struct
}

type ConfigUpdate struct {
	ChannelId string
	ReadSet   *ConfigGroup //type ConfigGroup struct
	WriteSet  *ConfigGroup //type ConfigGroup struct
}

type ConfigSignature struct {
	SignatureHeader []byte
	Signature       []byte
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

### 2.2、Template接口实现（simpleTemplate）

```go
type simpleTemplate struct {
	configGroup *cb.ConfigGroup
}

func NewSimpleTemplate(configGroups ...*cb.ConfigGroup) Template {
	sts := make([]Template, len(configGroups))
	for i, group := range configGroups {
		sts[i] = &simpleTemplate{
			configGroup: group,
		}
	}
	return NewCompositeTemplate(sts...)
}

func (st *simpleTemplate) Envelope(chainID string) (*cb.ConfigUpdateEnvelope, error) {
	config, err := proto.Marshal(&cb.ConfigUpdate{
		ChannelId: chainID,
		WriteSet:  st.configGroup,
	})
	return &cb.ConfigUpdateEnvelope{
		ConfigUpdate: config,
	}, nil
}
//代码在common/configtx/template.go
```

### 2.3、Template接口实现（compositeTemplate）

```go
type compositeTemplate struct {
	templates []Template
}

func NewCompositeTemplate(templates ...Template) Template {
	return &compositeTemplate{templates: templates}
}

func (ct *compositeTemplate) Envelope(chainID string) (*cb.ConfigUpdateEnvelope, error) {
	channel := cb.NewConfigGroup()
	for i := range ct.templates {
		configEnv, err := ct.templates[i].Envelope(chainID)
		config, err := UnmarshalConfigUpdate(configEnv.ConfigUpdate)
		err = copyGroup(config.WriteSet, channel)
	}

	marshaledConfig, err := proto.Marshal(&cb.ConfigUpdate{
		ChannelId: chainID,
		WriteSet:  channel,
	})
	return &cb.ConfigUpdateEnvelope{ConfigUpdate: marshaledConfig}, nil
}
//代码在common/configtx/template.go
```

### 2.4、Template接口实现（modPolicySettingTemplate）

```go
type modPolicySettingTemplate struct {
	modPolicy string
	template  Template
}

func NewModPolicySettingTemplate(modPolicy string, template Template) Template {
	return &modPolicySettingTemplate{
		modPolicy: modPolicy,
		template:  template,
	}
}

func (mpst *modPolicySettingTemplate) Envelope(channelID string) (*cb.ConfigUpdateEnvelope, error) {
	configUpdateEnv, err := mpst.template.Envelope(channelID)
	config, err := UnmarshalConfigUpdate(configUpdateEnv.ConfigUpdate)
	setGroupModPolicies(mpst.modPolicy, config.WriteSet)
	configUpdateEnv.ConfigUpdate = utils.MarshalOrPanic(config)
	return configUpdateEnv, nil
}
//代码在common/configtx/template.go
```

## 3、configtx工具函数

```go
func UnmarshalConfig(data []byte) (*cb.Config, error)
func UnmarshalConfigOrPanic(data []byte) *cb.Config
func UnmarshalConfigUpdate(data []byte) (*cb.ConfigUpdate, error)
func UnmarshalConfigUpdateOrPanic(data []byte) *cb.ConfigUpdate
func UnmarshalConfigUpdateEnvelope(data []byte) (*cb.ConfigUpdateEnvelope, error)
func UnmarshalConfigUpdateEnvelopeOrPanic(data []byte) *cb.ConfigUpdateEnvelope
func UnmarshalConfigEnvelope(data []byte) (*cb.ConfigEnvelope, error)
func UnmarshalConfigEnvelopeOrPanic(data []byte) *cb.ConfigEnvelope
//代码在common/configtx/util.go
```

## 4、Resources接口定义及实现

### 4.1、Resources接口定义

```go
type Resources interface {
	PolicyManager() policies.Manager //获取通道策略管理器，即policies.Manager
	ChannelConfig() config.Channel //获取通道配置
	OrdererConfig() (config.Orderer, bool) //获取Orderer配置
	ConsortiumsConfig() (config.Consortiums, bool) //获取config.Consortiums
	ApplicationConfig() (config.Application, bool) //获取config.Application
	MSPManager() msp.MSPManager //获取通道msp管理器，即msp.MSPManager
}
//代码在common/configtx/api/api.go
```

### 4.2、Resources接口实现

Resources接口实现，即resources结构体及方法。

```go
type resources struct {
	policyManager    *policies.ManagerImpl
	configRoot       *config.Root
	mspConfigHandler *configtxmsp.MSPConfigHandler
}
//代码在common/configtx/initializer.go
```

涉及方法如下：

```go
//获取r.policyManager
func (r *resources) PolicyManager() policies.Manager
//获取r.configRoot.Channel()
func (r *resources) ChannelConfig() config.Channel
//获取r.configRoot.Orderer()
func (r *resources) OrdererConfig() (config.Orderer, bool)
//获取r.configRoot.Application()
func (r *resources) ApplicationConfig() (config.Application, bool)
//获取r.configRoot.Consortiums()
func (r *resources) ConsortiumsConfig() (config.Consortiums, bool)
//获取r.mspConfigHandler
func (r *resources) MSPManager() msp.MSPManager
//构造resources
func newResources() *resources
//代码在common/configtx/initializer.go
```

config更详细内容参考：[Fabric 1.0源代码笔记 之 configtx（配置交易） #ChannelConfig（通道配置）](ChannelConfig.md)
