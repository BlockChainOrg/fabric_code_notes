# Fabric 1.0源代码笔记 之 scc（系统链码）

## 1、scc概述

scc，system chain codes，即系统链码。包括：
* cscc，configuration system chaincode，处理在peer通道配置。
* escc，endorser system chaincode，对交易申请的应答信息进行签名，来提供背书功能。
* lscc，lifecycle system chaincode，处理生命周期请求，如chaincode的安装，实例化，升级，卸载。
* qscc，querier system chaincode，提供账本查询，如获取块和交易信息。
* vscc，validator system chaincode，处理交易校验，包括检查背书策略和版本在并发时的控制。

[Fabric 1.0源代码笔记 之 scc（系统链码） #cscc（通道相关）](cscc.md)
[Fabric 1.0源代码笔记 之 scc（系统链码） #escc（背书相关）](escc.md)
[Fabric 1.0源代码笔记 之 scc（系统链码） #lscc（链码相关）](lscc.md)
[Fabric 1.0源代码笔记 之 scc（系统链码） #qscc（账本查询相关）](qscc.md)
[Fabric 1.0源代码笔记 之 scc（系统链码） #vscc（策略校验相关）](vscc.md)

scc代码分布在core/common/sysccprovider和core/scc目录下，目录结构如下：

* core/common/sysccprovider目录：
	* sysccprovider.go，SystemChaincodeProvider和SystemChaincodeProviderFactory接口定义。
* core/scc目录：
	* sysccapi.go，SystemChaincode结构体及方法。
	* sccproviderimpl.go，SystemChaincodeProvider和SystemChaincodeProviderFactory接口实现，即sccProviderFactory和sccProviderImpl结构体及方法。
	* importsysccs.go，scc工具函数。

## 2、接口定义

### 2.1、SystemChaincodeProviderFactory接口定义

接口定义如下：

```go
type SystemChaincodeProviderFactory interface {
	//创建SystemChaincodeProvider
	NewSystemChaincodeProvider() SystemChaincodeProvider
}
//代码在core/common/sysccprovider/sysccprovider.go
```

全局变量及相关函数：

```go
var sccFactory SystemChaincodeProviderFactory

//为sccFactory赋值
func RegisterSystemChaincodeProviderFactory(sccfact SystemChaincodeProviderFactory) 
//获取sccFactory.NewSystemChaincodeProvider()
func GetSystemChaincodeProvider() SystemChaincodeProvider {
//代码在core/common/sysccprovider/sysccprovider.go
```

补充ChaincodeInstance结构体定义：

```go
type ChaincodeInstance struct {
	ChainID          string //ID
	ChaincodeName    string //名称
	ChaincodeVersion string //版本
}
//代码在core/common/sysccprovider/sysccprovider.go
```

### 2.2、SystemChaincodeProvider接口定义

接口定义如下：

```go
type SystemChaincodeProvider interface {
	IsSysCC(name string) bool //是否系统链码
	IsSysCCAndNotInvokableCC2CC(name string) bool //确认是系统链码且不可通过CC2CC调用
	IsSysCCAndNotInvokableExternal(name string) bool //确认是系统链码且不可通过提案调用
	GetQueryExecutorForLedger(cid string) (ledger.QueryExecutor, error) //获取账本的查询执行器
}
//代码在core/common/sysccprovider/sysccprovider.go
```

## 3、SystemChaincodeProvider和SystemChaincodeProviderFactory接口实现

SystemChaincodeProviderFactory接口实现，即sccProviderFactory结构体及方法：

```go
type sccProviderFactory struct {
}

//构造sccProviderImpl{}
func (c *sccProviderFactory) NewSystemChaincodeProvider() sysccprovider.SystemChaincodeProvider
//代码在core/scc/sccproviderimpl.go
```

SystemChaincodeProvider接口实现，即sccProviderImpl结构体及方法：

```go
type sccProviderImpl struct {
}

func (c *sccProviderImpl) IsSysCC(name string) bool //IsSysCC(name)
func (c *sccProviderImpl) IsSysCCAndNotInvokableCC2CC(name string) bool //IsSysCCAndNotInvokableCC2CC(name)
//l := peer.GetLedger(cid)
//l.NewQueryExecutor()
func (c *sccProviderImpl) GetQueryExecutorForLedger(cid string) (ledger.QueryExecutor, error)
//IsSysCCAndNotInvokableExternal(name)
func (c *sccProviderImpl) IsSysCCAndNotInvokableExternal(name string) bool
//代码在core/scc/sccproviderimpl.go
```

## 4、scc工具函数

systemChaincodes定义：

```go
var systemChaincodes = []*SystemChaincode{
	{
		Enabled:           true,
		Name:              "cscc",
		Path:              "github.com/hyperledger/fabric/core/scc/cscc",
		InitArgs:          [][]byte{[]byte("")},
		Chaincode:         &cscc.PeerConfiger{},
		InvokableExternal: true, // cscc is invoked to join a channel
	},
	{
		Enabled:           true,
		Name:              "lscc",
		Path:              "github.com/hyperledger/fabric/core/scc/lscc",
		InitArgs:          [][]byte{[]byte("")},
		Chaincode:         &lscc.LifeCycleSysCC{},
		InvokableExternal: true, // lscc is invoked to deploy new chaincodes
		InvokableCC2CC:    true, // lscc can be invoked by other chaincodes
	},
	{
		Enabled:   true,
		Name:      "escc",
		Path:      "github.com/hyperledger/fabric/core/scc/escc",
		InitArgs:  [][]byte{[]byte("")},
		Chaincode: &escc.EndorserOneValidSignature{},
	},
	{
		Enabled:   true,
		Name:      "vscc",
		Path:      "github.com/hyperledger/fabric/core/scc/vscc",
		InitArgs:  [][]byte{[]byte("")},
		Chaincode: &vscc.ValidatorOneValidSignature{},
	},
	{
		Enabled:           true,
		Name:              "qscc",
		Path:              "github.com/hyperledger/fabric/core/chaincode/qscc",
		InitArgs:          [][]byte{[]byte("")},
		Chaincode:         &qscc.LedgerQuerier{},
		InvokableExternal: true, // qscc can be invoked to retrieve blocks
		InvokableCC2CC:    true, // qscc can be invoked to retrieve blocks also by a cc
	},
}
//代码在core/scc/importsysccs.go
```

涉及scc工具函数如下：

```go
func RegisterSysCCs() //遍历systemChaincodes，调用RegisterSysCC(sysCC)
func DeploySysCCs(chainID string)//遍历systemChaincodes，调用deploySysCC(chainID, sysCC)
func DeDeploySysCCs(chainID string)//遍历systemChaincodes，调用DeDeploySysCC(chainID, sysCC)
func IsSysCC(name string) bool //是否系统链码
func IsSysCCAndNotInvokableExternal(name string) bool //确认是系统链码且不可被发送到此节点的提案调用
func IsSysCCAndNotInvokableCC2CC(name string) bool //确认是系统链码且不可通过chaincode-to-chaincode方式调用
//代码在core/scc/importsysccs.go
```

## 5、SystemChaincode结构体及方法

```go
type SystemChaincode struct {
	Name string //系统链码唯一名称
	Path string //系统链码路径，当前未使用
	InitArgs [][]byte //启动系统链码的初始化参数
	Chaincode shim.Chaincode //实际的shim.Chaincode对象
	InvokableExternal bool //跟踪是否可以被发送到此节点的提案调用
	InvokableCC2CC bool //跟踪是否可以通过chaincode-to-chaincode方式调用
	Enabled bool //启用或禁用
}

//注册系统链码，调用inproccontroller.Register(syscc.Path, syscc.Chaincode)
func RegisterSysCC(syscc *SystemChaincode) error
func deploySysCC(chainID string, syscc *SystemChaincode) error //部署链码
func DeDeploySysCC(chainID string, syscc *SystemChaincode) error //停止链码
func buildSysCC(context context.Context, spec *pb.ChaincodeSpec) (*pb.ChaincodeDeploymentSpec, error) //编译链码
func isWhitelisted(syscc *SystemChaincode) bool //是否在白名单，基于chaincode.system配置
//代码在core/scc/sysccapi.go
```