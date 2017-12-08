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
func GetGossipService() GossipService 
func (g *gossipServiceImpl) NewConfigEventer() ConfigProcessor 
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
	if s == nil { //如果指定peerServer，则创建独立的grpc Server
		c, err = createCommWithServer(conf.BindPort, idMapper, selfIdentity, secureDialOpts)
	} else { //如果指定了peerServer，则将GossipService，直接注册到peerServer上
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
	g.logger.Info("Creating gossip service with self membership of", g.selfNetworkMember())

	g.certStore = newCertStore(g.createCertStorePuller(), idMapper, selfIdentity, mcs)

	if g.conf.ExternalEndpoint == "" {
		g.logger.Warning("External endpoint is empty, peer will not be accessible outside of its organization")
	}

	go g.start()
	go g.periodicalIdentityValidationAndExpiration()
	go g.connect2BootstrapPeers()

	return g
}
//代码在gossip/gossip/gossip_impl.go
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
