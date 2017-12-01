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

ChaincodeSupport更详细内容，参考：[Fabric 1.0源代码笔记 之 Chaincode（链码）（1）ChaincodeSupport（链码支持服务端）](ChaincodeSupport.md)

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

### 5.1、Platform接口定义

```go
type Platform interface {
	//验证ChaincodeSpec
	ValidateSpec(spec *pb.ChaincodeSpec) error
	//验证ChaincodeDeploymentSpec
	ValidateDeploymentSpec(spec *pb.ChaincodeDeploymentSpec) error
	//获取部署Payload
	GetDeploymentPayload(spec *pb.ChaincodeSpec) ([]byte, error)
	//生成Dockerfile
	GenerateDockerfile(spec *pb.ChaincodeDeploymentSpec) (string, error)
	//生成DockerBuild
	GenerateDockerBuild(spec *pb.ChaincodeDeploymentSpec, tw *tar.Writer) error
}
//代码在core/chaincode/platforms/platforms.go
```

### 5.2、golang语言平台实现

#### 5.2.1、golang.Platform结构体定义及方法

Platform接口golang语言平台实现，即golang.Platform结构体定义及方法。

```go
type Platform struct {
}
//验证ChaincodeSpec，即检查spec.ChaincodeId.Path是否存在
func (goPlatform *Platform) ValidateSpec(spec *pb.ChaincodeSpec) error
//验证ChaincodeDeploymentSpec，即检查cds.CodePackage（tar.gz文件）解压后文件合法性
func (goPlatform *Platform) ValidateDeploymentSpec(cds *pb.ChaincodeDeploymentSpec) error
//获取部署Payload，即将链码目录下文件及导入包所依赖的外部包目录下文件达成tar.gz包
func (goPlatform *Platform) GetDeploymentPayload(spec *pb.ChaincodeSpec) ([]byte, error)
func (goPlatform *Platform) GenerateDockerfile(cds *pb.ChaincodeDeploymentSpec) (string, error)
func (goPlatform *Platform) GenerateDockerBuild(cds *pb.ChaincodeDeploymentSpec, tw *tar.Writer) error

func pathExists(path string) (bool, error) //路径是否存在
func decodeUrl(spec *pb.ChaincodeSpec) (string, error) //url去掉http://或https://
func getGopath() (string, error) //获取GOPATH
func filter(vs []string, f func(string) bool) []string //按func(string) bool过滤[]string
func vendorDependencies(pkg string, files Sources) //重新映射依赖关系
//代码在core/chaincode/platforms/golang/platform.go
```

func (goPlatform *Platform) GetDeploymentPayload(spec *pb.ChaincodeSpec) ([]byte, error)代码如下：

```go
func (goPlatform *Platform) GetDeploymentPayload(spec *pb.ChaincodeSpec) ([]byte, error) {
	var err error
	code, err := getCode(spec) //获取代码，即构造CodeDescriptor，Gopath为代码真实路径，Pkg为代码相对路径
	env, err := getGoEnv()
	gopaths := splitEnvPaths(env["GOPATH"]) //GOPATH
	goroots := splitEnvPaths(env["GOROOT"]) //GOROOT，go安装路径
	gopaths[code.Gopath] = true //链码真实路径
	env["GOPATH"] = flattenEnvPaths(gopaths) //GOPATH、GOROOT、链码真实路径重新拼合为新GOPATH
	
	imports, err := listImports(env, code.Pkg) //获取导入包列表
	var provided = map[string]bool{ //如下两个包为ccenv已自带，可删除
		"github.com/hyperledger/fabric/core/chaincode/shim": true,
		"github.com/hyperledger/fabric/protos/peer":         true,
	}
	
	imports = filter(imports, func(pkg string) bool {
		if _, ok := provided[pkg]; ok == true { //从导入包中删除ccenv已自带的包
			return false
		}
		for goroot := range goroots { //删除goroot中自带的包
			fqp := filepath.Join(goroot, "src", pkg)
			exists, err := pathExists(fqp)
			if err == nil && exists {
				return false
			}
		}	
		return true
	})
	
	deps := make(map[string]bool)
	for _, pkg := range imports {
		transitives, err := listDeps(env, pkg) //列出所有导入包的依赖包
		deps[pkg] = true
		for _, dep := range transitives {
			deps[dep] = true
		}
	}
	delete(deps, "") //删除空
	
	fileMap, err := findSource(code.Gopath, code.Pkg) //遍历链码路径下文件
	for dep := range deps {
		for gopath := range gopaths {
			fqp := filepath.Join(gopath, "src", dep)
			exists, err := pathExists(fqp)
			if err == nil && exists {
				files, err := findSource(gopath, dep) //遍历依赖包下文件
				for _, file := range files {
					fileMap[file.Name] = file
				}
				
			}
		}
	}
	
	files := make(Sources, 0) //数组
	for _, file := range fileMap {
		files = append(files, file)
	}
	vendorDependencies(code.Pkg, files) //重新映射依赖关系
	sort.Sort(files)
	
	payload := bytes.NewBuffer(nil)
	gw := gzip.NewWriter(payload)
	tw := tar.NewWriter(gw)
	for _, file := range files {
		err = cutil.WriteFileToPackage(file.Path, file.Name, tw) //将文件写入压缩包中
	}
	tw.Close()
	gw.Close()
	return payload.Bytes(), nil
}
//代码在core/chaincode/platforms/golang/platform.go
```

#### 5.2.2、env相关函数

```go
type Env map[string]string
type Paths map[string]bool

func getEnv() Env //获取环境变量，写入map[string]string
func getGoEnv() (Env, error) //执行go env获取go环境变量，写入map[string]string
func flattenEnv(env Env) []string //拼合env，形式k=v，写入[]string
func splitEnvPaths(value string) Paths //分割多个路径字符串，linux下按:分割
func flattenEnvPaths(paths Paths) string //拼合多个路径字符串，以:分隔
//代码在core/chaincode/platforms/golang/env.go
```

#### 5.2.3、list相关函数

```go
//执行命令pgm，支持设置timeout，timeout后将kill进程
func runProgram(env Env, timeout time.Duration, pgm string, args ...string) ([]byte, error) 
//执行go list -f 规则 链码路径，获取导入包列表或依赖包列表
func list(env Env, template, pkg string) ([]string, error) 
//执行go list -f "{{ join .Deps \"\\n\"}}" 链码路径，获取依赖包列表
func listDeps(env Env, pkg string) ([]string, error) 
//执行go list -f "{{ join .Imports \"\\n\"}}" 链码路径，获取导入包列表
func listImports(env Env, pkg string) ([]string, error) 
//代码在core/chaincode/platforms/golang/list.go
```

#### 5.2.4、Sources类型及方法

```go
type Sources []SourceDescriptor
type SourceMap map[string]SourceDescriptor

type SourceDescriptor struct {
	Name, Path string
	Info       os.FileInfo
}

type CodeDescriptor struct {
	Gopath, Pkg string
	Cleanup     func()
}
//代码在core/chaincode/platforms/golang/package.go
```

涉及方法如下：

```go
//获取代码真实路径
func getCodeFromFS(path string) (codegopath string, err error) 
//获取代码，即构造CodeDescriptor，Gopath为代码真实路径，Pkg为代码相对路径
func getCode(spec *pb.ChaincodeSpec) (*CodeDescriptor, error)
//数组长度 
func (s Sources) Len() int
//交换数组i，j内容
func (s Sources) Swap(i, j int) 
//比较i，j的名称
func (s Sources) Less(i, j int) bool 
//遍历目录下文件，填充type SourceMap map[string]SourceDescriptor
func findSource(gopath, pkg string) (SourceMap, error) 
//代码在core/chaincode/platforms/golang/package.go
```