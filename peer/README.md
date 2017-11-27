# Fabric 1.0源代码笔记 之 Peer

## 1、Peer概述

在Fabric中，Peer（节点）是指在网络中负责接收交易请求、维护一致账本的各个fabric-peer实例。节点之间彼此通过gRPC通信。
按角色划分，Peer包括两种类型：
* Endorser（背书者）：负责对来自客户端的交易提案进行检查和背书。
* Committer（提交者）：负责检查交易请求，执行交易并维护区块链和账本结构。

Peer核心代码在peer目录下，其他相关代码分布在core/peer和protos/peer目录下。目录结构如下：

* peer目录：
	* main.go，peer命令入口。
	* node目录，peer node命令及子命令peer node start和peer node status实现。
		* node.go，peer node命令入口。
		* start.go，peer node start子命令实现。
		* status.go，peer node status子命令实现。
	* channel目录，peer channel命令及子命令实现。
	* chaincode目录，peer chaincode命令及子命令实现。
	* clilogging目录，peer clilogging命令及子命令实现。
	* version目录，peer version命令实现。
	* common目录，peer相关通用代码。
	* gossip目录，gossip最终一致性算法相关代码。
* core/peer目录：
	config.go，Peer配置相关工具函数。
	peer.go，Peer服务相关工具函数。
	
如下为分节说明Peer代码：

* [Fabric 1.0源代码笔记 之 Peer（1）peer根命令入口及加载子命令](peer_main.md)
* [Fabric 1.0源代码笔记 之 Peer（2）peer node start命令实现](peer_node_start.md)

## 2、Peer配置相关工具函数

```go
//为全局变量localAddress和peerEndpoint赋值
func CacheConfiguration() (err error) 
func cacheConfiguration() //调用CacheConfiguration()
//获取localAddress
func GetLocalAddress() (string, error)
//获取peerEndpoint
func GetPeerEndpoint() (*pb.PeerEndpoint, error) 
//获取Peer安全配置
func GetSecureConfig() (comm.SecureServerConfig, error) 
//代码在core/peer/config.go
```

PeerEndpoint结构体定义如下：

```go
type PeerID struct {
	Name string
}

type PeerEndpoint struct {
	Id      *PeerID
	Address string
}
//代码在protos/peer/peer.pb.go
```

SecureServerConfig结构体定义如下：

```go
type SecureServerConfig struct {
	ServerCertificate []byte //签名公钥，取自peer.tls.cert.file
	ServerKey []byte //签名私钥，取自peer.tls.key.file
	ServerRootCAs [][]byte //根CA证书，取自peer.tls.rootcert.file
	ClientRootCAs [][]byte
	UseTLS bool //是否启用TLS，取自peer.tls.enabled
	RequireClientCert bool
}
//代码在core/comm/server.go
```

## 3、Peer服务相关工具函数

```go
func (cs *chainSupport) Ledger() ledger.PeerLedger
func (cs *chainSupport) GetMSPIDs(cid string) []string
func MockInitialize()
func MockSetMSPIDGetter(mspIDGetter func(string) []string)
func Initialize(init func(string)) //Peer初始化，并部署系统链码
func InitChain(cid string)
func getCurrConfigBlockFromLedger(ledger ledger.PeerLedger) (*common.Block, error)
func createChain(cid string, ledger ledger.PeerLedger, cb *common.Block) error
func CreateChainFromBlock(cb *common.Block) error
func MockCreateChain(cid string) error
func GetLedger(cid string) ledger.PeerLedger
func GetPolicyManager(cid string) policies.Manager
func GetCurrConfigBlock(cid string) *common.Block
func updateTrustedRoots(cm configtxapi.Manager)
func buildTrustedRootsForChain(cm configtxapi.Manager)
func GetMSPIDs(cid string) []string
func SetCurrConfigBlock(block *common.Block, cid string) error
func NewPeerClientConnection() (*grpc.ClientConn, error)
func GetLocalIP() string
func NewPeerClientConnectionWithAddress(peerAddress string) (*grpc.ClientConn, error)
func GetChannelsInfo() []*pb.ChannelInfo
//构造type channelPolicyManagerGetter struct{}
func NewChannelPolicyManagerGetter() policies.ChannelPolicyManagerGetter
func (c *channelPolicyManagerGetter) Manager(channelID string) (policies.Manager, bool)
func CreatePeerServer(listenAddress string,secureConfig comm.SecureServerConfig) (comm.GRPCServer, error)
func GetPeerServer() comm.GRPCServer
//代码在core/peer/peer.go
```