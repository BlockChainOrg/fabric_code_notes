# Fabric 1.0源码旅程 之 Peer

## 1、加载环境变量配置和配置文件

Fabric支持通过环境变量对部分配置进行更新，如：CORE_LOGGING_LEVEL为输出的日志级别、CORE_PEER_ID为Peer的ID等。
此部分功能由第三方包viper来实现，viper除支持环境变量的配置方式外，还支持配置文件方式。viper使用方法参考：https://github.com/spf13/viper。
如下代码为加载环境变量配置，其中cmdRoot为"core"，即CORE_开头的环境变量。

```go
viper.SetEnvPrefix(cmdRoot)
viper.AutomaticEnv()
replacer := strings.NewReplacer(".", "_")
viper.SetEnvKeyReplacer(replacer)
//代码在peer/main.go
```

加载配置文件，同样由第三方包viper来实现，具体代码如下：
其中cmdRoot为"core"，即/etc/hyperledger/fabric/core.yaml。

```go
err := common.InitConfig(cmdRoot) 
//代码在peer/main.go
```

如下代码为common.InitConfig(cmdRoot)的具体实现：

```go
config.InitViper(nil, cmdRoot)
err := viper.ReadInConfig()
//代码在peer/common/common.go
```

另附config.InitViper(nil, cmdRoot)的代码实现：
优先从环境变量FABRIC_CFG_PATH中获取配置文件路径，其次为当前目录、开发环境目录（即：src/github.com/hyperledger/fabric/sampleconfig）、和OfficialPath（即：/etc/hyperledger/fabric）。
AddDevConfigPath是对addConfigPath的封装，目的是通过GetDevConfigDir()调取sampleconfig路径。

```go
var altPath = os.Getenv("FABRIC_CFG_PATH")
if altPath != "" {
	addConfigPath(v, altPath)
} else {
	addConfigPath(v, "./")
	err := AddDevConfigPath(v)
	addConfigPath(v, OfficialPath)
}
viper.SetConfigName(configName)
//代码在core/config/config.go
```

## 2、加载命令行工具和根命令

Fabric支持类似peer node start、peer channel create、peer chaincode install这种命令、子命令、命令选项的命令行形式。
此功能由第三方包cobra来实现，以peer chaincode install -n test_cc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02为例，
其中peer、chaincode、install、-n分别为命令、子命令、子命令的子命令、命令选项。

如下代码为mainCmd的初始化，其中Use为命令名称，PersistentPreRunE先于Run执行用于初始化日志系统，Run此处用于打印版本信息或帮助信息。cobra使用方法参考：https://github.com/spf13/cobra。
初始化日志系统代码flogging.InitFromSpec(loggingSpec)下文另行分析。

```go
var mainCmd = &cobra.Command{
	Use: "peer",
	PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
		loggingSpec := viper.GetString("logging_level")

		if loggingSpec == "" {
			loggingSpec = viper.GetString("logging.peer")
		}
		flogging.InitFromSpec(loggingSpec)

		return nil
	},
	Run: func(cmd *cobra.Command, args []string) {
		if versionFlag {
			fmt.Print(version.GetInfo())
		} else {
			cmd.HelpFunc()(cmd, args)
		}
	},
}
//代码在peer/main.go
```

如下代码为添加命令行选项，-v, --version、--logging-level和--test.coverprofile分别用于版本信息、日志级别和测试覆盖率分析。

```go
mainFlags := mainCmd.PersistentFlags()
mainFlags.BoolVarP(&versionFlag, "version", "v", false, "Display current version of fabric peer server")
mainFlags.String("logging-level", "", "Default logging level and overrides, see core.yaml for full syntax")
viper.BindPFlag("logging_level", mainFlags.Lookup("logging-level"))
testCoverProfile := ""
mainFlags.StringVarP(&testCoverProfile, "test.coverprofile", "", "coverage.cov", "Done")
//代码在peer/main.go
```

如下代码为逐一加载peer命令下子命令：node、channel、chaincode、clilogging、version。

```go
mainCmd.AddCommand(version.Cmd())
mainCmd.AddCommand(node.Cmd())
mainCmd.AddCommand(chaincode.Cmd(nil))
mainCmd.AddCommand(clilogging.Cmd(nil))
mainCmd.AddCommand(channel.Cmd(nil))
//代码在peer/main.go　
```

mainCmd.Execute()为命令启动。

如下为mainCmd.AddCommand(node.Cmd()) peer node相关命令的加载（与上述代码相似）：

```go
func Cmd() *cobra.Command {
	nodeCmd.AddCommand(startCmd()) //peer node start
	nodeCmd.AddCommand(statusCmd()) //peer node status

	return nodeCmd
}

var nodeCmd = &cobra.Command{
	Use:   nodeFuncName,
	Short: fmt.Sprint(shortDes),
	Long:  fmt.Sprint(longDes),
}
//代码在peer/node/node.go　
```

以及nodeCmd.AddCommand(startCmd()) peer node start相关命令的加载：

```go
func startCmd() *cobra.Command {
	flags := nodeStartCmd.Flags()
	flags.BoolVarP(&chaincodeDevMode, "peer-chaincodedev", "", false, //chaincodedev
		"Whether peer in chaincode development mode")
	flags.BoolVarP(&peerDefaultChain, "peer-defaultchain", "", false, //defaultchain
		"Whether to start peer with chain testchainid")
	flags.StringVarP(&orderingEndpoint, "orderer", "o", "orderer:7050", "Ordering service endpoint") //orderer

	return nodeStartCmd
}

var nodeStartCmd = &cobra.Command{
	Use:   "start",
	Short: "Starts the node.",
	Long:  `Starts a node that interacts with the network.`,
	RunE: func(cmd *cobra.Command, args []string) error {
		return serve(args) //peer node start实际代码
	},
}
//代码在peer/node/start.go　
```

## 3、初始化日志系统（输出对象、日志格式、日志级别）

如下为初始日志系统代码入口，其中loggingSpec取自环境变量CORE_LOGGING_LEVEL或配置文件中logging.peer，即：全局的默认日志级别。

```go
flogging.InitFromSpec(loggingSpec)
//代码在peer/main.go
```

flogging，即：fabric logging，为Fabric基于第三方包go-logging封装的日志包，go-logging使用方法参考：https://github.com/op/go-logging
如下代码为flogging包的初始化函数：

```go
func init() {
	logger = logging.MustGetLogger(pkgLogID) //pkgLogID为"flogging"，创建一个名称为flogging的日志对象
	Reset() //初始化flogging部分全局变量，创建并初始化日志输出对象输出格式和日志级别
	initgrpclogger() //初始化gRPC日志对象
}
//代码在common/flogging/logging.go
```

如下代码为Reset()的具体实现：

```go
modules = make(map[string]string) //创建日志对象名称和日志级别的映射
lock = sync.RWMutex{} //读写锁
defaultOutput = os.Stderr //标准错误输出
InitBackend(SetFormat(defaultFormat), defaultOutput) //初始化日志输出对象
InitFromSpec("") //初始化日志级别
//代码在common/flogging/logging.go
```

如下代码为InitBackend(SetFormat(defaultFormat), defaultOutput)的具体实现，创建日志输出对象，设置日志格式和日志级别。

```go
backend := logging.NewLogBackend(output, "", 0)
//defaultFormat = "%{color}%{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}"
backendFormatter := logging.NewBackendFormatter(backend, formatter)
logging.SetBackend(backendFormatter).SetLevel(defaultLevel, "") //defaultLevel  = logging.INFO
//代码在common/flogging/logging.go
```

如下代码为InitFromSpec("")的具体实现。
另外InitFromSpec()函数支持以[<module>[,<module>...]=]<level>[:[<module>[,<module>...]=]<level>...]字符串方式对各个模块设置日志级别，但此处因传入字符串为空，因此不涉及此功能。
MustGetLogger()函数中有代码modules[module] = GetModuleLevel(module)，目的为将模块的日志级别写入modules map中。

```go
levelAll := defaultLevel //defaultLevel  = logging.INFO
logging.SetLevel(levelAll, "")
MustGetLogger(pkgLogID) //pkgLogID为"flogging"
//代码在common/flogging/logging.go
```

如下代码为initgrpclogger()的具体实现，其中MustGetLogger是对logging.MustGetLogger(module)的封装，同时将module的日志级别写入modules map中。
grpclog为gRPC第三方包grpc-go中的日志包，grpc-go默认使用go语言标准日志接口，此处可使得grpc-go也可以使用flogging包的实现。grpc-go：https://github.com/grpc/grpc-go
有关gRPC内容下文会有涉及。

```go
type grpclogger struct {
	logger *logging.Logger
}
glogger := MustGetLogger(GRPCModuleID)
grpclog.SetLogger(&grpclogger{glogger})
//代码在common/flogging/grpclogger.go
```

init()执行结束后，peer/main.go中调用flogging.InitFromSpec(loggingSpec)，将再次初始化全局日志级别为loggingSpec，之前默认为logging.INFO。

如下代码为从配置文件中获取logging.format日志格式，并重新初始化日志输出对象。

```go
flogging.InitBackend(flogging.SetFormat(viper.GetString("logging.format")), logOutput)
//代码在peer/main.go
```

## 4、初始化 MSP （Membership Service Provider会员服务提供者）

如下代码为初始化MSP，获取peer.mspConfigPath路径和peer.localMspId，分别表示MSP的本地路径（/etc/hyperledger/fabric/msp/）和Peer所关联的MSP ID，并初始化组织和身份信息。

```go
var mspMgrConfigDir = config.GetPath("peer.mspConfigPath")
var mspID = viper.GetString("peer.localMspId")
err = common.InitCrypto(mspMgrConfigDir, mspID)
//代码在peer/main.go
```

/etc/hyperledger/fabric/msp/目录下包括：admincerts、cacerts、keystore、signcerts、tlscacerts。其中：

* admincerts：为管理员证书的PEM文件，如Admin@org1.example.com-cert.pem。
* cacerts：为根CA证书的PEM文件，如ca.org1.example.com-cert.pem。
* keystore：为具有节点的签名密钥的PEM文件，如91e54fccbb82b29d07657f6df9587c966edee6366786d234bbb8c96707ec7c16_sk。
* signcerts：为节点X.509证书的PEM文件，如peer1.org1.example.com-cert.pem。
* tlscacerts：为TLS根CA证书的PEM文件，如tlsca.org1.example.com-cert.pem。

如下代码为common.InitCrypto(mspMgrConfigDir, mspID)的具体实现，peer.BCCSP为密码库相关配置，包括算法和文件路径等，格式如下：

```go
BCCSP:
	Default: SW
	SW:
		Hash: SHA2
		Security: 256
		FileKeyStore:
			KeyStore:
			
var bccspConfig *factory.FactoryOpts
err = viperutil.EnhancedExactUnmarshalKey("peer.BCCSP", &bccspConfig) //将peer.BCCSP配置信息加载至bccspConfig中
err = mspmgmt.LoadLocalMsp(mspMgrConfigDir, bccspConfig, localMSPID) //从指定目录中加载本地MSP
//代码在peer/common/common.go
```

factory.FactoryOpts定义为：

```go
type FactoryOpts struct {
	ProviderName string  `mapstructure:"default" json:"default" yaml:"Default"`
	SwOpts       *SwOpts `mapstructure:"SW,omitempty" json:"SW,omitempty" yaml:"SwOpts"`
}
//FactoryOpts代码在bccsp/factory/nopkcs11.go，本目录下另有代码文件pkcs11.go，在-tags "nopkcs11"条件下二选一编译。
```

```go
type SwOpts struct {
	// Default algorithms when not specified (Deprecated?)
	SecLevel   int    `mapstructure:"security" json:"security" yaml:"Security"`
	HashFamily string `mapstructure:"hash" json:"hash" yaml:"Hash"`

	// Keystore Options
	Ephemeral     bool               `mapstructure:"tempkeys,omitempty" json:"tempkeys,omitempty"`
	FileKeystore  *FileKeystoreOpts  `mapstructure:"filekeystore,omitempty" json:"filekeystore,omitempty" yaml:"FileKeyStore"`
	DummyKeystore *DummyKeystoreOpts `mapstructure:"dummykeystore,omitempty" json:"dummykeystore,omitempty"`
}
type FileKeystoreOpts struct {
	KeyStorePath string `mapstructure:"keystore" yaml:"KeyStore"`
}
//SwOpts和FileKeystoreOpts代码均在bccsp/factory/swfactory.go
```

如下代码为viperutil.EnhancedExactUnmarshalKey("peer.BCCSP", &bccspConfig)的具体实现，getKeysRecursively为递归读取peer.BCCSP配置信息。
mapstructure为第三方包：github.com/mitchellh/mapstructure，用于将map[string]interface{}转换为struct。
示例代码：https://godoc.org/github.com/mitchellh/mapstructure#example-Decode--WeaklyTypedInput

```go
func EnhancedExactUnmarshalKey(baseKey string, output interface{}) error {
	m := make(map[string]interface{})
	m[baseKey] = nil
	leafKeys := getKeysRecursively("", viper.Get, m)

	config := &mapstructure.DecoderConfig{
		Metadata:         nil,
		Result:           output,
		WeaklyTypedInput: true,
	}
	decoder, err := mapstructure.NewDecoder(config)
	return decoder.Decode(leafKeys[baseKey])
}
//代码在common/viperutil/config_util.go
```

如下代码为mspmgmt.LoadLocalMsp(mspMgrConfigDir, bccspConfig, localMSPID)的具体实现，从指定目录中加载本地MSP。

```go
conf, err := msp.GetLocalMspConfig(dir, bccspConfig, mspID) //获取本地MSP配置，序列化后写入msp.MSPConfig，即conf
return GetLocalMSP().Setup(conf) //调取msp.NewBccspMsp()创建bccspmsp实例，调取bccspmsp.Setup(conf)解码conf.Config并设置bccspmsp
//代码在msp/mgmt/mgmt.go
```

如下代码为msp.GetLocalMspConfig(dir, bccspConfig, mspID)的具体实现。
SetupBCCSPKeystoreConfig()核心代码为bccspConfig.SwOpts.FileKeystore = &factory.FileKeystoreOpts{KeyStorePath: keystoreDir}，目的是在FileKeystore或KeyStorePath为空时设置默认值。

```go
signcertDir := filepath.Join(dir, signcerts) //signcerts为"signcerts"，signcertDir即/etc/hyperledger/fabric/msp/signcerts/
keystoreDir := filepath.Join(dir, keystore) //keystore为"keystore"，keystoreDir即/etc/hyperledger/fabric/msp/keystore/
bccspConfig = SetupBCCSPKeystoreConfig(bccspConfig, keystoreDir) //设置bccspConfig.SwOpts.Ephemeral = false和bccspConfig.SwOpts.FileKeystore = &factory.FileKeystoreOpts{KeyStorePath: keystoreDir}
	//bccspConfig.SwOpts.Ephemeral是否短暂的
err := factory.InitFactories(bccspConfig) //初始化bccsp factory，并创建bccsp实例
signcert, err := getPemMaterialFromDir(signcertDir) //读取X.509证书的PEM文件
sigid := &msp.SigningIdentityInfo{PublicSigner: signcert[0], PrivateSigner: nil} //构造SigningIdentityInfo
return getMspConfig(dir, ID, sigid) //分别读取cacerts、admincerts、tlscacerts文件，以及config.yaml中组织信息，构造msp.FabricMSPConfig，序列化后用于构造msp.MSPConfig
//代码在msp/configbuilder.go
```
factory.InitFactories(bccspConfig)及bccsp后续实现，参考：[Fabric 1.0源码旅程 之 BCCSP（区块链加密服务提供者）](../bccsp/bccsp.md)

MSP相关深入内容，参考：[Fabric 1.0源码旅程 之 MSP（成员关系服务提供者）](../msp/msp.md)

至此，peer/main.go结束，接下来将进入peer/node/start.go中serve(args)函数。

## 5、初始化账本管理



## 10、本文使用到的网络内容

* [fabric源码解析2——peer命令结构](http://blog.csdn.net/idsuf698987/article/details/75034998)
* [fabric源码解析3——日志系统](http://blog.csdn.net/idsuf698987/article/details/75223986)
* [fabric源码解析4——配置系统](http://blog.csdn.net/idsuf698987/article/details/75224228)
* [fabric源码解析12——peer的MSP服务](http://blog.csdn.net/idsuf698987/article/details/77103011)
* [fabric源码解析13——peer的BCCSP服务](http://blog.csdn.net/idsuf698987/article/details/77200287)