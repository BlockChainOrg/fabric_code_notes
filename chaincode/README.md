# Fabric 1.0源代码笔记 之 Chaincode（链码）

## 1、Chaincode概述

Chaincode，即链码或智能合约，代码分布在protos/peer目录和core/chaincode目录，目录结构如下：

* protos/peer目录：
	* chaincode.pb.go，ChaincodeDeploymentSpec结构体定义。
* core/chaincode目录：
	* platforms目录，链码的编写语言平台实现，如golang或java。
		* platforms.go，Platform接口定义，及部分工具函数。
		* java目录，java语言平台实现。
		* golang目录，golang语言平台实现。

## 2、ChaincodeDeploymentSpec结构体定义（用于Chaincode部署）

### 2.1 ChaincodeDeploymentSpec结构体定义

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

### 2.2、ChaincodeSpec结构体定义

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

## 3、platforms（链码的编写语言平台）

### 3.1、Platform接口定义

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

### 3.2、golang语言平台实现

### 3.2.1、golang.Platform结构体定义及方法

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


### 3.2.2、env相关函数

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

### 3.2.3、list相关函数

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

### 3.2.4、Sources类型及方法

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