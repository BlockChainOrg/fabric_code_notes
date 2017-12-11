# Fabric 1.0源代码笔记 之 gossip（流言算法）

## 1、gossip概述

gossip，翻译为流言蜚语，即为一种可最终达到一致的算法。最终一致的另外的含义就是，不保证同时达到一致。
gossip中有三种基本的操作：
* push - A节点将数据(key,value,version)及对应的版本号推送给B节点，B节点更新A中比自己新的数据
* pull - A仅将数据key,version推送给B，B将本地比A新的数据（Key,value,version）推送给A，A更新本地
* push/pull - 与pull类似，只是多了一步，A再将本地比B新的数据推送给B，B更新本地

gossip在Fabric中作用：
* 管理组织内节点和通道信息，并加测节点是否在线或离线。
* 广播数据，使组织内相同channel的节点同步相同数据。
* 管理新加入的节点，并同步数据到新节点。

gossip代码，分布在gossip、peer/gossip目录下，目录结构如下：

* gossip目录：
	* service目录，GossipService接口定义及实现。
	* integration目录，NewGossipComponent工具函数。
	* gossip目录，Gossip接口定义及实现。
	* comm目录，GossipServer接口实现。
	* state目录，GossipStateProvider接口定义及实现（状态复制）。
	* api目录：消息加密服务接口定义。
		* crypto.go，MessageCryptoService接口定义。
		* channel.go，SecurityAdvisor接口定义。
* peer/gossip目录：
	* mcs.go，MessageCryptoService接口实现，即mspMessageCryptoService结构体及方法。
	* sa.go，SecurityAdvisor接口实现，即mspSecurityAdvisor结构体及方法。
	
GossipServer更详细内容，参考：[Fabric 1.0源代码笔记 之 gossip（流言算法） #GossipServer（Gossip服务端）](GossipServer.md)

## 2、GossipService接口定义及实现

### 2.1、GossipService接口定义

```go
type GossipService interface {
	gossip.Gossip
	NewConfigEventer() ConfigProcessor
	InitializeChannel(chainID string, committer committer.Committer, endpoints []string)
	GetBlock(chainID string, index uint64) *common.Block
	AddPayload(chainID string, payload *proto.Payload) error
}
//代码在gossip/service/gossip_service.go
```

补充gossip.Gossip：

```go
type Gossip interface {
	Send(msg *proto.GossipMessage, peers ...*comm.RemotePeer)
	Peers() []discovery.NetworkMember
	PeersOfChannel(common.ChainID) []discovery.NetworkMember
	UpdateMetadata(metadata []byte)
	UpdateChannelMetadata(metadata []byte, chainID common.ChainID)
	Gossip(msg *proto.GossipMessage)
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)
	JoinChan(joinMsg api.JoinChannelMessage, chainID common.ChainID)
	SuspectPeers(s api.PeerSuspector)
	Stop()
}
//代码在gossip/gossip/gossip.go
```

### 2.2、GossipService接口实现

GossipService接口实现，即gossipServiceImpl结构体及方法。

```go
type gossipSvc gossip.Gossip

type gossipServiceImpl struct {
	gossipSvc
	chains          map[string]state.GossipStateProvider //链
	leaderElection  map[string]election.LeaderElectionService //选举服务
	deliveryService deliverclient.DeliverService
	deliveryFactory DeliveryServiceFactory
	lock            sync.RWMutex
	idMapper        identity.Mapper
	mcs             api.MessageCryptoService
	peerIdentity    []byte
	secAdv          api.SecurityAdvisor
}

//初始化Gossip Service，调取InitGossipServiceCustomDeliveryFactory()
func InitGossipService(peerIdentity []byte, endpoint string, s *grpc.Server, mcs api.MessageCryptoService,secAdv api.SecurityAdvisor, secureDialOpts api.PeerSecureDialOpts, bootPeers ...string) error
//初始化Gossip Service
func InitGossipServiceCustomDeliveryFactory(peerIdentity []byte, endpoint string, s *grpc.Server,factory DeliveryServiceFactory, mcs api.MessageCryptoService, secAdv api.SecurityAdvisor,secureDialOpts api.PeerSecureDialOpts, bootPeers ...string) error
//获取gossipServiceInstance
func GetGossipService() GossipService 
//调取 newConfigEventer(g)
func (g *gossipServiceImpl) NewConfigEventer() ConfigProcessor 
//初始化通道
func (g *gossipServiceImpl) InitializeChannel(chainID string, committer committer.Committer, endpoints []string) 
func (g *gossipServiceImpl) configUpdated(config Config) 
func (g *gossipServiceImpl) GetBlock(chainID string, index uint64) *common.Block 
func (g *gossipServiceImpl) AddPayload(chainID string, payload *proto.Payload) error 
//停止gossip服务
func (g *gossipServiceImpl) Stop() 
func (g *gossipServiceImpl) newLeaderElectionComponent(chainID string, callback func(bool)) election.LeaderElectionService 
func (g *gossipServiceImpl) amIinChannel(myOrg string, config Config) bool 
func (g *gossipServiceImpl) onStatusChangeFactory(chainID string, committer blocksprovider.LedgerInfo) func(bool) 
func orgListFromConfig(config Config) []string 
//代码在gossip/service/gossip_service.go
```

#### 2.2.1、func InitGossipService(peerIdentity []byte, endpoint string, s *grpc.Server, mcs api.MessageCryptoService,secAdv api.SecurityAdvisor, secureDialOpts api.PeerSecureDialOpts, bootPeers ...string) error

```go
func InitGossipService(peerIdentity []byte, endpoint string, s *grpc.Server, mcs api.MessageCryptoService,
	secAdv api.SecurityAdvisor, secureDialOpts api.PeerSecureDialOpts, bootPeers ...string) error {
	util.GetLogger(util.LoggingElectionModule, "")
	return InitGossipServiceCustomDeliveryFactory(peerIdentity, endpoint, s, &deliveryFactoryImpl{},
		mcs, secAdv, secureDialOpts, bootPeers...)
}

func InitGossipServiceCustomDeliveryFactory(peerIdentity []byte, endpoint string, s *grpc.Server,
	factory DeliveryServiceFactory, mcs api.MessageCryptoService, secAdv api.SecurityAdvisor,
	secureDialOpts api.PeerSecureDialOpts, bootPeers ...string) error {
	var err error
	var gossip gossip.Gossip
	once.Do(func() {
		//peer.gossip.endpoint，本节点在组织内的gossip id，或者使用peerEndpoint.Address
		if overrideEndpoint := viper.GetString("peer.gossip.endpoint"); overrideEndpoint != "" {
			endpoint = overrideEndpoint
		}
		idMapper := identity.NewIdentityMapper(mcs, peerIdentity)
		gossip, err = integration.NewGossipComponent(peerIdentity, endpoint, s, secAdv,
			mcs, idMapper, secureDialOpts, bootPeers...)
		gossipServiceInstance = &gossipServiceImpl{
			mcs:             mcs,
			gossipSvc:       gossip,
			chains:          make(map[string]state.GossipStateProvider),
			leaderElection:  make(map[string]election.LeaderElectionService),
			deliveryFactory: factory,
			idMapper:        idMapper,
			peerIdentity:    peerIdentity,
			secAdv:          secAdv,
		}
	})
	return err
}

//代码在gossip/service/gossip_service.go
```

#### 2.2.2、func (g *gossipServiceImpl) Stop()

```go
func (g *gossipServiceImpl) Stop() {
	g.lock.Lock()
	defer g.lock.Unlock()
	for _, ch := range g.chains {
		logger.Info("Stopping chain", ch)
		ch.Stop()
	}

	for chainID, electionService := range g.leaderElection {
		logger.Infof("Stopping leader election for %s", chainID)
		electionService.Stop()
	}
	g.gossipSvc.Stop()
	if g.deliveryService != nil {
		g.deliveryService.Stop()
	}
}
//代码在gossip/service/gossip_service.go
```

#### 2.2.3、func (g *gossipServiceImpl) InitializeChannel(chainID string, committer committer.Committer, endpoints []string) 

```go
func (g *gossipServiceImpl) InitializeChannel(chainID string, committer committer.Committer, endpoints []string) {
	g.lock.Lock()
	defer g.lock.Unlock()

	//构造GossipStateProviderImpl，并启动goroutine处理从orderer或其他节点接收的block
	g.chains[chainID] = state.NewGossipStateProvider(chainID, g, committer, g.mcs)
	if g.deliveryService == nil {
		var err error
		g.deliveryService, err = g.deliveryFactory.Service(gossipServiceInstance, endpoints, g.mcs)
	}
	if g.deliveryService != nil {
		//如下两者只可以二选一
		leaderElection := viper.GetBool("peer.gossip.useLeaderElection") //是否动态选举代表节点
		isStaticOrgLeader := viper.GetBool("peer.gossip.orgLeader") //是否指定本节点为组织代表节点

		if leaderElection {
			//选举代表节点
			g.leaderElection[chainID] = g.newLeaderElectionComponent(chainID, g.onStatusChangeFactory(chainID, committer))
		} else if isStaticOrgLeader {
			//如果指定本节点为代表节点，则启动Deliver client
			g.deliveryService.StartDeliverForChannel(chainID, committer, func() {})
		}
}

//代码在gossip/service/gossip_service.go
```

#### 2.2.4、func (g *gossipServiceImpl) AddPayload(chainID string, payload *proto.Payload) error 

```go
func (g *gossipServiceImpl) AddPayload(chainID string, payload *proto.Payload) error {
	g.lock.RLock()
	defer g.lock.RUnlock()
	return g.chains[chainID].AddPayload(payload)
}
//代码在gossip/service/gossip_service.go
```

## 3、NewGossipComponent工具函数

```go
func NewGossipComponent(peerIdentity []byte, endpoint string, s *grpc.Server,
	secAdv api.SecurityAdvisor, cryptSvc api.MessageCryptoService, idMapper identity.Mapper,
	secureDialOpts api.PeerSecureDialOpts, bootPeers ...string) (gossip.Gossip, error) {

	//peer.gossip.externalEndpoint为节点被组织外节点感知时的地址
	externalEndpoint := viper.GetString("peer.gossip.externalEndpoint")

	//endpoint，取自peer.address，即节点对外服务的地址
	//bootPeers取自peer.gossip.bootstrap，即启动节点后所进行gossip连接的初始节点，可为多个
	conf, err := newConfig(endpoint, externalEndpoint, bootPeers...)
	gossipInstance := gossip.NewGossipService(conf, s, secAdv, cryptSvc, idMapper,
		peerIdentity, secureDialOpts)

	return gossipInstance, nil
}

func newConfig(selfEndpoint string, externalEndpoint string, bootPeers ...string) (*gossip.Config, error) {
	_, p, err := net.SplitHostPort(selfEndpoint)
	port, err := strconv.ParseInt(p, 10, 64) //节点对外服务的端口

	var cert *tls.Certificate
	if viper.GetBool("peer.tls.enabled") {
		certTmp, err := tls.LoadX509KeyPair(config.GetPath("peer.tls.cert.file"), config.GetPath("peer.tls.key.file"))
		cert = &certTmp
	}

	return &gossip.Config{
		BindPort:                   int(port),
		BootstrapPeers:             bootPeers,
		ID:                         selfEndpoint,
		MaxBlockCountToStore:       util.GetIntOrDefault("peer.gossip.maxBlockCountToStore", 100),
		MaxPropagationBurstLatency: util.GetDurationOrDefault("peer.gossip.maxPropagationBurstLatency", 10*time.Millisecond),
		MaxPropagationBurstSize:    util.GetIntOrDefault("peer.gossip.maxPropagationBurstSize", 10),
		PropagateIterations:        util.GetIntOrDefault("peer.gossip.propagateIterations", 1),
		PropagatePeerNum:           util.GetIntOrDefault("peer.gossip.propagatePeerNum", 3),
		PullInterval:               util.GetDurationOrDefault("peer.gossip.pullInterval", 4*time.Second),
		PullPeerNum:                util.GetIntOrDefault("peer.gossip.pullPeerNum", 3),
		InternalEndpoint:           selfEndpoint,
		ExternalEndpoint:           externalEndpoint,
		PublishCertPeriod:          util.GetDurationOrDefault("peer.gossip.publishCertPeriod", 10*time.Second),
		RequestStateInfoInterval:   util.GetDurationOrDefault("peer.gossip.requestStateInfoInterval", 4*time.Second),
		PublishStateInfoInterval:   util.GetDurationOrDefault("peer.gossip.publishStateInfoInterval", 4*time.Second),
		SkipBlockVerification:      viper.GetBool("peer.gossip.skipBlockVerification"),
		TLSServerCert:              cert,
	}, nil
}
//代码在gossip/integration/integration.go
```

## 4、Gossip接口定义及实现

### 4.1、Gossip接口定义

```go
type Gossip interface {
	//向节点发送消息
	Send(msg *proto.GossipMessage, peers ...*comm.RemotePeer)
	//返回活的节点
	Peers() []discovery.NetworkMember
	//返回指定通道的活的节点
	PeersOfChannel(common.ChainID) []discovery.NetworkMember
	//更新metadata
	UpdateMetadata(metadata []byte)
	//更新通道metadata
	UpdateChannelMetadata(metadata []byte, chainID common.ChainID)
	//向网络内其他节点发送数据
	Gossip(msg *proto.GossipMessage)
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)
	//加入通道
	JoinChan(joinMsg api.JoinChannelMessage, chainID common.ChainID)
	//验证可疑节点身份，并关闭无效链接
	SuspectPeers(s api.PeerSuspector)
	//停止Gossip服务
	Stop()
}
//代码在gossip/gossip/gossip.go
```

### 4.2、Config结构体定义

```go
type Config struct {
	BindPort            int      //绑定的端口，仅用于测试
	ID                  string   //本实例id，即对外开放的节点地址
	BootstrapPeers      []string //启动时要链接的节点
	PropagateIterations int      //取自peer.gossip.propagateIterations，消息转发的次数，默认为1
	PropagatePeerNum    int      //取自peer.gossip.propagatePeerNum，消息推送给节点的个数，默认为3
	MaxBlockCountToStore int //取自peer.gossip.maxBlockCountToStore，保存到内存中的区块个数上限，默认100
	MaxPropagationBurstSize    int           //取自peer.gossip.maxPropagationBurstSize，保存的最大消息个数，超过则转发给其他节点，默认为10
	MaxPropagationBurstLatency time.Duration //取自peer.gossip.maxPropagationBurstLatency，保存消息的最大时间，超过则转发给其他节点，默认为10毫秒

	PullInterval time.Duration //取自peer.gossip.pullInterval，拉取消息的时间间隔，默认为4秒
	PullPeerNum  int           //取自peer.gossip.pullPeerNum，从指定个数的节点拉取信息，默认3个

	SkipBlockVerification bool //取自peer.gossip.skipBlockVerification，是否不对区块消息进行校验，默认false即需校验
	PublishCertPeriod        time.Duration    //取自peer.gossip.publishCertPeriod，包括在消息中的启动证书的时间，默认10s
	PublishStateInfoInterval time.Duration    //取自peer.gossip.publishStateInfoInterval，向其他节点推送状态信息的时间间隔，默认4s
	RequestStateInfoInterval time.Duration    //取自peer.gossip.requestStateInfoInterval，从其他节点拉取状态信息的时间间隔，默认4s
	TLSServerCert            *tls.Certificate //本节点TLS 证书

	InternalEndpoint string //本节点在组织内使用的地址
	ExternalEndpoint string //本节点对外部组织公布的地址
}
//代码在gossip/gossip/gossip.go
```

### 4.3、Gossip接口实现

Gossip接口接口实现，即gossipServiceImpl结构体及方法。

#### 4.3.1、gossipServiceImpl结构体及方法

```go
type gossipServiceImpl struct {
	selfIdentity          api.PeerIdentityType
	includeIdentityPeriod time.Time
	certStore             *certStore
	idMapper              identity.Mapper
	presumedDead          chan common.PKIidType
	disc                  discovery.Discovery
	comm                  comm.Comm
	incTime               time.Time
	selfOrg               api.OrgIdentityType
	*comm.ChannelDeMultiplexer
	logger            *logging.Logger
	stopSignal        *sync.WaitGroup
	conf              *Config
	toDieChan         chan struct{}
	stopFlag          int32
	emitter           batchingEmitter
	discAdapter       *discoveryAdapter
	secAdvisor        api.SecurityAdvisor
	chanState         *channelState
	disSecAdap        *discoverySecurityAdapter
	mcs               api.MessageCryptoService
	stateInfoMsgStore msgstore.MessageStore
}

func NewGossipService(conf *Config, s *grpc.Server, secAdvisor api.SecurityAdvisor,
func NewGossipServiceWithServer(conf *Config, secAdvisor api.SecurityAdvisor, mcs api.MessageCryptoService,
func (g *gossipServiceImpl) JoinChan(joinMsg api.JoinChannelMessage, chainID common.ChainID)
func (g *gossipServiceImpl) SuspectPeers(isSuspected api.PeerSuspector)
func (g *gossipServiceImpl) Gossip(msg *proto.GossipMessage)
func (g *gossipServiceImpl) Send(msg *proto.GossipMessage, peers ...*comm.RemotePeer)
func (g *gossipServiceImpl) Peers() []discovery.NetworkMember
func (g *gossipServiceImpl) PeersOfChannel(channel common.ChainID) []discovery.NetworkMember
func (g *gossipServiceImpl) Stop()
func (g *gossipServiceImpl) UpdateMetadata(md []byte)
func (g *gossipServiceImpl) UpdateChannelMetadata(md []byte, chainID common.ChainID)
func (g *gossipServiceImpl) Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)

func (g *gossipServiceImpl) newStateInfoMsgStore() msgstore.MessageStore
func (g *gossipServiceImpl) selfNetworkMember() discovery.NetworkMember
func newChannelState(g *gossipServiceImpl) *channelState
func createCommWithoutServer(s *grpc.Server, cert *tls.Certificate, idStore identity.Mapper,
func createCommWithServer(port int, idStore identity.Mapper, identity api.PeerIdentityType,
func (g *gossipServiceImpl) toDie() bool
func (g *gossipServiceImpl) periodicalIdentityValidationAndExpiration()
func (g *gossipServiceImpl) periodicalIdentityValidation(suspectFunc api.PeerSuspector, interval time.Duration)
func (g *gossipServiceImpl) learnAnchorPeers(orgOfAnchorPeers api.OrgIdentityType, anchorPeers []api.AnchorPeer)
func (g *gossipServiceImpl) handlePresumedDead()
func (g *gossipServiceImpl) syncDiscovery()
func (g *gossipServiceImpl) start()
func (g *gossipServiceImpl) acceptMessages(incMsgs <-chan proto.ReceivedMessage)
func (g *gossipServiceImpl) handleMessage(m proto.ReceivedMessage)
func (g *gossipServiceImpl) forwardDiscoveryMsg(msg proto.ReceivedMessage)
func (g *gossipServiceImpl) validateMsg(msg proto.ReceivedMessage) bool
func (g *gossipServiceImpl) sendGossipBatch(a []interface{})
func (g *gossipServiceImpl) gossipBatch(msgs []*proto.SignedGossipMessage)
func (g *gossipServiceImpl) sendAndFilterSecrets(msg *proto.SignedGossipMessage, peers ...*comm.RemotePeer)
func (g *gossipServiceImpl) gossipInChan(messages []*proto.SignedGossipMessage, chanRoutingFactory channelRoutingFilterFactory)
func selectOnlyDiscoveryMessages(m interface{}) bool
func (g *gossipServiceImpl) newDiscoverySecurityAdapter() *discoverySecurityAdapter
func (sa *discoverySecurityAdapter) ValidateAliveMsg(m *proto.SignedGossipMessage) bool
func (sa *discoverySecurityAdapter) SignMessage(m *proto.GossipMessage, internalEndpoint string) *proto.Envelope
func (sa *discoverySecurityAdapter) validateAliveMsgSignature(m *proto.SignedGossipMessage, identity api.PeerIdentityType) bool
func (g *gossipServiceImpl) createCertStorePuller() pull.Mediator
func (g *gossipServiceImpl) sameOrgOrOurOrgPullFilter(msg proto.ReceivedMessage) func(string) bool
func (g *gossipServiceImpl) connect2BootstrapPeers()
func (g *gossipServiceImpl) createStateInfoMsg(metadata []byte, chainID common.ChainID) (*proto.SignedGossipMessage, error)
func (g *gossipServiceImpl) hasExternalEndpoint(PKIID common.PKIidType) bool
func (g *gossipServiceImpl) isInMyorg(member discovery.NetworkMember) bool
func (g *gossipServiceImpl) getOrgOfPeer(PKIID common.PKIidType) api.OrgIdentityType
func (g *gossipServiceImpl) validateLeadershipMessage(msg *proto.SignedGossipMessage) error
func (g *gossipServiceImpl) validateStateInfoMsg(msg *proto.SignedGossipMessage) error
func (g *gossipServiceImpl) disclosurePolicy(remotePeer *discovery.NetworkMember) (discovery.Sieve, discovery.EnvelopeFilter)
func (g *gossipServiceImpl) peersByOriginOrgPolicy(peer discovery.NetworkMember) filter.RoutingFilter
func partitionMessages(pred common.MessageAcceptor, a []*proto.SignedGossipMessage) ([]*proto.SignedGossipMessage, []*proto.SignedGossipMessage)
func extractChannels(a []*proto.SignedGossipMessage) []common.ChainID

func (g *gossipServiceImpl) newDiscoveryAdapter() *discoveryAdapter
func (da *discoveryAdapter) close()
func (da *discoveryAdapter) toDie() bool
func (da *discoveryAdapter) Gossip(msg *proto.SignedGossipMessage)
func (da *discoveryAdapter) SendToPeer(peer *discovery.NetworkMember, msg *proto.SignedGossipMessage)
func (da *discoveryAdapter) Ping(peer *discovery.NetworkMember) bool
func (da *discoveryAdapter) Accept() <-chan *proto.SignedGossipMessage
func (da *discoveryAdapter) PresumedDead() <-chan common.PKIidType
func (da *discoveryAdapter) CloseConn(peer *discovery.NetworkMember)
//代码在gossip/gossip/gossip_impl.go
```

#### 4.3.2、func NewGossipService(conf *Config, s *grpc.Server, secAdvisor api.SecurityAdvisor,mcs api.MessageCryptoService, idMapper identity.Mapper, selfIdentity api.PeerIdentityType,secureDialOpts api.PeerSecureDialOpts) Gossip

创建附加到grpc.Server上的Gossip实例。

```go
func NewGossipService(conf *Config, s *grpc.Server, secAdvisor api.SecurityAdvisor,
	mcs api.MessageCryptoService, idMapper identity.Mapper, selfIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts) Gossip {

	var c comm.Comm
	var err error

	lgr := util.GetLogger(util.LoggingGossipModule, conf.ID)
	if s == nil { 创建并启动gRPC Server，以及注册GossipServer实例
		c, err = createCommWithServer(conf.BindPort, idMapper, selfIdentity, secureDialOpts)
	} else { //将GossipServer实例注册至peerServer
		c, err = createCommWithoutServer(s, conf.TLSServerCert, idMapper, selfIdentity, secureDialOpts)
	}

	g := &gossipServiceImpl{
		selfOrg:               secAdvisor.OrgByPeerIdentity(selfIdentity),
		secAdvisor:            secAdvisor,
		selfIdentity:          selfIdentity,
		presumedDead:          make(chan common.PKIidType, presumedDeadChanSize),
		idMapper:              idMapper,
		disc:                  nil,
		mcs:                   mcs,
		comm:                  c,
		conf:                  conf,
		ChannelDeMultiplexer:  comm.NewChannelDemultiplexer(),
		logger:                lgr,
		toDieChan:             make(chan struct{}, 1),
		stopFlag:              int32(0),
		stopSignal:            &sync.WaitGroup{},
		includeIdentityPeriod: time.Now().Add(conf.PublishCertPeriod),
	}
	g.stateInfoMsgStore = g.newStateInfoMsgStore()
	g.chanState = newChannelState(g)
	g.emitter = newBatchingEmitter(conf.PropagateIterations,
		conf.MaxPropagationBurstSize, conf.MaxPropagationBurstLatency,
		g.sendGossipBatch)
	g.discAdapter = g.newDiscoveryAdapter()
	g.disSecAdap = g.newDiscoverySecurityAdapter()
	g.disc = discovery.NewDiscoveryService(g.selfNetworkMember(), g.discAdapter, g.disSecAdap, g.disclosurePolicy)
	g.certStore = newCertStore(g.createCertStorePuller(), idMapper, selfIdentity, mcs)

	go g.start()
	go g.periodicalIdentityValidationAndExpiration()
	go g.connect2BootstrapPeers()
	return g
}
//代码在gossip/gossip/gossip_impl.go
```

#### 4.3.3、go g.start()

```go
func (g *gossipServiceImpl) start() {
	go g.syncDiscovery()
	go g.handlePresumedDead()

	msgSelector := func(msg interface{}) bool {
		gMsg, isGossipMsg := msg.(proto.ReceivedMessage)
		if !isGossipMsg {
			return false
		}
		isConn := gMsg.GetGossipMessage().GetConn() != nil
		isEmpty := gMsg.GetGossipMessage().GetEmpty() != nil
		return !(isConn || isEmpty)
	}

	incMsgs := g.comm.Accept(msgSelector)
	go g.acceptMessages(incMsgs)
}
//代码在gossip/gossip/gossip_impl.go
```

go g.acceptMessages(incMsgs)代码如下：

```go
func (g *gossipServiceImpl) acceptMessages(incMsgs <-chan proto.ReceivedMessage) {
	defer g.logger.Debug("Exiting")
	g.stopSignal.Add(1)
	defer g.stopSignal.Done()
	for {
		select {
		case s := <-g.toDieChan:
			g.toDieChan <- s
			return
		case msg := <-incMsgs:
			g.handleMessage(msg) //此处会调取gc.HandleMessage(m)，处理转发消息
		}
	}
}
//代码在gossip/gossip/gossip_impl.go
```

gc.HandleMessage(m)代码如下：

```go
func (gc *gossipChannel) HandleMessage(msg proto.ReceivedMessage) {
	m := msg.GetGossipMessage()
	orgID := gc.GetOrgOfPeer(msg.GetConnectionInfo().ID)
	if m.IsDataMsg() || m.IsStateInfoMsg() {
		added := false

		if m.IsDataMsg() {
			added = gc.blockMsgStore.Add(msg.GetGossipMessage())
		} else {
			added = gc.stateInfoMsgStore.Add(msg.GetGossipMessage())
		}

		if added {
			gc.Gossip(msg.GetGossipMessage()) //转发给其他节点，最终会调用func (g *gossipServiceImpl) gossipBatch(msgs []*proto.SignedGossipMessage)实现批量分发
			gc.DeMultiplex(m)
			if m.IsDataMsg() {
				gc.blocksPuller.Add(msg.GetGossipMessage())
			}
		}
		return
	}
	//...
}
//代码在gossip/gossip/channel/channel.go
```

func (g *gossipServiceImpl) gossipBatch(msgs []*proto.SignedGossipMessage)代码如下：

```go
func (g *gossipServiceImpl) gossipBatch(msgs []*proto.SignedGossipMessage) {
	var blocks []*proto.SignedGossipMessage
	var stateInfoMsgs []*proto.SignedGossipMessage
	var orgMsgs []*proto.SignedGossipMessage
	var leadershipMsgs []*proto.SignedGossipMessage

	isABlock := func(o interface{}) bool {
		return o.(*proto.SignedGossipMessage).IsDataMsg()
	}
	isAStateInfoMsg := func(o interface{}) bool {
		return o.(*proto.SignedGossipMessage).IsStateInfoMsg()
	}
	isOrgRestricted := func(o interface{}) bool {
		return aliveMsgsWithNoEndpointAndInOurOrg(o) || o.(*proto.SignedGossipMessage).IsOrgRestricted()
	}
	isLeadershipMsg := func(o interface{}) bool {
		return o.(*proto.SignedGossipMessage).IsLeadershipMsg()
	}

	// Gossip blocks
	blocks, msgs = partitionMessages(isABlock, msgs)
	g.gossipInChan(blocks, func(gc channel.GossipChannel) filter.RoutingFilter {
		return filter.CombineRoutingFilters(gc.EligibleForChannel, gc.IsMemberInChan, g.isInMyorg)
	})

	// Gossip Leadership messages
	leadershipMsgs, msgs = partitionMessages(isLeadershipMsg, msgs)
	g.gossipInChan(leadershipMsgs, func(gc channel.GossipChannel) filter.RoutingFilter {
		return filter.CombineRoutingFilters(gc.EligibleForChannel, gc.IsMemberInChan, g.isInMyorg)
	})

	// Gossip StateInfo messages
	stateInfoMsgs, msgs = partitionMessages(isAStateInfoMsg, msgs)
	for _, stateInfMsg := range stateInfoMsgs {
		peerSelector := g.isInMyorg
		gc := g.chanState.lookupChannelForGossipMsg(stateInfMsg.GossipMessage)
		if gc != nil && g.hasExternalEndpoint(stateInfMsg.GossipMessage.GetStateInfo().PkiId) {
			peerSelector = gc.IsMemberInChan
		}

		peers2Send := filter.SelectPeers(g.conf.PropagatePeerNum, g.disc.GetMembership(), peerSelector)
		g.comm.Send(stateInfMsg, peers2Send...)
	}

	// Gossip messages restricted to our org
	orgMsgs, msgs = partitionMessages(isOrgRestricted, msgs)
	peers2Send := filter.SelectPeers(g.conf.PropagatePeerNum, g.disc.GetMembership(), g.isInMyorg)
	for _, msg := range orgMsgs {
		g.comm.Send(msg, peers2Send...)
	}

	// Finally, gossip the remaining messages
	for _, msg := range msgs {
		selectByOriginOrg := g.peersByOriginOrgPolicy(discovery.NetworkMember{PKIid: msg.GetAliveMsg().Membership.PkiId})
		peers2Send := filter.SelectPeers(g.conf.PropagatePeerNum, g.disc.GetMembership(), selectByOriginOrg)
		g.sendAndFilterSecrets(msg, peers2Send...)
	}
}
//代码在gossip/gossip/gossip_impl.go
```

## 5、GossipStateProvider接口定义及实现（状态复制）

### 5.1、GossipStateProvider接口定义

通过状态复制填充缺少的块，以及发送请求从其他节点获取缺少的块

```go
type GossipStateProvider interface {
	//按索引获取块
	GetBlock(index uint64) *common.Block
	//添加块
	AddPayload(payload *proto.Payload) error
	//终止状态复制
	Stop()
}
//代码在gossip/state/state.go
```

### 5.2、GossipStateProvider接口实现

```go
type GossipStateProviderImpl struct {
	mcs api.MessageCryptoService //消息加密服务
	chainID string //Chain id
	gossip GossipAdapter //gossiping service
	gossipChan <-chan *proto.GossipMessage
	commChan <-chan proto.ReceivedMessage
	payloads PayloadsBuffer //Payloads队列缓存
	committer committer.Committer
	stateResponseCh chan proto.ReceivedMessage
	stateRequestCh chan proto.ReceivedMessage
	stopCh chan struct{}
	done sync.WaitGroup
	once sync.Once
	stateTransferActive int32
}

func (s *GossipStateProviderImpl) GetBlock(index uint64) *common.Block
func (s *GossipStateProviderImpl) AddPayload(payload *proto.Payload) error
func (s *GossipStateProviderImpl) Stop()

func NewGossipStateProvider(chainID string, g GossipAdapter, committer committer.Committer, mcs api.MessageCryptoService) GossipStateProvider
func (s *GossipStateProviderImpl) listen()
func (s *GossipStateProviderImpl) directMessage(msg proto.ReceivedMessage)
func (s *GossipStateProviderImpl) processStateRequests()
func (s *GossipStateProviderImpl) handleStateRequest(msg proto.ReceivedMessage)
func (s *GossipStateProviderImpl) handleStateResponse(msg proto.ReceivedMessage) (uint64, error)
func (s *GossipStateProviderImpl) queueNewMessage(msg *proto.GossipMessage)
func (s *GossipStateProviderImpl) deliverPayloads()
func (s *GossipStateProviderImpl) antiEntropy()
func (s *GossipStateProviderImpl) maxAvailableLedgerHeight() uint64
func (s *GossipStateProviderImpl) requestBlocksInRange(start uint64, end uint64)
func (s *GossipStateProviderImpl) stateRequestMessage(beginSeq uint64, endSeq uint64) *proto.GossipMessage
func (s *GossipStateProviderImpl) selectPeerToRequestFrom(height uint64) (*comm.RemotePeer, error)
func (s *GossipStateProviderImpl) filterPeers(predicate func(peer discovery.NetworkMember) bool) []*comm.RemotePeer
func (s *GossipStateProviderImpl) hasRequiredHeight(height uint64) func(peer discovery.NetworkMember) bool
func (s *GossipStateProviderImpl) addPayload(payload *proto.Payload, blockingMode bool) error
func (s *GossipStateProviderImpl) commitBlock(block *common.Block) error

//代码在gossip/state/state.go
```

#### 5.2.1、func (s *GossipStateProviderImpl) AddPayload(payload *proto.Payload) error

```go
func (s *GossipStateProviderImpl) AddPayload(payload *proto.Payload) error {
	blockingMode := blocking
	if viper.GetBool("peer.gossip.nonBlockingCommitMode") { //非阻塞提交模式
		blockingMode = false
	}
	return s.addPayload(payload, blockingMode)
}
//代码在gossip/state/state.go
```

s.addPayload(payload, blockingMode)代码如下：

```go
func (s *GossipStateProviderImpl) addPayload(payload *proto.Payload, blockingMode bool) error {
	height, err := s.committer.LedgerHeight()
	return s.payloads.Push(payload) //加入缓存
}
//代码在gossip/state/state.go
```

#### 5.2.2、func NewGossipStateProvider(chainID string, g GossipAdapter, committer committer.Committer, mcs api.MessageCryptoService) GossipStateProvider

```go
func NewGossipStateProvider(chainID string, g GossipAdapter, committer committer.Committer, mcs api.MessageCryptoService) GossipStateProvider {
	logger := util.GetLogger(util.LoggingStateModule, "")

	gossipChan, _ := g.Accept(func(message interface{}) bool {
		return message.(*proto.GossipMessage).IsDataMsg() &&
			bytes.Equal(message.(*proto.GossipMessage).Channel, []byte(chainID))
	}, false)

	remoteStateMsgFilter := func(message interface{}) bool {
		receivedMsg := message.(proto.ReceivedMessage)
		msg := receivedMsg.GetGossipMessage()
		connInfo := receivedMsg.GetConnectionInfo()
		authErr := mcs.VerifyByChannel(msg.Channel, connInfo.Identity, connInfo.Auth.Signature, connInfo.Auth.SignedData)
		return true
	}

	_, commChan := g.Accept(remoteStateMsgFilter, true)

	height, err := committer.LedgerHeight()

	s := &GossipStateProviderImpl{
		mcs: mcs,
		chainID: chainID,
		gossip: g,
		gossipChan: gossipChan,
		commChan: commChan,
		payloads: NewPayloadsBuffer(height),
		committer: committer,
		stateResponseCh: make(chan proto.ReceivedMessage, defChannelBufferSize),
		stateRequestCh: make(chan proto.ReceivedMessage, defChannelBufferSize),
		stopCh: make(chan struct{}, 1),
		stateTransferActive: 0,
		once: sync.Once{},
	}

	nodeMetastate := NewNodeMetastate(height - 1)
	b, err := nodeMetastate.Bytes()
	s.done.Add(4)

	go s.listen()
	go s.deliverPayloads() //处理从orderer获取的块
	go s.antiEntropy() //处理缺失块
	go s.processStateRequests() //处理状态请求消息
	return s
}
//代码在gossip/state/state.go
```

#### 5.2.3、func (s *GossipStateProviderImpl) deliverPayloads()

```go
func (s *GossipStateProviderImpl) deliverPayloads() {
	defer s.done.Done()

	for {
		select {
		case <-s.payloads.Ready():
			for payload := s.payloads.Pop(); payload != nil; payload = s.payloads.Pop() {
				rawBlock := &common.Block{}
				err := pb.Unmarshal(payload.Data, rawBlock)
				err := s.commitBlock(rawBlock)
			}
		case <-s.stopCh:
			s.stopCh <- struct{}{}
			logger.Debug("State provider has been stoped, finishing to push new blocks.")
			return
		}
	}
}

func (s *GossipStateProviderImpl) commitBlock(block *common.Block) error {
	err := s.committer.Commit(block)
	nodeMetastate := NewNodeMetastate(block.Header.Number)
	b, err := nodeMetastate.Bytes()
	return nil
}
//代码在gossip/state/state.go
```

#### 5.2.4、func (s *GossipStateProviderImpl) antiEntropy()

定时获取当前高度和节点中最大高度的差值

```go
func (s *GossipStateProviderImpl) antiEntropy() {
	defer s.done.Done()

	for {
		select {
		case <-s.stopCh:
			s.stopCh <- struct{}{}
			return
		case <-time.After(defAntiEntropyInterval):
			current, err := s.committer.LedgerHeight()
			max := s.maxAvailableLedgerHeight() //最大高度
			s.requestBlocksInRange(uint64(current), uint64(max))
		}
	}
}
//代码在gossip/state/state.go
```

s.requestBlocksInRange(uint64(current), uint64(max))代码如下：

```go
func (s *GossipStateProviderImpl) requestBlocksInRange(start uint64, end uint64) {
	atomic.StoreInt32(&s.stateTransferActive, 1)
	defer atomic.StoreInt32(&s.stateTransferActive, 0)

	for prev := start; prev <= end; {
		next := min(end, prev+defAntiEntropyBatchSize) //一批最大10个
		gossipMsg := s.stateRequestMessage(prev, next) //构造GossipMessage_StateRequest

		responseReceived := false
		tryCounts := 0

		for !responseReceived {
			peer, err := s.selectPeerToRequestFrom(next) //确定peer
			s.gossip.Send(gossipMsg, peer)
			tryCounts++
			select {
			case msg := <-s.stateResponseCh:
				if msg.GetGossipMessage().Nonce != gossipMsg.Nonce {
					continue
				}
				index, err := s.handleStateResponse(msg)
				prev = index + 1
				responseReceived = true
			case <-time.After(defAntiEntropyStateResponseTimeout):
			case <-s.stopCh:
				s.stopCh <- struct{}{}
				return
			}
		}
	}
}
//代码在gossip/state/state.go
```

index, err := s.handleStateResponse(msg)代码如下：

```go
func (s *GossipStateProviderImpl) handleStateResponse(msg proto.ReceivedMessage) (uint64, error) {
	max := uint64(0)
	response := msg.GetGossipMessage().GetStateResponse()
	for _, payload := range response.GetPayloads() {
		err := s.mcs.VerifyBlock(common2.ChainID(s.chainID), payload.SeqNum, payload.Data)
		err := s.addPayload(payload, blocking) //存入本地
	}
	return max, nil
}
//代码在gossip/state/state.go
```

committer更详细内容，参考：[Fabric 1.0源代码笔记 之 Peer #committer（提交者）](../peer/committer.md)

#### 5.2.5、func (s *GossipStateProviderImpl) listen()

```go
func (s *GossipStateProviderImpl) listen() {
	defer s.done.Done()

	for {
		select {
		case msg := <-s.gossipChan:
			//处理从通道中其他节点分发的消息
			go s.queueNewMessage(msg)
		case msg := <-s.commChan:
			logger.Debug("Direct message ", msg)
			go s.directMessage(msg)
		case <-s.stopCh:
			s.stopCh <- struct{}{}
			return
		}
	}
}
//代码在gossip/state/state.go
```

go s.queueNewMessage(msg)代码如下：

```go
func (s *GossipStateProviderImpl) queueNewMessage(msg *proto.GossipMessage) {
	dataMsg := msg.GetDataMsg()
	if dataMsg != nil {
		err := s.addPayload(dataMsg.GetPayload(), nonBlocking) //写入本地
	}
}
//代码在gossip/state/state.go
```
	
## 10、MessageCryptoService接口及实现

MessageCryptoService接口定义：消息加密服务。

```go
type MessageCryptoService interface {
	//获取Peer身份的PKI ID（Public Key Infrastructure，公钥基础设施）
	GetPKIidOfCert(peerIdentity PeerIdentityType) common.PKIidType
	//校验块签名
	VerifyBlock(chainID common.ChainID, seqNum uint64, signedBlock []byte) error
	//使用Peer的签名密钥对消息签名
	Sign(msg []byte) ([]byte, error)
	//校验签名是否为有效签名
	Verify(peerIdentity PeerIdentityType, signature, message []byte) error
	//在特定通道下校验签名是否为有效签名
	VerifyByChannel(chainID common.ChainID, peerIdentity PeerIdentityType, signature, message []byte) error
	//校验Peer身份
	ValidateIdentity(peerIdentity PeerIdentityType) error
}
//代码在gossip/api/crypto.go
```

MessageCryptoService接口实现，即mspMessageCryptoService结构体及方法：

```go
type mspMessageCryptoService struct {
	channelPolicyManagerGetter policies.ChannelPolicyManagerGetter //通道策略管理器，type ChannelPolicyManagerGetter interface
	localSigner                crypto.LocalSigner //本地签名者，type LocalSigner interface
	deserializer               mgmt.DeserializersManager //反序列化管理器，type DeserializersManager interface
}

//构造mspMessageCryptoService
func NewMCS(channelPolicyManagerGetter policies.ChannelPolicyManagerGetter, localSigner crypto.LocalSigner, deserializer mgmt.DeserializersManager) api.MessageCryptoService
//调取s.getValidatedIdentity(peerIdentity)
func (s *mspMessageCryptoService) ValidateIdentity(peerIdentity api.PeerIdentityType) error
func (s *mspMessageCryptoService) GetPKIidOfCert(peerIdentity api.PeerIdentityType) common.PKIidType
func (s *mspMessageCryptoService) VerifyBlock(chainID common.ChainID, seqNum uint64, signedBlock []byte) error
func (s *mspMessageCryptoService) Sign(msg []byte) ([]byte, error)
func (s *mspMessageCryptoService) Verify(peerIdentity api.PeerIdentityType, signature, message []byte) error
func (s *mspMessageCryptoService) VerifyByChannel(chainID common.ChainID, peerIdentity api.PeerIdentityType, signature, message []byte) error
func (s *mspMessageCryptoService) getValidatedIdentity(peerIdentity api.PeerIdentityType) (msp.Identity, common.ChainID, error)
//代码在peer/gossip/mcs.go
```

## 20、本文使用的网络内容

* [hyperledger fabric 代码分析之 gossip 协议](https://zhuanlan.zhihu.com/p/27989809)
* [fabric源码解析14——peer的gossip服务之初始化](http://blog.csdn.net/idsuf698987/article/details/77898724)
