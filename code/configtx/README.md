# Fabric 1.0源代码笔记 之 configtx（配置交易）

## 1、configtx概述

configtx代码分布在common/configtx目录，目录结构如下：

* api目录，核心接口定义，如Manager、Resources、Transactional、PolicyHandler、Proposer、Initializer。
* initializer.go，Resources和Initializer接口实现。

## 2、Resources接口定义及实现

### 2.1、Resources接口定义

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

### 2.2、Resources接口实现

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
