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
	* api目录：消息加密服务接口定义。
		* crypto.go，MessageCryptoService接口定义。
		* channel.go，SecurityAdvisor接口定义。
* peer/gossip目录：
	* mcs.go，MessageCryptoService接口实现，即mspMessageCryptoService结构体及方法。
	* sa.go，SecurityAdvisor接口实现，即mspSecurityAdvisor结构体及方法。

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
	chains          map[string]state.GossipStateProvider
	leaderElection  map[string]election.LeaderElectionService
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
func (g *gossipServiceImpl) Stop() 
func (g *gossipServiceImpl) newLeaderElectionComponent(chainID string, callback func(bool)) election.LeaderElectionService 
func (g *gossipServiceImpl) amIinChannel(myOrg string, config Config) bool 
func (g *gossipServiceImpl) onStatusChangeFactory(chainID string, committer blocksprovider.LedgerInfo) func(bool) 
func orgListFromConfig(config Config) []string 
//代码在gossip/service/gossip_service.go
```

### 2.2.1、func InitGossipService(peerIdentity []byte, endpoint string, s *grpc.Server, mcs api.MessageCryptoService,secAdv api.SecurityAdvisor, secureDialOpts api.PeerSecureDialOpts, bootPeers ...string) error

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
	
## 3、MessageCryptoService接口及实现

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
