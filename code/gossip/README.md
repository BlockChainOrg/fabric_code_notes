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
	* api目录：消息加密服务接口定义。
		* crypto.go，MessageCryptoService接口定义。
		* channel.go，SecurityAdvisor接口定义。
* peer/gossip目录：
	* mcs.go，MessageCryptoService接口实现，即mspMessageCryptoService结构体及方法。
	* sa.go，SecurityAdvisor接口实现，即mspSecurityAdvisor结构体及方法。
	
## 2、MessageCryptoService接口及实现

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

补充policies.ChannelPolicyManagerGetter：

## 20、本文使用的网络内容

* [hyperledger fabric 代码分析之 gossip 协议](https://zhuanlan.zhihu.com/p/27989809)
* [fabric源码解析14——peer的gossip服务之初始化](http://blog.csdn.net/idsuf698987/article/details/77898724)
