# Fabric 1.0源代码笔记 之 Peer（2）peer node命令实现

## 1、peer node加载子命令start和status

peer node加载子命令start和status，代码如下：

```go
func Cmd() *cobra.Command {
	nodeCmd.AddCommand(startCmd()) //加载子命令start
	nodeCmd.AddCommand(statusCmd()) //加载子命令status
	return nodeCmd
}

var nodeCmd = &cobra.Command{
	Use:   nodeFuncName,
	Short: fmt.Sprint(shortDes),
	Long:  fmt.Sprint(longDes),
}
//代码在peer/node/node.go
```

startCmd()代码如下：
其中serve(args)为peer node start的实现代码，比较复杂，本文将重点讲解。
另statusCmd()代码与startCmd()相近，暂略。

```go
func startCmd() *cobra.Command {
	flags := nodeStartCmd.Flags()
	flags.BoolVarP(&chaincodeDevMode, "peer-chaincodedev", "", false, "Whether peer in chaincode development mode")
	flags.BoolVarP(&peerDefaultChain, "peer-defaultchain", "", false, "Whether to start peer with chain testchainid")
	flags.StringVarP(&orderingEndpoint, "orderer", "o", "orderer:7050", "Ordering service endpoint") //orderer
	return nodeStartCmd
}

var nodeStartCmd = &cobra.Command{
	Use:   "start",
	Short: "Starts the node.",
	Long:  `Starts a node that interacts with the network.`,
	RunE: func(cmd *cobra.Command, args []string) error {
		return serve(args) //serve(args)为peer node start的实现代码
	},
}
//代码在peer/node/start.go
```

**注：如下内容均为serve(args)的代码，即peer node start命令执行流程。**

## 2、初始化账本管理




