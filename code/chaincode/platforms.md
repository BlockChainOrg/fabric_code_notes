# Fabric 1.0源代码笔记 之 Chaincode（链码） #platforms（链码语言平台）

## 1、platforms概述

platforms代码集中在core/chaincode/platforms目录下。

* core/chaincode/platforms目录，链码的编写语言平台实现，如golang或java。
	* platforms.go，Platform接口定义，及platforms相关工具函数。
	* util目录，Docker相关工具函数。
	* java目录，java语言平台实现。
	* golang目录，golang语言平台实现。
	
## 2、Platform接口定义

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

## 3、platforms相关工具函数

### 3.1、platforms相关工具函数

```go
//按链码类型构造Platform接口实例，如golang.Platform{}
func Find(chaincodeType pb.ChaincodeSpec_Type) (Platform, error)
//调取platform.GetDeploymentPayload(spec)，获取部署Payload
func GetDeploymentPayload(spec *pb.ChaincodeSpec) ([]byte, error)
//优先获取tls根证书，如无则获取tls证书
func getPeerTLSCert() ([]byte, error)
//调取platform.GenerateDockerfile(cds)，创建Dockerfile
func generateDockerfile(platform Platform, cds *pb.ChaincodeDeploymentSpec, tls bool) ([]byte, error)
//调取platform.GenerateDockerBuild(cds, tw)，创建DockerBuild
func generateDockerBuild(platform Platform, cds *pb.ChaincodeDeploymentSpec, inputFiles InputFiles, tw *tar.Writer) error
//调取generateDockerfile(platform, cds, cert != nil)
func GenerateDockerBuild(cds *pb.ChaincodeDeploymentSpec) (io.Reader, error)
//代码在core/chaincode/platforms/platforms.go
```

### 3.2、Docker相关工具函数

```go
//contents+hash合并后再哈希
func ComputeHash(contents []byte, hash []byte) []byte 
//哈希目录下文件并打包
func HashFilesInDir(rootDir string, dir string, hash []byte, tw *tar.Writer) ([]byte, error) 
//目录是否存在
func IsCodeExist(tmppath string) error 
//编译链码
func DockerBuild(opts DockerBuildOptions) error 
//代码在core/chaincode/platforms/util/utils.go
```

func DockerBuild(opts DockerBuildOptions) error代码如下：

```go
type DockerBuildOptions struct {
	Image        string
	Env          []string
	Cmd          string
	InputStream  io.Reader
	OutputStream io.Writer
}

func DockerBuild(opts DockerBuildOptions) error {
	client, err := cutil.NewDockerClient()
	if err != nil {
		return fmt.Errorf("Error creating docker client: %s", err)
	}
	if opts.Image == "" {
		//通用的本地编译环境
		opts.Image = cutil.GetDockerfileFromConfig("chaincode.builder")
	}

	//确认镜像是否存在或从远程拉取
	_, err = client.InspectImage(opts.Image)
	if err != nil {
		err = client.PullImage(docker.PullImageOptions{Repository: opts.Image}, docker.AuthConfiguration{})
	}

	//创建一个暂时的容器
	container, err := client.CreateContainer(docker.CreateContainerOptions{
		Config: &docker.Config{
			Image:        opts.Image,
			Env:          opts.Env,
			Cmd:          []string{"/bin/sh", "-c", opts.Cmd},
			AttachStdout: true,
			AttachStderr: true,
		},
	})
	//删除容器
	defer client.RemoveContainer(docker.RemoveContainerOptions{ID: container.ID})

	//上传输入
	err = client.UploadToContainer(container.ID, docker.UploadToContainerOptions{
		Path:        "/chaincode/input",
		InputStream: opts.InputStream,
	})

	stdout := bytes.NewBuffer(nil)
	_, err = client.AttachToContainerNonBlocking(docker.AttachToContainerOptions{
		Container:    container.ID,
		OutputStream: stdout,
		ErrorStream:  stdout,
		Logs:         true,
		Stdout:       true,
		Stderr:       true,
		Stream:       true,
	})

	//启动容器
	err = client.StartContainer(container.ID, nil)
	//等待容器返回
	retval, err := client.WaitContainer(container.ID)
	//获取容器输出
	err = client.DownloadFromContainer(container.ID, docker.DownloadFromContainerOptions{
		Path:         "/chaincode/output/.",
		OutputStream: opts.OutputStream,
	})
	return nil
}
//代码在core/chaincode/platforms/util/utils.go
```

## 4、golang语言平台实现

### 4.1、golang.Platform结构体定义及方法

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
func decodeUrl(spec *pb.ChaincodeSpec) (string, error) //spec.ChaincodeId.Path去掉http://或https://
func getGopath() (string, error) //获取GOPATH
func filter(vs []string, f func(string) bool) []string //按func(string) bool过滤[]string
func vendorDependencies(pkg string, files Sources) //重新映射依赖关系
//代码在core/chaincode/platforms/golang/platform.go
```

#### 4.1.1 func (goPlatform *Platform) GetDeploymentPayload(spec *pb.ChaincodeSpec) ([]byte, error)

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

#### 4.1.2、func (goPlatform *Platform) GenerateDockerfile(cds *pb.ChaincodeDeploymentSpec) (string, error)

```go
func (goPlatform *Platform) GenerateDockerfile(cds *pb.ChaincodeDeploymentSpec) (string, error) {
	var buf []string
	//go语言链码部署依赖的基础镜像
	buf = append(buf, "FROM "+cutil.GetDockerfileFromConfig("chaincode.golang.runtime"))
	//binpackage.tar添加到/usr/local/bin目录下
	buf = append(buf, "ADD binpackage.tar /usr/local/bin")
	dockerFileContents := strings.Join(buf, "\n")
	return dockerFileContents, nil
}
//代码在core/chaincode/platforms/golang/platform.go
```

#### 4.1.3、func (goPlatform *Platform) GenerateDockerBuild(cds *pb.ChaincodeDeploymentSpec, tw *tar.Writer) error

```go
func (goPlatform *Platform) GenerateDockerBuild(cds *pb.ChaincodeDeploymentSpec, tw *tar.Writer) error {
	spec := cds.ChaincodeSpec
	pkgname, err := decodeUrl(spec)
	const ldflags = "-linkmode external -extldflags '-static'"

	codepackage := bytes.NewReader(cds.CodePackage)
	binpackage := bytes.NewBuffer(nil)
	//编译链码
	err = util.DockerBuild(util.DockerBuildOptions{
		Cmd:          fmt.Sprintf("GOPATH=/chaincode/input:$GOPATH go build -ldflags \"%s\" -o /chaincode/output/chaincode %s", ldflags, pkgname),
		InputStream:  codepackage,
		OutputStream: binpackage,
	})
	return cutil.WriteBytesToPackage("binpackage.tar", binpackage.Bytes(), tw)
}
//代码在core/chaincode/platforms/golang/platform.go
```

### 4.2、env相关函数

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

### 4.3、list相关函数

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

### 4.4、Sources类型及方法

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

