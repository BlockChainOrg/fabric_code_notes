# Fabric 1.0源代码笔记 之 Chaincode（链码）

## 1、Chaincode概述

Chaincode，即链码或智能合约，代码分布在protos/peer目录、core/chaincode和core/common/ccprovider目录，目录结构如下：

* protos/peer目录：
	* chaincode.pb.go，ChaincodeDeploymentSpec、ChaincodeInvocationSpec结构体定义。
* core/chaincode目录：
	* platforms目录，链码的编写语言平台实现，如golang或java。
		* platforms.go，Platform接口定义，及部分工具函数。
		* java目录，java语言平台实现。
		* golang目录，golang语言平台实现。
* core/common/ccprovider目录：ccprovider相关实现。

## 2、protos相关结构体定义

### 2.1、ChaincodeDeploymentSpec结构体定义（用于Chaincode部署）

#### 2.1.1 ChaincodeDeploymentSpec结构体定义

```go
type ChaincodeDeploymentSpec struct {
	ChaincodeSpec *ChaincodeSpec //ChaincodeSpec消息
	EffectiveDate *google_protobuf1.Timestamp
	CodePackage   []byte //链码文件打包
	ExecEnv       ChaincodeDeploymentSpec_ExecutionEnvironment //链码执行环境，DOCKER或SYSTEM
}

type ChaincodeDeploymentSpec_ExecutionEnvironment int32

const (
	ChaincodeDeploymentSpec_DOCKER ChaincodeDeploymentSpec_ExecutionEnvironment = 0
	ChaincodeDeploymentSpec_SYSTEM ChaincodeDeploymentSpec_ExecutionEnvironment = 1
)
//代码在protos/peer/chaincode.pb.go
```

#### 2.1.2、ChaincodeSpec结构体定义

```go
type ChaincodeSpec struct {
	Type        ChaincodeSpec_Type //链码的编写语言，GOLANG、JAVA
	ChaincodeId *ChaincodeID //ChaincodeId，链码路径、链码名称、链码版本
	Input       *ChaincodeInput //链码的具体执行参数信息
	Timeout     int32
}

type ChaincodeSpec_Type int32

const (
	ChaincodeSpec_UNDEFINED ChaincodeSpec_Type = 0
	ChaincodeSpec_GOLANG    ChaincodeSpec_Type = 1
	ChaincodeSpec_NODE      ChaincodeSpec_Type = 2
	ChaincodeSpec_CAR       ChaincodeSpec_Type = 3
	ChaincodeSpec_JAVA      ChaincodeSpec_Type = 4
)

type ChaincodeID struct {
	Path string
	Name string
	Version string
}

type ChaincodeInput struct { //链码的具体执行参数信息
	Args [][]byte
}
//代码在protos/peer/chaincode.pb.go
```

### 2.2、ChaincodeInvocationSpec结构体定义

```go
type ChaincodeInvocationSpec struct {
	ChaincodeSpec *ChaincodeSpec //参考本文2.2
	IdGenerationAlg string
}
//代码在protos/peer/chaincode.pb.go
```

## 3、ccprovider目录相关实现

### 3.1、ChaincodeData结构体

```go
type ChaincodeData struct {
	Name string
	Version string
	Escc string
	Vscc string
	Policy []byte //chaincode 实例的背书策略
	Data []byte
	Id []byte
	InstantiationPolicy []byte //实例化策略
}

//获取ChaincodeData，优先从缓存中读取
func GetChaincodeData(ccname string, ccversion string) (*ChaincodeData, error)
//代码在core/common/ccprovider/ccprovider.go
```

func GetChaincodeData(ccname string, ccversion string) (*ChaincodeData, error)代码如下：

```go
var ccInfoFSProvider = &CCInfoFSImpl{}
var ccInfoCache = NewCCInfoCache(ccInfoFSProvider)

func GetChaincodeData(ccname string, ccversion string) (*ChaincodeData, error) {
	//./peer/node/start.go:	ccprovider.EnableCCInfoCache()
	//如果启用ccInfoCache，优先从缓存中读取ChaincodeData
	if ccInfoCacheEnabled { 
		return ccInfoCache.GetChaincodeData(ccname, ccversion)
	}
	ccpack, err := ccInfoFSProvider.GetChaincode(ccname, ccversion)
	return ccpack.GetChaincodeData(), nil
}
//代码在core/common/ccprovider/ccprovider.go
```

### 3.2、ccInfoCacheImpl结构体

```go
type ccInfoCacheImpl struct {
	sync.RWMutex
	cache        map[string]*ChaincodeData //ChaincodeData
	cacheSupport CCCacheSupport
}

//构造ccInfoCacheImpl
func NewCCInfoCache(cs CCCacheSupport) *ccInfoCacheImpl
//获取ChaincodeData，优先从c.cache中获取，如果c.cache中没有，则从c.cacheSupport（即CCInfoFSImpl）中获取并写入c.cache
func (c *ccInfoCacheImpl) GetChaincodeData(ccname string, ccversion string) (*ChaincodeData, error) 
//代码在core/common/ccprovider/ccinfocache.go
```

func (c *ccInfoCacheImpl) GetChaincodeData(ccname string, ccversion string) (*ChaincodeData, error) 代码如下：

```go
func (c *ccInfoCacheImpl) GetChaincodeData(ccname string, ccversion string) (*ChaincodeData, error) {
	key := ccname + "/" + ccversion
	c.RLock()
	ccdata, in := c.cache[key] //优先从c.cache中获取
	c.RUnlock()

	if !in { //如果c.cache中没有
		var err error
		//从c.cacheSupport中获取
		ccpack, err := c.cacheSupport.GetChaincode(ccname, ccversion)
		c.Lock()
		//并写入c.cache
		ccdata = ccpack.GetChaincodeData()
		c.cache[key] = ccdata
		c.Unlock()
	}
	return ccdata, nil
}
//代码在core/common/ccprovider/ccinfocache.go
```

### 3.3、CCCacheSupport接口定义及实现

#### 3.3.1、CCCacheSupport接口定义

```go
type CCCacheSupport interface {
	//获取Chaincode包
	GetChaincode(ccname string, ccversion string) (CCPackage, error)
}
//代码在core/common/ccprovider/ccprovider.go
```

#### 3.3.2、CCCacheSupport接口实现（即CCInfoFSImpl结构体）

```go
type CCInfoFSImpl struct{}

//从文件系统中读取并构造CDSPackage或SignedCDSPackage
func (*CCInfoFSImpl) GetChaincode(ccname string, ccversion string) (CCPackage, error) {
	cccdspack := &CDSPackage{}
	_, _, err := cccdspack.InitFromFS(ccname, ccversion)
	if err != nil {
		//try signed CDS
		ccscdspack := &SignedCDSPackage{}
		_, _, err = ccscdspack.InitFromFS(ccname, ccversion)
		if err != nil {
			return nil, err
		}
		return ccscdspack, nil
	}
	return cccdspack, nil
}

//将ChaincodeDeploymentSpec序列化后写入文件系统
func (*CCInfoFSImpl) PutChaincode(depSpec *pb.ChaincodeDeploymentSpec) (CCPackage, error) {
	buf, err := proto.Marshal(depSpec)
	cccdspack := &CDSPackage{}
	_, err := cccdspack.InitFromBuffer(buf)
	err = cccdspack.PutChaincodeToFS()
}
//代码在core/common/ccprovider/ccprovider.go
```

### 3.4、CCPackage接口定义及实现

#### 3.4.1、CCPackage接口定义

```go
type CCPackage interface {
	//从字节初始化包
	InitFromBuffer(buf []byte) (*ChaincodeData, error)
	//从文件系统初始化包
	InitFromFS(ccname string, ccversion string) ([]byte, *pb.ChaincodeDeploymentSpec, error)
	//将chaincode包写入文件系统
	PutChaincodeToFS() error
	//从包中获取ChaincodeDeploymentSpec
	GetDepSpec() *pb.ChaincodeDeploymentSpec
	//从包中获取序列化的ChaincodeDeploymentSpec
	GetDepSpecBytes() []byte
	//校验ChaincodeData
	ValidateCC(ccdata *ChaincodeData) error
	//包转换为proto.Message
	GetPackageObject() proto.Message
	//获取ChaincodeData
	GetChaincodeData() *ChaincodeData
	//基于包计算获取chaincode Id
	GetId() []byte
}
//代码在core/common/ccprovider/ccprovider.go
```

#### 3.4.2、CCPackage接口实现（CDSPackage）

```go
type CDSData struct {
	CodeHash []byte //ChaincodeDeploymentSpec.CodePackage哈希
	MetaDataHash []byte //ChaincodeSpec.ChaincodeId.Name和ChaincodeSpec.ChaincodeId.Version哈希
}

type CDSPackage struct {
	buf     []byte //ChaincodeDeploymentSpec哈希
	depSpec *pb.ChaincodeDeploymentSpec //ChaincodeDeploymentSpec
	data    *CDSData
	datab   []byte
	id      []byte //id为CDSData.CodeHash和CDSData.MetaDataHash求哈希
}

//获取ccpack.id
func (ccpack *CDSPackage) GetId() []byte
//获取ccpack.depSpec
func (ccpack *CDSPackage) GetDepSpec() *pb.ChaincodeDeploymentSpec
//获取ccpack.buf，即ChaincodeDeploymentSpec哈希
func (ccpack *CDSPackage) GetDepSpecBytes() []byte
//获取ccpack.depSpec
func (ccpack *CDSPackage) GetPackageObject() proto.Message
//构造ChaincodeData
func (ccpack *CDSPackage) GetChaincodeData() *ChaincodeData
//获取ChaincodeDeploymentSpec哈希、CDSData、id
func (ccpack *CDSPackage) getCDSData(cds *pb.ChaincodeDeploymentSpec) ([]byte, []byte, *CDSData, error)
//校验CDSPackage和ChaincodeData
func (ccpack *CDSPackage) ValidateCC(ccdata *ChaincodeData) error
//[]byte反序列化为ChaincodeDeploymentSpec，构造CDSPackage，进而构造ChaincodeData
func (ccpack *CDSPackage) InitFromBuffer(buf []byte) (*ChaincodeData, error)
//从文件系统中获取ChaincodeData，即buf, err := GetChaincodePackage(ccname, ccversion)和_, err = ccpack.InitFromBuffer(buf)
func (ccpack *CDSPackage) InitFromFS(ccname string, ccversion string) ([]byte, *pb.ChaincodeDeploymentSpec, error)
//ccpack.buf写入文件，文件名为/var/hyperledger/production/chaincodes/Name.Version
func (ccpack *CDSPackage) PutChaincodeToFS() error
//代码在core/common/ccprovider/cdspackage.go
```

### 3.5、CCContext结构体

```go
type CCContext struct { //ChaincodeD上下文
	ChainID string
	Name string
	Version string
	TxID string
	Syscc bool
	SignedProposal *pb.SignedProposal
	Proposal *pb.Proposal
	canonicalName string
}

//构造CCContext
func NewCCContext(cid, name, version, txid string, syscc bool, signedProp *pb.SignedProposal, prop *pb.Proposal) *CCContext
//name + ":" + version
func (cccid *CCContext) GetCanonicalName() string
//代码在core/common/ccprovider/ccprovider.go
```

## 4、chaincode目录相关实现

### 4.1、ChaincodeProviderFactory接口定义及实现

#### 4.1.1、ChaincodeProviderFactory接口定义

```go
type ChaincodeProviderFactory interface {
	//构造ChaincodeProvider实例
	NewChaincodeProvider() ChaincodeProvider
}

func RegisterChaincodeProviderFactory(ccfact ChaincodeProviderFactory) {
	ccFactory = ccfact
}
func GetChaincodeProvider() ChaincodeProvider {
	return ccFactory.NewChaincodeProvider()
}
//代码在core/common/ccprovider/ccprovider.go
```

#### 4.1.2、ChaincodeProviderFactory接口实现

```go
type ccProviderFactory struct {
}

func (c *ccProviderFactory) NewChaincodeProvider() ccprovider.ChaincodeProvider {
	return &ccProviderImpl{}
}

func init() {
	ccprovider.RegisterChaincodeProviderFactory(&ccProviderFactory{})
}
//代码在core/chaincode/ccproviderimpl.go
```

### 4.2、ChaincodeProvider接口定义及实现

#### 4.2.1、ChaincodeProvider接口定义

```go
type ChaincodeProvider interface {
	GetContext(ledger ledger.PeerLedger) (context.Context, error)
	GetCCContext(cid, name, version, txid string, syscc bool, signedProp *pb.SignedProposal, prop *pb.Proposal) interface{}
	GetCCValidationInfoFromLSCC(ctxt context.Context, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal, chainID string, chaincodeID string) (string, []byte, error)
	ExecuteChaincode(ctxt context.Context, cccid interface{}, args [][]byte) (*pb.Response, *pb.ChaincodeEvent, error)
	Execute(ctxt context.Context, cccid interface{}, spec interface{}) (*pb.Response, *pb.ChaincodeEvent, error)
	ExecuteWithErrorFilter(ctxt context.Context, cccid interface{}, spec interface{}) ([]byte, *pb.ChaincodeEvent, error)
	Stop(ctxt context.Context, cccid interface{}, spec *pb.ChaincodeDeploymentSpec) error
	ReleaseContext()
}
//代码在core/common/ccprovider/ccprovider.go
```

#### 4.2.2、ChaincodeProvider接口实现

```go
type ccProviderImpl struct {
	txsim ledger.TxSimulator //交易模拟器
}

type ccProviderContextImpl struct {
	ctx *ccprovider.CCContext
}

//获取context.Context，添加TXSimulatorKey绑定c.txsim
func (c *ccProviderImpl) GetContext(ledger ledger.PeerLedger) (context.Context, error)
//构造CCContext，并构造ccProviderContextImpl
func (c *ccProviderImpl) GetCCContext(cid, name, version, txid string, syscc bool, signedProp *pb.SignedProposal, prop *pb.Proposal) interface{}
//调用GetChaincodeDataFromLSCC(ctxt, txid, signedProp, prop, chainID, chaincodeID)获取ChaincodeData中Vscc和Policy
func (c *ccProviderImpl) GetCCValidationInfoFromLSCC(ctxt context.Context, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal, chainID string, chaincodeID string) (string, []byte, error)
//调用ExecuteChaincode(ctxt, cccid.(*ccProviderContextImpl).ctx, args)执行上下文中指定的链码
func (c *ccProviderImpl) ExecuteChaincode(ctxt context.Context, cccid interface{}, args [][]byte) (*pb.Response, *pb.ChaincodeEvent, error)
//调用Execute(ctxt, cccid.(*ccProviderContextImpl).ctx, spec)
func (c *ccProviderImpl) Execute(ctxt context.Context, cccid interface{}, spec interface{}) (*pb.Response, *pb.ChaincodeEvent, error)
//调用ExecuteWithErrorFilter(ctxt, cccid.(*ccProviderContextImpl).ctx, spec)
func (c *ccProviderImpl) ExecuteWithErrorFilter(ctxt context.Context, cccid interface{}, spec interface{}) ([]byte, *pb.ChaincodeEvent, error)
//调用theChaincodeSupport.Stop(ctxt, cccid.(*ccProviderContextImpl).ctx, spec)
func (c *ccProviderImpl) Stop(ctxt context.Context, cccid interface{}, spec *pb.ChaincodeDeploymentSpec) error
//调用c.txsim.Done()
func (c *ccProviderImpl) ReleaseContext() {
//代码在core/chaincode/ccproviderimpl.go
```

### 4.3、ChaincodeSupport结构体

ChaincodeSupport更详细内容，参考：[Fabric 1.0源代码笔记 之 Chaincode（链码） #ChaincodeSupport（链码支持服务端）](ChaincodeSupport.md)

### 4.4、ExecuteChaincode函数（执行链码）

执行链码上下文中指定的链码。

```go
func ExecuteChaincode(ctxt context.Context, cccid *ccprovider.CCContext, args [][]byte) (*pb.Response, *pb.ChaincodeEvent, error) {
	var spec *pb.ChaincodeInvocationSpec
	var err error
	var res *pb.Response
	var ccevent *pb.ChaincodeEvent

	spec, err = createCIS(cccid.Name, args) //构造ChaincodeInvocationSpec
	res, ccevent, err = Execute(ctxt, cccid, spec)
	return res, ccevent, err
}
//代码在core/chaincode/chaincodeexec.go
```

res, ccevent, err = Execute(ctxt, cccid, spec)代码如下：

```go
func Execute(ctxt context.Context, cccid *ccprovider.CCContext, spec interface{}) (*pb.Response, *pb.ChaincodeEvent, error) {
	var err error
	var cds *pb.ChaincodeDeploymentSpec
	var ci *pb.ChaincodeInvocationSpec

	cctyp := pb.ChaincodeMessage_INIT //初始化
	if cds, _ = spec.(*pb.ChaincodeDeploymentSpec); cds == nil { //优先判断ChaincodeDeploymentSpec
		if ci, _ = spec.(*pb.ChaincodeInvocationSpec); ci == nil { //其次判断ChaincodeInvocationSpec
			panic("Execute should be called with deployment or invocation spec")
		}
		cctyp = pb.ChaincodeMessage_TRANSACTION //交易
	}

	_, cMsg, err := theChaincodeSupport.Launch(ctxt, cccid, spec)
	var ccMsg *pb.ChaincodeMessage
	ccMsg, err = createCCMessage(cctyp, cccid.TxID, cMsg)
	resp, err := theChaincodeSupport.Execute(ctxt, cccid, ccMsg, theChaincodeSupport.executetimeout)
	if resp.ChaincodeEvent != nil {
		resp.ChaincodeEvent.ChaincodeId = cccid.Name
		resp.ChaincodeEvent.TxId = cccid.TxID
	}
	if resp.Type == pb.ChaincodeMessage_COMPLETED {
		res := &pb.Response{}
		unmarshalErr := proto.Unmarshal(resp.Payload, res)
		return res, resp.ChaincodeEvent, nil
	}
}
//代码在core/chaincode
```

## 5、platforms（链码的编写语言平台）

platforms更详细内容，参考：[Fabric 1.0源代码笔记 之 Chaincode（链码） #platforms（链码语言平台）](platforms.md)