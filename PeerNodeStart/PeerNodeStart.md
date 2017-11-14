# Fabric源码之旅(1)-Peer节点启动流程
如下内容从peer/main.go开始。
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
