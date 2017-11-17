# Fabric 1.0源码旅程 之 Peer

在Fabric中，Peer（节点）是指在网络中负责接收交易请求、维护一致账本的各个fabric-peer实例。节点之间彼此通过gRPC通信。
按角色划分，Peer包括两种类型：
* Endorser（背书者）：负责对来自客户端的交易提案进行检查和背书。
* Committer（提交者）：负责检查交易请求，执行交易并维护区块链和账本结构。

Peer核心代码在peer目录下，其他相关代码分布在core/peer和protos/peer目录下。目录结构如下：

* peer
	* main.go，peer命令入口。
	* node目录，peer node命令及子命令peer node start和peer node status实现。
	* channel目录，peer channel命令及子命令实现。
	* chaincode目录，peer chaincode命令及子命令实现。
	* clilogging目录，peer clilogging命令及子命令实现。
	* version目录，peer version命令实现。
	* common目录，peer相关通用代码。
	* gossip目录，gossip最终一致性算法相关代码。
	
如下为分节说明Peer代码：

* [Fabric 1.0源码旅程 之 Peer（1）-peer命令入口及加载子命令](peer_main.md)
