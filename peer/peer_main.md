# Fabric 1.0源代码笔记 之 Peer（1）peer根命令入口及加载子命令

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

## 2、加载命令行工具和命令

Fabric支持类似peer node start、peer channel create、peer chaincode install这种命令、子命令、命令选项的命令行形式。
此功能由第三方包cobra来实现，以peer chaincode install -n test_cc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02为例，
其中peer、chaincode、install、-n分别为命令、子命令、子命令的子命令、命令选项。

如下代码为mainCmd的初始化，其中Use为命令名称，PersistentPreRunE先于Run执行用于初始化日志系统，Run此处用于打印版本信息或帮助信息。cobra使用方法参考：https://github.com/spf13/cobra。

```go
var mainCmd = &cobra.Command{
	Use: "peer",
	PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
		loggingSpec := viper.GetString("logging_level")

		if loggingSpec == "" {
			loggingSpec = viper.GetString("logging.peer")
		}
		flogging.InitFromSpec(loggingSpec) //初始化flogging日志系统

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
	logger = logging.MustGetLogger(pkgLogID) //创建仅在flogging包内代码使用的logging.Logger对象
	Reset() //全局变量初始化为默认值
	initgrpclogger() //初始化gRPC Logger，即创建logging.Logger对象，并用这个对象设置grpclog
}
//代码在common/flogging/logging.go
```

init()执行结束后，peer/main.go中调用flogging.InitFromSpec(loggingSpec)，将再次初始化全局日志级别为loggingSpec，之前默认为logging.INFO。

func InitFromSpec(spec string) string代码如下。
其中spec格式为：[<module>[,<module>...]=]<level>[:[<module>[,<module>...]=]<level>...]。
此处传入spec为""，将""模块日志级别设置为defaultLevel，并会将modules初始化为defaultLevel。

```go
levelAll := defaultLevel //defaultLevel为logging.INFO
var err error

if spec != "" { //如果spec不为空，则按既定格式读取
	fields := strings.Split(spec, ":") //按:分割
	for _, field := range fields {
		split := strings.Split(field, "=") //按=分割
		switch len(split) {
		case 1: //只有level
			if levelAll, err = logging.LogLevel(field); err != nil { //levelAll赋值为logging.LogLevel枚举中定义的Level级别
				levelAll = defaultLevel // 如果没有定义，则使用默认日志级别
			}
		case 2: //针对module,module...=level，split[0]为模块集，split[1]为要设置的日志级别
			levelSingle, err := logging.LogLevel(split[1]) //levelSingle赋值为logging.LogLevel枚举中定义的Level级别
			modules := strings.Split(split[0], ",") //按,分割获取模块名
			for _, module := range modules {
				logging.SetLevel(levelSingle, module) //本条规则中所有模块日志级别均设置为levelSingle
			}
		default:
			//...
		}
	}
}
//代码在common/flogging/logging.go
```

flogging（Fabric日志系统）更详细信息参考：[Fabric 1.0源代码笔记 之 flogging（Fabric日志系统）](../flogging/README.md)

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
factory.InitFactories(bccspConfig)及BCCSP（区块链加密服务提供者）更详细内容，参考：[Fabric 1.0源代码笔记 之 BCCSP（区块链加密服务提供者）](../bccsp/README.md)

MSP（成员关系服务提供者）更详细内容，参考：[Fabric 1.0源代码笔记 之 MSP（成员关系服务提供者）](../msp/README.md)

至此，peer/main.go结束，接下来将进入peer/node/start.go中serve(args)函数。