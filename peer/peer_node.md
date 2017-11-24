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

## 2、初始化Ledger（账本）

初始化账本，即如下一条代码：

```go
ledgermgmt.Initialize()
//代码在peer/node/start.go
```

ledgermgmt.Initialize()代码展开如下：

```go
func initialize() {
	openedLedgers = make(map[string]ledger.PeerLedger) //openedLedgers为全局变量，存储目前使用的账本列表
	provider, err := kvledger.NewProvider() //创建账本Provider实例
	ledgerProvider = provider
}
//代码在core/ledger/ledgermgmt/ledger_mgmt.go
```

kvledger.NewProvider()代码如下：

```go
func NewProvider() (ledger.PeerLedgerProvider, error) {
	idStore := openIDStore(ledgerconfig.GetLedgerProviderPath()) //打开idStore

	//创建并初始化blkstorage
	attrsToIndex := []blkstorage.IndexableAttr{
		blkstorage.IndexableAttrBlockHash,
		blkstorage.IndexableAttrBlockNum,
		blkstorage.IndexableAttrTxID,
		blkstorage.IndexableAttrBlockNumTranNum,
		blkstorage.IndexableAttrBlockTxID,
		blkstorage.IndexableAttrTxValidationCode,
	}
	indexConfig := &blkstorage.IndexConfig{AttrsToIndex: attrsToIndex}
	blockStoreProvider := fsblkstorage.NewProvider(
		fsblkstorage.NewConf(ledgerconfig.GetBlockStorePath(), ledgerconfig.GetMaxBlockfileSize()),
		indexConfig)

	//创建并初始化statedb
	var vdbProvider statedb.VersionedDBProvider
	if !ledgerconfig.IsCouchDBEnabled() {
		vdbProvider = stateleveldb.NewVersionedDBProvider()
	} else {
		vdbProvider, err = statecouchdb.NewVersionedDBProvider()
	}

	//创建并初始化historydb
	var historydbProvider historydb.HistoryDBProvider
	historydbProvider = historyleveldb.NewHistoryDBProvider()
	//构造Provider
	provider := &Provider{idStore, blockStoreProvider, vdbProvider, historydbProvider}
	provider.recoverUnderConstructionLedger()
	return provider, nil
}
//代码在core/ledger/kvledger/kv_ledger_provider.go
```

Ledger更详细内容，参考：[Fabric 1.0源代码笔记 之 Ledger（账本）](../ledger/README.md)

### 3、配置及启动PeerServer、启动EventHubServer（事件中心服务器）

```go
//初始化全局变量localAddress和peerEndpoint
err := peer.CacheConfiguration() 
peerEndpoint, err := peer.GetPeerEndpoint() //获取peerEndpoint
listenAddr := viper.GetString("peer.listenAddress") //PeerServer监听地址
secureConfig, err := peer.GetSecureConfig() //获取PeerServer安全配置，是否启用TLS、公钥、私钥、根证书
//以监听地址和安全配置，启动Peer GRPC Server
peerServer, err := peer.CreatePeerServer(listenAddr, secureConfig)
//启动EventHubServer（事件中心服务器）
ehubGrpcServer, err := createEventHubServer(secureConfig)
//代码在peer/node/start.go
```

func createEventHubServer(secureConfig comm.SecureServerConfig) (comm.GRPCServer, error)代码如下：

```go
var lis net.Listener
var err error
lis, err = net.Listen("tcp", viper.GetString("peer.events.address")) //创建Listen
grpcServer, err := comm.NewGRPCServerFromListener(lis, secureConfig) //从Listen创建GRPCServer
ehServer := producer.NewEventsServer(
	uint(viper.GetInt("peer.events.buffersize")), //最大进行缓冲的消息数
	viper.GetDuration("peer.events.timeout")) //缓冲已满的情况下，往缓冲中发送消息的超时
pb.RegisterEventsServer(grpcServer.Server(), ehServer) //EventHubServer注册至grpcServer
return grpcServer, nil
//代码在peer/node/start.go
```
