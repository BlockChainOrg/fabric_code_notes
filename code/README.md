# 从源码角度白话解读Hyperledger Fabric运行全过程

### 三种节点和作用

Fabric有三种节点：CA、peer和orderer，其中peer节点包含两种职能endorser和commiter。
其中CA节点负责Fabirc网络中身份证书的管理，orderer节点负责网络中所有交易的排序。endorser负责所有交易的模拟执行和背书，而commiter负责最终交易向区块链账本中的提交和写入。
一般的业务流程为：客户端从CA获取证书，向多个endorser发起链码执行请求（即交易提案），endorser模拟执行链码并对执行结果背书，
客户端收集符合要求的背书后，打包交易提交给orderer，orderer将交易排序后组成区块发送给commiter，由commiter写入各自节点区块账本中。

### 两个初始化工具

Fabric中有两个关键的初始化工具：cryptogen和configtxgen，其中cryptogen用于用于生成组织结构和身份证书文件，configtxgen用于生成通道相关配置文件。

创建Fabric区块链网络之初，即需先使用cryptogen工具创建组织、组织成员、用户，以及各自的身份证书。
cryptogen工具依赖配置文件crypto-config.yaml，这个配置文件中支持定义两种组织：OrdererOrgs和PeerOrgs。
其中OrdererOrgs定义orderer节点的名称、域、主机名；PeerOrgs中定义多个组织，以及各自的名称、域、节点数量、用户数量。
配置文件定义完后，执行cryptogen工具，即可基于配置文件创建：orderer和peer节点证书和私钥，以及管理员、用户的证书和私钥。

Fabric中支持多通道，即多链，通道几乎可等同为链。有关Fabric网络的配置，即存储在一种特殊的系统通道中。系统通道为Fabric网络的第一个通道。
configtxgen的第一个功能即为创建系统通道的初始区块，该区块文件用于orderer节点的启动。
configtxgen依赖配置文件configtx.yaml，该配置文件中可以定义orderer节点的配置，如共识插件、身份证书、加密算法等，以及peer节点的配置，如身份证书、加密算法、锚点等。
configtxgen ... -outputBlock ...，即可基于配置文件创建系统通道的初始区块。

系统通道创建后，需创建应用通道，而configtxgen第二个功能即为生成创建应用通道所需的配置交易文件，该配置交易文件可直接用于在peer节点创建应用通道时使用。
configtxgen ... -outputCreateChannelTx ...即可基于配置文件创建应用通道配置交易文件。
