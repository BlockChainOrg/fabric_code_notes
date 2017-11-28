# Fabric 1.0源代码笔记 之 Orderer（1）orderer start命令实现

## 1、加载命令行工具并解析命令行参数

orderer的命令行工具，基于gopkg.in/alecthomas/kingpin.v2实现，地址：http://gopkg.in/alecthomas/kingpin.v2。
相关代码如下

```go
var (
	//创建kingpin.Application
	app = kingpin.New("orderer", "Hyperledger Fabric orderer node")
	//创建子命令start和version
	start   = app.Command("start", "Start the orderer node").Default()
	version = app.Command("version", "Show version information")
)

kingpin.Version("0.0.1")
//解析命令行参数
switch kingpin.MustParse(app.Parse(os.Args[1:])) {
case start.FullCommand():
	//orderer start的命令实现，下文展开讲解
case version.FullCommand():
	//输出版本信息
	fmt.Println(metadata.GetVersionInfo())
}
代码在orderer/main.go
```

metadata.GetVersionInfo()代码如下：

```go
func GetVersionInfo() string {
	Version = common.Version //var Version string，全局变量
	if Version == "" {
		Version = "development build"
	}

	return fmt.Sprintf("%s:\n Version: %s\n Go version: %s\n OS/Arch: %s",
		ProgramName, Version, runtime.Version(),
		fmt.Sprintf("%s/%s", runtime.GOOS, runtime.GOARCH))
}
//代码在orderer/metadata/metadata.go
```

## 2、加载配置文件

配置文件的加载，基于viper实现，即https://github.com/spf13/viper。

```go
conf := config.Load()
代码在orderer/main.go
```

conf := config.Load()代码如下：

```go
func Load() *TopLevel {
	config := viper.New()
	//cf.InitViper作用为加载配置文件路径及设置配置文件名称
	cf.InitViper(config, configName) //configName = strings.ToLower(Prefix)，其中Prefix = "ORDERER"

	config.SetEnvPrefix(Prefix) //Prefix = "ORDERER"
	config.AutomaticEnv()
	replacer := strings.NewReplacer(".", "_")
	config.SetEnvKeyReplacer(replacer)

	err := config.ReadInConfig() //加载配置文件内容

	var uconf TopLevel
	//将配置文件内容输出到结构体中
	err = viperutil.EnhancedExactUnmarshal(config, &uconf)
	//完成初始化，即检查空项，并赋默认值
	uconf.completeInitialization(filepath.Dir(config.ConfigFileUsed()))

	return &uconf
}

//代码在orderer/localconfig/config.go
```

TopLevel结构体及本地配置更详细内容，参考：[Fabric 1.0源代码笔记 之 Orderer（2）localconfig（Orderer配置文件定义）](localconfig.md)