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
