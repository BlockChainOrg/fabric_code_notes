# Fabric 1.0源代码笔记 之 configtx（配置交易） #configtxgen（生成通道配置）

## 1、configtxgen概述

configtxgen，用于生成通道配置，具体有如下三种用法：

* 生成Orderer服务启动的初始区块
	* configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
* 生成新建应用通道的配置交易
	* configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel
* 生成锚节点配置更新文件
	* configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP
	
configtxgen代码分布在common/configtx/tool目录，目录结构如下：

* localconfig/config.go，configtx.yaml配置文件相关的结构体及方法。

## 2、configtx.yaml配置文件示例

```bash
Profiles:
    TwoOrgsOrdererGenesis: #Orderer系统通道配置
        Orderer:
            <<: *OrdererDefaults #引用OrdererDefaults并合并到当前
            Organizations: #属于Orderer通道的组织
                - *OrdererOrg 
        Consortiums: #Orderer所服务的联盟列表
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2
    TwoOrgsChannel: #应用通道配置
        Consortium: SampleConsortium #应用通道关联的联盟
        Application: 
            <<: *ApplicationDefaults #引用ApplicationDefaults并合并到当前
            Organizations: #初始加入应用通道的组织
                - *Org1
                - *Org2
Organizations:
    - &OrdererOrg
        Name: OrdererOrg
        ID: OrdererMSP # MSP ID
        MSPDir: crypto-config/ordererOrganizations/example.com/msp #MSP相关文件本地路径
    - &Org1
        Name: Org1MSP
        ID: Org1MSP
        MSPDir: crypto-config/peerOrganizations/org1.example.com/msp
        AnchorPeers: #锚节点地址，用于跨组织的Gossip通信
            - Host: peer0.org1.example.com
              Port: 7051
    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: crypto-config/peerOrganizations/org2.example.com/msp
        AnchorPeers: #锚节点地址，用于跨组织的Gossip通信
            - Host: peer0.org2.example.com
              Port: 7051
Orderer: &OrdererDefaults
    OrdererType: solo # Orderer共识插件类型，分solo或kafka
    Addresses:
        - orderer.example.com:7050 #服务地址
    BatchTimeout: 2s #创建批量交易的最大超时，一批交易构成一个块
    BatchSize: #写入区块内的交易个数
        MaxMessageCount: 10 #一批消息的最大个数
        AbsoluteMaxBytes: 98 MB #一批交易的最大字节数，任何时候均不能超过
        PreferredMaxBytes: 512 KB #批量交易的建议字节数
    Kafka:
        Brokers: #Kafka端口
            - 127.0.0.1:9092
    Organizations:
Application: &ApplicationDefaults
    Organizations: #加入到通道的组织信息，此处为不包括任何组织
```

配置文件解读：

* 每个Profile表示某种场景下的通道配置模板，包括Orderer系统通道模板和应用通道模板，其中TwoOrgsOrdererGenesis为系统通道模板，TwoOrgsChannel为应用通道模板。
* Orderer系统通道模板，包括Orderer和Consortiums，其中Orderer指定系统通道配置，Consortiums为Orderer服务的联盟列表。
* 应用通道，包括Application和Consortium，其中Application为应用通道的配置，Consortium为应用通道所关联的联盟名称。
	
附：[YAML 语言教程](http://www.ruanyifeng.com/blog/2016/07/yaml.html?f=tt)
-表示数组，&表示锚点，*表示引用，<<表示合并到当前数据。

## 3、configtx.yaml配置文件相关的结构体及方法

### 3.1、configtx.yaml配置文件相关的结构体定义

```go

//代码在common/configtx/tool/localconfig/
```