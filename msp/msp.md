# Fabric 1.0源码之旅(3)-MSP

Peer节点启动流程，涉及MSP，本文专门讲解MSP。

## 1、MSP概要

MSP，全称Membership Service Provider，即成员关系服务提供者，作用为管理Fabric中的众多参与者。
MSP的核心代码在msp目录下，其他相关代码分布在common/config/msp、protos/msp下。目录结构如下：

* msp目录
	* msp.go，定义接口MSP、MSPManager、Identity、SigningIdentity、IdentityDeserializer。
	* mspimpl.go，实现MSP接口，即bccspmsp。
	* mspmgrimpl.go，实现MSPManager接口，即mspManagerImpl。
	* identities.go，实现Identity、SigningIdentity接口，即identity和signingidentity。
	* configbuilder.go，提供读取证书文件并将其组装成MSP等接口所需的数据结构，以及转换配置结构体（FactoryOpts->MSPConfig）等工具函数。
	* cert.go，证书相关结构体及方法。
	* mgmt目录
		* mgmt.go，msp相关管理方法实现。
		* principal.go，MSPPrincipalGetter接口及其实现，即localMSPPrincipalGetter。
		* deserializer.go，DeserializersManager接口及其实现，即mspDeserializersManager。
* common/config/msp目录
	* config.go，定义了MSPConfigHandler及其方法，用于配置MSP和configtx工具。
* protos/msp目录，msp相关Protocol Buffer原型文件。

## 2、核心接口定义

IdentityDeserializer为身份反序列化接口，同时被MSP和MSPManger的接口嵌入。定义如下：

```go
type IdentityDeserializer interface {
	DeserializeIdentity(serializedIdentity []byte) (Identity, error)
}
//代码在msp/msp.go
```

MSP接口定义：

```go
type MSP interface {
	IdentityDeserializer //需要实现IdentityDeserializer接口
	Setup(config *msp.MSPConfig) error //根据MSPConfig设置MSP实例
	GetType() ProviderType //获取MSP类型，即FABRIC
	GetIdentifier() (string, error) //获取MSP标识符，即名称
	GetDefaultSigningIdentity() (SigningIdentity, error) //获取默认的签名身份
	GetTLSRootCerts() [][]byte //获取TLS根CA证书
	Validate(id Identity) error //校验身份是否有效
	SatisfiesPrincipal(id Identity, principal *msp.MSPPrincipal) error //验证给定的身份与principal中所描述的类型是否相匹配
}
//代码在msp/msp.go
```

MSPManager接口定义：

```go
type MSPManager interface {
	IdentityDeserializer //需要实现IdentityDeserializer接口
	Setup(msps []MSP) error //用给定的msps填充实例中的mspsMap
	GetMSPs() (map[string]MSP, error) //获取MSP列表，即mspsMap
}
//代码在msp/msp.go
```

Identity接口定义（身份）：

```go
type Identity interface {
	GetIdentifier() *IdentityIdentifier //获取身份ID
	GetMSPIdentifier() string //获取MSP ID，即id.Mspid
	Validate() error //校验身份是否有效，即调取msp.Validate(id)
	GetOrganizationalUnits() []*OUIdentifier //获取组织单元
	Verify(msg []byte, sig []byte) error //用这个身份校验消息签名
	Serialize() ([]byte, error) //身份序列化
	SatisfiesPrincipal(principal *msp.MSPPrincipal) error //调用msp的SatisfiesPrincipal检查身份与principal中所描述的类型是否匹配
}
//代码在msp/msp.go
```

SigningIdentity接口定义（签名身份）：

```go
type SigningIdentity interface {
	Identity //需要实现Identity接口
	Sign(msg []byte) ([]byte, error) //签名msg
}
//代码在msp/msp.go
```

## 3、MSP接口实现




## 6、本文使用到的网络内容

* [fabric源码解析12——peer的MSP服务](http://blog.csdn.net/idsuf698987/article/details/77103011)
* [MSP&ACL](https://hyperledgercn.github.io/hyperledgerDocs/msp_acl_zh/)