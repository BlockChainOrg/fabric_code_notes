# Fabric 1.0源码之旅(3)-MSP

Peer节点启动流程，涉及MSP，本文专门讲解MSP。

## 1、MSP概要

MSP，全称Membership Service Provider，即成员关系服务提供者，作用为管理Fabric中的众多参与者。

成员服务提供者（MSP）是一个提供抽象化成员操作框架的组件。
MSP将颁发与校验证书，以及用户认证背后的所有密码学机制与协议都抽象了出来。一个MSP可以自己定义身份，以及身份的管理（身份验证）与认证（生成与验证签名）规则。
一个Hyperledger Fabric区块链网络可以被一个或多个MSP管理。

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
	GetIdentifier() (string, error) //获取MSP名字
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

MSP接口实现，即bccspmsp结构体及方法，bccspmsp定义如下：

```go
type bccspmsp struct {
	rootCerts []Identity //信任的CA证书列表
	intermediateCerts []Identity 信任的中间证书列表
	tlsRootCerts [][]byte //信任的CA TLS 证书列表
	tlsIntermediateCerts [][]byte //信任的中间TLS 证书列表
	certificationTreeInternalNodesMap map[string]bool //待定
	signer SigningIdentity //签名身份
	admins []Identity //管理身份列表
	bccsp bccsp.BCCSP //加密服务提供者
	name string //MSP名字
	opts *x509.VerifyOptions //MSP成员验证选项
	CRL []*pkix.CertificateList //证书吊销列表
	ouIdentifiers map[string][][]byte //组织列表
	cryptoConfig *m.FabricCryptoConfig //加密选项
}
//代码在msp/mspimpl.go
```

涉及方法如下：

```go
func NewBccspMsp() (MSP, error) //创建bccsp实例，以及创建并初始化bccspmsp实例
func (msp *bccspmsp) Setup(conf1 *m.MSPConfig) error ////根据MSPConfig设置MSP实例
func (msp *bccspmsp) GetType() ProviderType //获取MSP类型，即FABRIC
func (msp *bccspmsp) GetIdentifier() (string, error) //获取MSP名字
func (msp *bccspmsp) GetTLSRootCerts() [][]byte //获取信任的CA TLS 证书列表msp.tlsRootCerts
func (msp *bccspmsp) GetTLSIntermediateCerts() [][]byte //获取信任的中间TLS 证书列表msp.tlsIntermediateCerts
func (msp *bccspmsp) GetDefaultSigningIdentity() (SigningIdentity, error) ////获取默认的签名身份msp.signer
func (msp *bccspmsp) GetSigningIdentity(identifier *IdentityIdentifier) (SigningIdentity, error) //暂未实现，可忽略
func (msp *bccspmsp) Validate(id Identity) error //校验身份是否有效，调取msp.validateIdentity(id)实现
func (msp *bccspmsp) DeserializeIdentity(serializedID []byte) (Identity, error) //身份反序列化
func (msp *bccspmsp) SatisfiesPrincipal(id Identity, principal *m.MSPPrincipal) error //验证给定的身份与principal中所描述的类型是否相匹配
//代码在msp/mspimpl.go
```

func (msp *bccspmsp) Setup(conf1 *m.MSPConfig) error代码如下：

```go
conf := &m.FabricMSPConfig{}
err := proto.Unmarshal(conf1.Config, conf) //将conf1.Config []byte解码为FabricMSPConfig
msp.name = conf.Name
err := msp.setupCrypto(conf) //设置加密选项msp.cryptoConfig
err := msp.setupCAs(conf) //设置MSP成员验证选项msp.opts，并添加信任的CA证书msp.rootCerts和信任的中间证书msp.intermediateCerts
err := msp.setupAdmins(conf) //设置管理身份列表msp.admins
err := msp.setupCRLs(conf) //设置证书吊销列表msp.CRL
err := msp.finalizeSetupCAs(conf); err != nil //设置msp.certificationTreeInternalNodesMap
err := msp.setupSigningIdentity(conf) //设置签名身份msp.signer
err := msp.setupOUs(conf) //设置组织列表msp.ouIdentifiers
err := msp.setupTLSCAs(conf) //设置并添加信任的CA TLS 证书列表msp.tlsRootCerts，以及信任的CA TLS 证书列表msp.tlsIntermediateCerts
for i, admin := range msp.admins {
	err = admin.Validate() //确保管理员是有效的成员
}
//代码在msp/mspimpl.go
```

func (msp *bccspmsp) validateIdentity(id *identity)代码如下：

```go
validationChain, err := msp.getCertificationChainForBCCSPIdentity(id) //获取BCCSP身份认证链
err = msp.validateIdentityAgainstChain(id, validationChain) //根据链验证身份
err = msp.validateIdentityOUs(id) //验证身份中所携带的组织信息有效
//代码在msp/mspimpl.go
```

## 4、MSPManager接口实现








## 6、本文使用到的网络内容

* [成员服务提供者（MSP）](https://hyperledgercn.github.io/hyperledgerDocs/msp_zh/)
* [MSP&ACL](https://hyperledgercn.github.io/hyperledgerDocs/msp_acl_zh/)
* [fabric源码解析12——peer的MSP服务](http://blog.csdn.net/idsuf698987/article/details/77103011)
* [fabric源码解析9——文档翻译之MSP](http://blog.csdn.net/idsuf698987/article/details/76474094)