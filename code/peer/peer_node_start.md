# Fabric 1.0源代码笔记 之 Peer #peer node start命令实现

![](../peer/peer_node_start.png)

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

## 3、配置及创建PeerServer、创建EventHubServer（事件中心服务器）

```go
//初始化全局变量localAddress和peerEndpoint
err := peer.CacheConfiguration() 
peerEndpoint, err := peer.GetPeerEndpoint() //获取peerEndpoint
listenAddr := viper.GetString("peer.listenAddress") //PeerServer监听地址
secureConfig, err := peer.GetSecureConfig() //获取PeerServer安全配置，是否启用TLS、公钥、私钥、根证书
//以监听地址和安全配置，创建Peer GRPC Server
peerServer, err := peer.CreatePeerServer(listenAddr, secureConfig)
//创建EventHubServer（事件中心服务器）
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

pb.RegisterEventsServer(grpcServer.Server(), ehServer)代码如下：

```go
func RegisterEventsServer(s *grpc.Server, srv EventsServer) {
	s.RegisterService(&_Events_serviceDesc, srv) 
}
//代码在protos/peer/events.pb.go
```

events（事件服务）更详细内容，参考：[Fabric 1.0源代码笔记 之 events（事件服务）](../events/README.md)

## 4、创建并启动Chaincode Server，并注册系统链码

代码如下：

```go
ccprovider.EnableCCInfoCache() //ccInfoCacheEnabled = true
//创建Chaincode Server
//如果peer.chaincodeListenAddress，没有定义或者定义和peerListenAddress相同，均直接使用peerServer
//否则另行创建NewGRPCServer用于Chaincode Server
ccSrv, ccEpFunc := createChaincodeServer(peerServer, listenAddr)
//Chaincode service注册到grpcServer，并注册系统链码
registerChaincodeSupport(ccSrv.Server(), ccEpFunc)
go ccSrv.Start() //启动grpcServer
//代码在peer/node/start.go
```

ccSrv, ccEpFunc := createChaincodeServer(peerServer, listenAddr)代码如下：创建Chaincode Server。

```go
func createChaincodeServer(peerServer comm.GRPCServer, peerListenAddress string) (comm.GRPCServer, ccEndpointFunc) {
	//peer.chaincodeListenAddress，链码容器连接时的监听地址
	cclistenAddress := viper.GetString("peer.chaincodeListenAddress")

	var srv comm.GRPCServer
	var ccEpFunc ccEndpointFunc
	
	//如果peer.chaincodeListenAddress，没有定义或者定义和peerListenAddress相同，均直接使用peerServer
	//否则另行创建NewGRPCServer用于Chaincode Server
	if cclistenAddress == "" {
		ccEpFunc = peer.GetPeerEndpoint
		srv = peerServer
	} else if cclistenAddress == peerListenAddress {
		ccEpFunc = peer.GetPeerEndpoint
		srv = peerServer
	} else {
		config, err := peer.GetSecureConfig()
		srv, err = comm.NewGRPCServer(cclistenAddress, config)
		ccEpFunc = getChaincodeAddressEndpoint
	}

	return srv, ccEpFunc
}
//代码在peer/node/start.go
```

registerChaincodeSupport(ccSrv.Server(), ccEpFunc)代码如下：Chaincode service注册到grpcServer。

```go
func registerChaincodeSupport(grpcServer *grpc.Server, ccEpFunc ccEndpointFunc) {
	userRunsCC := chaincode.IsDevMode() //是否开发模式
	ccStartupTimeout := viper.GetDuration("chaincode.startuptimeout") //启动链码容器的超时
	if ccStartupTimeout < time.Duration(5)*time.Second { //至少5秒
		ccStartupTimeout = time.Duration(5) * time.Second
	} else {
	}
	//构造ChaincodeSupport
	ccSrv := chaincode.NewChaincodeSupport(ccEpFunc, userRunsCC, ccStartupTimeout)
	scc.RegisterSysCCs() //注册系统链码
	pb.RegisterChaincodeSupportServer(grpcServer, ccSrv) //service注册到grpcServer
}
//代码在peer/node/start.go
```

ccSrv := chaincode.NewChaincodeSupport(ccEpFunc, userRunsCC, ccStartupTimeout)代码如下：构造ChaincodeSupport。

```go
var theChaincodeSupport *ChaincodeSupport

func NewChaincodeSupport(getCCEndpoint func() (*pb.PeerEndpoint, error), userrunsCC bool, ccstartuptimeout time.Duration) *ChaincodeSupport {
	//即/var/hyperledger/production/chaincodes
	ccprovider.SetChaincodesPath(config.GetPath("peer.fileSystemPath") + string(filepath.Separator) + "chaincodes")

	pnid := viper.GetString("peer.networkId") //网络ID
	pid := viper.GetString("peer.id") //节点ID

	//构造ChaincodeSupport
	theChaincodeSupport = &ChaincodeSupport{runningChaincodes: &runningChaincodes{chaincodeMap: make(map[string]*chaincodeRTEnv), launchStarted: make(map[string]bool)}, peerNetworkID: 
pnid, peerID: pid}

	ccEndpoint, err := getCCEndpoint() //此处传入ccEpFunc
	if err != nil {
		theChaincodeSupport.peerAddress = viper.GetString("chaincode.peerAddress")
	} else {
		theChaincodeSupport.peerAddress = ccEndpoint.Address
	}
	if theChaincodeSupport.peerAddress == "" {
		theChaincodeSupport.peerAddress = peerAddressDefault
	}

	theChaincodeSupport.userRunsCC = userrunsCC //是否开发模式
	theChaincodeSupport.ccStartupTimeout = ccstartuptimeout //启动链码容器的超时

	theChaincodeSupport.peerTLS = viper.GetBool("peer.tls.enabled") //是否启用TLS
	if theChaincodeSupport.peerTLS {
		theChaincodeSupport.peerTLSCertFile = config.GetPath("peer.tls.cert.file")
		theChaincodeSupport.peerTLSKeyFile = config.GetPath("peer.tls.key.file")
		theChaincodeSupport.peerTLSSvrHostOrd = viper.GetString("peer.tls.serverhostoverride")
	}

	kadef := 0
	// Peer和链码之间的心跳超时，小于或等于0意味着关闭
	if ka := viper.GetString("chaincode.keepalive"); ka == "" {
		theChaincodeSupport.keepalive = time.Duration(kadef) * time.Second //0
	} else {
		t, terr := strconv.Atoi(ka)
		if terr != nil {
			t = kadef //0
		} else if t <= 0 {
			t = kadef //0
		}
		theChaincodeSupport.keepalive = time.Duration(t) * time.Second //非0
	}

	execto := time.Duration(30) * time.Second
	//invoke和initialize命令执行超时
	if eto := viper.GetDuration("chaincode.executetimeout"); eto <= time.Duration(1)*time.Second {
		//小于1秒时，默认30秒
	} else {
		execto = eto
	}
	theChaincodeSupport.executetimeout = execto

	viper.SetEnvPrefix("CORE")
	viper.AutomaticEnv()
	replacer := strings.NewReplacer(".", "_")
	viper.SetEnvKeyReplacer(replacer)

	theChaincodeSupport.chaincodeLogLevel = getLogLevelFromViper("level")
	theChaincodeSupport.shimLogLevel = getLogLevelFromViper("shim")
	theChaincodeSupport.logFormat = viper.GetString("chaincode.logging.format")

	return theChaincodeSupport
}

//代码在core/chaincode/chaincode_support.go
```

scc.RegisterSysCCs()代码如下：注册系统链码。

```go
func RegisterSysCCs() {
	//cscc、lscc、escc、vscc、qscc
	for _, sysCC := range systemChaincodes {
		RegisterSysCC(sysCC)
	}
}
代码在core/scc/importsysccs.go
```

## 5、注册Admin server和Endorser server

代码如下：

```go
//s.RegisterService(&_Admin_serviceDesc, srv)
//var _Admin_serviceDesc = grpc.ServiceDesc{...}
//core.NewAdminServer()构造ServerAdmin结构体，ServerAdmin结构体实现type AdminServer interface接口
pb.RegisterAdminServer(peerServer.Server(), core.NewAdminServer())
//构造结构体Endorser，Endorser结构体实现type EndorserServer interface接口
serverEndorser := endorser.NewEndorserServer()
//s.RegisterService(&_Endorser_serviceDesc, srv)
//var _Endorser_serviceDesc = grpc.ServiceDesc{...}
pb.RegisterEndorserServer(peerServer.Server(), serverEndorser)
//代码在peer/node/start.go
```

附type AdminServer interface接口定义：

```go
type AdminServer interface {
	GetStatus(context.Context, *google_protobuf.Empty) (*ServerStatus, error)
	StartServer(context.Context, *google_protobuf.Empty) (*ServerStatus, error)
	GetModuleLogLevel(context.Context, *LogLevelRequest) (*LogLevelResponse, error)
	SetModuleLogLevel(context.Context, *LogLevelRequest) (*LogLevelResponse, error)
	RevertLogLevels(context.Context, *google_protobuf.Empty) (*google_protobuf.Empty, error)
}
//代码在protos/peer/admin.pb.go
```

附type EndorserServer interface接口定义：

```go
type EndorserServer interface {
	ProcessProposal(context.Context, *SignedProposal) (*ProposalResponse, error)
}
//代码在protos/peer/peer.pb.go
```

## 6、初始化Gossip服务

```go
bootstrap := viper.GetStringSlice("peer.gossip.bootstrap") //启动节点后gossip连接的初始节点
serializedIdentity, err := mgmt.GetLocalSigningIdentityOrPanic().Serialize() //获取签名身份
messageCryptoService := peergossip.NewMCS( //构造mspMessageCryptoService（消息加密服务）
	peer.NewChannelPolicyManagerGetter(), //构造type channelPolicyManagerGetter struct{}
	localmsp.NewSigner(), //构造type mspSigner struct {}
	mgmt.NewDeserializersManager()) //构造type mspDeserializersManager struct{}
secAdv := peergossip.NewSecurityAdvisor(mgmt.NewDeserializersManager()) //构造mspSecurityAdvisor（安全顾问）

secureDialOpts := func() []grpc.DialOption {
	var dialOpts []grpc.DialOption
	dialOpts = append(dialOpts, grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(comm.MaxRecvMsgSize()), //MaxRecvMsgSize
		grpc.MaxCallSendMsgSize(comm.MaxSendMsgSize()))) //MaxSendMsgSize
	dialOpts = append(dialOpts, comm.ClientKeepaliveOptions()...) //ClientKeepaliveOptions
		
	if comm.TLSEnabled() {
		tlsCert := peerServer.ServerCertificate()
		dialOpts = append(dialOpts, grpc.WithTransportCredentials(comm.GetCASupport().GetPeerCredentials(tlsCert)))
	} else {
		dialOpts = append(dialOpts, grpc.WithInsecure())
	}
	return dialOpts
}

err = service.InitGossipService(serializedIdentity, peerEndpoint.Address, peerServer.Server(),
	messageCryptoService, secAdv, secureDialOpts, bootstrap...) //构造gossipServiceImpl
defer service.GetGossipService().Stop()
//代码在peer/node/start.go
```

Gossip更详细内容参考：[Fabric 1.0源代码笔记 之 gossip（流言算法）](../gossip/README.md)

## 7、初始化、部署并执行系统链码（scc）

代码如下：

```go
initSysCCs() //初始化系统链码，调用scc.DeploySysCCs("")
peer.Initialize(func(cid string) { //初始化所有链
	scc.DeploySysCCs(cid) //按chain id部署并运行系统链码
})
//代码在peer/node/start.go
```

scc更详细内容参考：[Fabric 1.0源代码笔记 之 scc（系统链码）](../scc/README.md)

## 8、启动peerServer和ehubGrpcServer，并监控系统信号，以及启动Go自带的profiling支持进行调试

代码如下：

```go
serve := make(chan error)
sigs := make(chan os.Signal, 1)
signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

go func() {
	sig := <-sigs //接收系统信号
	serve <- nil
}

go func() {
	var grpcErr error
	grpcErr = peerServer.Start() //启动peerServer
	serve <- grpcErr
}

err := writePid(config.GetPath("peer.fileSystemPath")+"/peer.pid", os.Getpid()) //写入pid

go ehubGrpcServer.Start() //启动ehubGrpcServer

if viper.GetBool("peer.profile.enabled") {
	go func() { //启动Go自带的profiling支持进行调试
		profileListenAddress := viper.GetString("peer.profile.listenAddress")
		profileErr := http.ListenAndServe(profileListenAddress, nil)
	}
}

return <-serve //等待serve
//代码在peer/node/start.go
```

## 9、按配置文件重新更新模块日志级别

```go
overrideLogModules := []string{"msp", "gossip", "ledger", "cauthdsl", "policies", "grpc"}
for _, module := range overrideLogModules {
	err = common.SetLogLevelFromViper(module)
}
flogging.SetPeerStartupModulesMap()
//代码在peer/node/start.go
```