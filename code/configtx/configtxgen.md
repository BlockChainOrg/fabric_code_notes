# Fabric 1.0源代码笔记 之 configtx（配置交易） #configtxgen（生成通道配置）

## 1、configtxgen概述

configtxgen，用于生成通道配置，具体有如下三种用法：

* 生成Orderer服务启动的初始区块（即系统通道的创世区块文件）
	* configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
* 生成新建应用通道的配置交易（即用于创建应用通道的配置交易文件）
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
    Organizations: #参与维护Orderer的组织，默认空
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
type TopLevel struct {
	Profiles      map[string]*Profile //通道配置
	Organizations []*Organization //组织
	Application   *Application //应用通道配置
	Orderer       *Orderer //系统通道配置
}

type Profile struct { //通道配置：系统通道配置或应用通道配置
	Consortium  string //应用通道配置中通道所关联的联盟名称
	Application *Application //应用通道配置
	Orderer     *Orderer //系统通道配置
	Consortiums map[string]*Consortium //系统通道配置中Orderer服务的联盟列表
}

type Consortium struct { //联盟，即组织列表
	Organizations []*Organization //组织
}

type Application struct { //应用通道配置，即初始加入通道的组织
	Organizations []*Organization
}

type Organization struct { //组织
	Name           string //组织名称
	ID             string //组织MSP ID
	MSPDir         string //组织MSP文件所在路径
	AdminPrincipal string //管理员身份规则
	AnchorPeers []*AnchorPeer //锚节点列表
}

type AnchorPeer struct { //锚节点，即主机和端口
	Host string
	Port int
}

type Orderer struct { //系统通道配置
	OrdererType   string //共识插件类型
	Addresses     []string //Orderer服务地址
	BatchTimeout  time.Duration //批处理超时
	BatchSize     BatchSize //批处理大小
	Kafka         Kafka //Kafka
	Organizations []*Organization //参与维护Orderer的组织，默认空
	MaxChannels   uint64 //Orderer最大通道数
}

type BatchSize struct { //批处理大小
	MaxMessageCount   uint32 //最大交易数量
	AbsoluteMaxBytes  uint32 //最大字节数
	PreferredMaxBytes uint32 //建议字节数
}

type Kafka struct {
	Brokers []string //Kafka Broker
}
//代码在common/configtx/tool/localconfig/config.go
```

### 3.2、configtx.yaml配置文件相关的方法

```go
//获取指定profile的Profile结构
func Load(profile string) *Profile 
//将Profile校验并补充完整
func (p *Profile) completeInitialization(configDir string) 
func translatePaths(configDir string, org *Organization) 
//代码在common/configtx/tool/localconfig/config.go
```

## 4、Generator接口及实现

Generator接口定义：

```go
type Generator interface {
	bootstrap.Helper
	ChannelTemplate() configtx.Template //获取用于初始化通道的模板
	GenesisBlockForChannel(channelID string) *cb.Block //用于outputBlock
}
//代码在common/configtx/tool/provisional/provisional.go
```

Generator接口实现，即bootstrapper。

```go
type bootstrapper struct {
	channelGroups     []*cb.ConfigGroup
	ordererGroups     []*cb.ConfigGroup
	applicationGroups []*cb.ConfigGroup
	consortiumsGroups []*cb.ConfigGroup
}

func New(conf *genesisconfig.Profile) Generator
func (bs *bootstrapper) ChannelTemplate() configtx.Template
func (bs *bootstrapper) GenesisBlock() *cb.Block
func (bs *bootstrapper) GenesisBlockForChannel(channelID string) *cb.Block
//代码在common/configtx/tool/provisional/provisional.go
```

func New(conf *genesisconfig.Profile) Generator代码如下：

```go
func New(conf *genesisconfig.Profile) Generator {
	bs := &bootstrapper{
		channelGroups: []*cb.ConfigGroup{
			config.DefaultHashingAlgorithm(), //默认哈希函数
			config.DefaultBlockDataHashingStructure(), //默认块数据哈希结构
			
			//默认通道策略，包括读策略、写策略和管理策略
			//ReadersPolicyKey = "Readers"，ImplicitMetaPolicy_ANY，任意
			policies.TemplateImplicitMetaAnyPolicy([]string{}, configvaluesmsp.ReadersPolicyKey),
			//WritersPolicyKey = "Writers"，ImplicitMetaPolicy_ANY，任意
			policies.TemplateImplicitMetaAnyPolicy([]string{}, configvaluesmsp.WritersPolicyKey),
			//AdminsPolicyKey = "Admins"，ImplicitMetaPolicy_MAJORITY，大多数
			policies.TemplateImplicitMetaMajorityPolicy([]string{}, configvaluesmsp.AdminsPolicyKey),
		},
	}

	if conf.Orderer != nil { //系统通道配置
		oa := config.TemplateOrdererAddresses(conf.Orderer.Addresses) //设置Orderer地址
		oa.Values[config.OrdererAddressesKey].ModPolicy = OrdererAdminsPolicy //OrdererAdminsPolicy = "/Channel/Orderer/Admins"

		bs.ordererGroups = []*cb.ConfigGroup{
			oa,
			config.TemplateConsensusType(conf.Orderer.OrdererType), //设置共识插件类型
			config.TemplateBatchSize(&ab.BatchSize{ //设置批处理大小
				MaxMessageCount:   conf.Orderer.BatchSize.MaxMessageCount,
				AbsoluteMaxBytes:  conf.Orderer.BatchSize.AbsoluteMaxBytes,
				PreferredMaxBytes: conf.Orderer.BatchSize.PreferredMaxBytes,
			}),
			config.TemplateBatchTimeout(conf.Orderer.BatchTimeout.String()), //设置批处理超时
			config.TemplateChannelRestrictions(conf.Orderer.MaxChannels), //设置最大通道数

			//初始化Orderer读、写、管理策略
			policies.TemplateImplicitMetaPolicyWithSubPolicy([]string{config.OrdererGroupKey}, BlockValidationPolicyKey, configvaluesmsp.WritersPolicyKey, cb.ImplicitMetaPolicy_ANY),
			policies.TemplateImplicitMetaAnyPolicy([]string{config.OrdererGroupKey}, configvaluesmsp.ReadersPolicyKey),
			policies.TemplateImplicitMetaAnyPolicy([]string{config.OrdererGroupKey}, configvaluesmsp.WritersPolicyKey),
			policies.TemplateImplicitMetaMajorityPolicy([]string{config.OrdererGroupKey}, configvaluesmsp.AdminsPolicyKey),
		}

		for _, org := range conf.Orderer.Organizations {
			mspConfig, err := msp.GetVerifyingMspConfig(org.MSPDir, org.ID)
			bs.ordererGroups = append(bs.ordererGroups,
				configvaluesmsp.TemplateGroupMSPWithAdminRolePrincipal([]string{config.OrdererGroupKey, org.Name},
					mspConfig, org.AdminPrincipal == genesisconfig.AdminRoleAdminPrincipal,
				),
			)
		}

		switch conf.Orderer.OrdererType {
		case ConsensusTypeSolo:
		case ConsensusTypeKafka:
			bs.ordererGroups = append(bs.ordererGroups, config.TemplateKafkaBrokers(conf.Orderer.Kafka.Brokers)) //设置Kafka
		default:
		}
	}

	if conf.Application != nil { //应用通道配置
		bs.applicationGroups = []*cb.ConfigGroup{
			policies.TemplateImplicitMetaAnyPolicy([]string{config.ApplicationGroupKey}, configvaluesmsp.ReadersPolicyKey),
			policies.TemplateImplicitMetaAnyPolicy([]string{config.ApplicationGroupKey}, configvaluesmsp.WritersPolicyKey),
			policies.TemplateImplicitMetaMajorityPolicy([]string{config.ApplicationGroupKey}, configvaluesmsp.AdminsPolicyKey),
		}
		for _, org := range conf.Application.Organizations {
			mspConfig, err := msp.GetVerifyingMspConfig(org.MSPDir, org.ID)
			bs.applicationGroups = append(bs.applicationGroups,
				configvaluesmsp.TemplateGroupMSPWithAdminRolePrincipal([]string{config.ApplicationGroupKey, org.Name},
					mspConfig, org.AdminPrincipal == genesisconfig.AdminRoleAdminPrincipal,
				),
			)
			var anchorProtos []*pb.AnchorPeer
			for _, anchorPeer := range org.AnchorPeers {
				anchorProtos = append(anchorProtos, &pb.AnchorPeer{
					Host: anchorPeer.Host,
					Port: int32(anchorPeer.Port),
				})
			}
			bs.applicationGroups = append(bs.applicationGroups, config.TemplateAnchorPeers(org.Name, anchorProtos))
		}

	}

	if conf.Consortiums != nil { //联盟相关
		tcg := config.TemplateConsortiumsGroup()
		tcg.Groups[config.ConsortiumsGroupKey].ModPolicy = OrdererAdminsPolicy
		tcg.Groups[config.ConsortiumsGroupKey].Policies[configvaluesmsp.AdminsPolicyKey] = &cb.ConfigPolicy{
			Policy: &cb.Policy{
				Type:  int32(cb.Policy_SIGNATURE),
				Value: utils.MarshalOrPanic(cauthdsl.AcceptAllPolicy),
			},
			ModPolicy: OrdererAdminsPolicy,
		}
		bs.consortiumsGroups = append(bs.consortiumsGroups, tcg)
		for consortiumName, consortium := range conf.Consortiums {
			cg := config.TemplateConsortiumChannelCreationPolicy(consortiumName, policies.ImplicitMetaPolicyWithSubPolicy(
				configvaluesmsp.AdminsPolicyKey,
				cb.ImplicitMetaPolicy_ANY,
			).Policy)

			cg.Groups[config.ConsortiumsGroupKey].Groups[consortiumName].ModPolicy = OrdererAdminsPolicy
			cg.Groups[config.ConsortiumsGroupKey].Groups[consortiumName].Values[config.ChannelCreationPolicyKey].ModPolicy = OrdererAdminsPolicy
			bs.consortiumsGroups = append(bs.consortiumsGroups, cg)

			for _, org := range consortium.Organizations {
				mspConfig, err := msp.GetVerifyingMspConfig(org.MSPDir, org.ID)
				bs.consortiumsGroups = append(bs.consortiumsGroups,
					configvaluesmsp.TemplateGroupMSPWithAdminRolePrincipal(
						[]string{config.ConsortiumsGroupKey, consortiumName, org.Name},
						mspConfig, org.AdminPrincipal == genesisconfig.AdminRoleAdminPrincipal,
					),
				)
			}
		}
	}

	return bs
}
//代码在common/configtx/tool/provisional/provisional.go
```

func (bs *bootstrapper) GenesisBlockForChannel(channelID string) *cb.Block代码如下：

```go
func (bs *bootstrapper) GenesisBlockForChannel(channelID string) *cb.Block {
	block, err := genesis.NewFactoryImpl(
		configtx.NewModPolicySettingTemplate(
			configvaluesmsp.AdminsPolicyKey,
			configtx.NewCompositeTemplate(
				configtx.NewSimpleTemplate(bs.consortiumsGroups...),
				bs.ChannelTemplate(),
			),
		),
	).Block(channelID)
	return block
}

//代码在common/configtx/tool/provisional/provisional.go
```

## 5、configtxgen命令

### 5.1、main函数

```go
func main() {
	var outputBlock, outputChannelCreateTx, profile, channelID, inspectBlock, inspectChannelCreateTx, outputAnchorPeersUpdate, asOrg string

	//-outputBlock，初始区块写入指定文件
	flag.StringVar(&outputBlock, "outputBlock", "", "The path to write the genesis block to (if set)")
	//-channelID，指定通道名称
	flag.StringVar(&channelID, "channelID", provisional.TestChainID, "The channel ID to use in the configtx")
	//-outputCreateChannelTx，将通道创建交易写入指定文件
	flag.StringVar(&outputChannelCreateTx, "outputCreateChannelTx", "", "The path to write a channel creation configtx to (if set)")
	//-profile，指定profile
	flag.StringVar(&profile, "profile", genesisconfig.SampleInsecureProfile, "The profile from configtx.yaml to use for generation.")
	//-inspectBlock，打印指定区块的配置信息
	flag.StringVar(&inspectBlock, "inspectBlock", "", "Prints the configuration contained in the block at the specified path")
	//-inspectChannelCreateTx，打印通道创建交易文件中的配置更新信息
	flag.StringVar(&inspectChannelCreateTx, "inspectChannelCreateTx", "", "Prints the configuration contained in the transaction at the specified path")
	//-outputAnchorPeersUpdate，生成锚节点配置更新文件，需同时指定-asOrg
	flag.StringVar(&outputAnchorPeersUpdate, "outputAnchorPeersUpdate", "", "Creates an config update to update an anchor peer (works only with the default channel creation, and only for the first update)")
	//-asOrg，以指定身份执行更新配置交易，如更新锚节点配置信息
	flag.StringVar(&asOrg, "asOrg", "", "Performs the config generation as a particular organization (by name), only including values in the write set that org (likely) has privilege to set")
	flag.Parse()

	factory.InitFactories(nil)
	config := genesisconfig.Load(profile) //读取指定配置

	if outputBlock != "" { //生成Orderer服务启动的初始区块
		err := doOutputBlock(config, channelID, outputBlock)
	}
	if outputChannelCreateTx != "" { //生成新建应用通道的配置交易
		err := doOutputChannelCreateTx(config, channelID, outputChannelCreateTx)
	}
	if outputAnchorPeersUpdate != "" { //生成锚节点配置更新文件
		err := doOutputAnchorPeersUpdate(config, channelID, outputAnchorPeersUpdate, asOrg)
	}
}
//代码在common/configtx/tool/configtxgen/main.go
```

### 5.2、doOutputBlock（生成Orderer服务启动的初始区块，即系统通道的初始区块文件）

```go
func doOutputBlock(config *genesisconfig.Profile, channelID string, outputBlock string) error {
	pgen := provisional.New(config) //构建Generator实例
	genesisBlock := pgen.GenesisBlockForChannel(channelID) //生成创世区块
	err := ioutil.WriteFile(outputBlock, utils.MarshalOrPanic(genesisBlock), 0644) //创世区块写入文件
	return nil
}
//代码在common/configtx/tool/configtxgen/main.go
```

genesis更详细内容，参考：[Fabric 1.0源代码笔记 之 configtx（配置交易） #genesis（系统通道创世区块）](genesis.md)

### 5.3、doOutputChannelCreateTx（生成新建应用通道的配置交易）

```go
func doOutputChannelCreateTx(conf *genesisconfig.Profile, channelID string, outputChannelCreateTx string) error {
	var orgNames []string
	for _, org := range conf.Application.Organizations {
		orgNames = append(orgNames, org.Name)
	}
	configtx, err := configtx.MakeChainCreationTransaction(channelID, conf.Consortium, nil, orgNames...)
	err = ioutil.WriteFile(outputChannelCreateTx, utils.MarshalOrPanic(configtx), 0644)
	return nil
}
//代码在common/configtx/tool/configtxgen/main.go
```

### 5.4、doOutputAnchorPeersUpdate（生成锚节点配置更新文件）

```go
func doOutputAnchorPeersUpdate(conf *genesisconfig.Profile, channelID string, outputAnchorPeersUpdate string, asOrg string) error {
	var org *genesisconfig.Organization
	for _, iorg := range conf.Application.Organizations {
		if iorg.Name == asOrg {
			org = iorg
		}
	}
	anchorPeers := make([]*pb.AnchorPeer, len(org.AnchorPeers))
	for i, anchorPeer := range org.AnchorPeers {
		anchorPeers[i] = &pb.AnchorPeer{
			Host: anchorPeer.Host,
			Port: int32(anchorPeer.Port),
		}
	}

	configGroup := config.TemplateAnchorPeers(org.Name, anchorPeers)
	configGroup.Groups[config.ApplicationGroupKey].Groups[org.Name].Values[config.AnchorPeersKey].ModPolicy = mspconfig.AdminsPolicyKey
	configUpdate := &cb.ConfigUpdate{
		ChannelId: channelID,
		WriteSet:  configGroup,
		ReadSet:   cb.NewConfigGroup(),
	}

	configUpdate.ReadSet.Groups[config.ApplicationGroupKey] = cb.NewConfigGroup()
	configUpdate.ReadSet.Groups[config.ApplicationGroupKey].Version = 1
	configUpdate.ReadSet.Groups[config.ApplicationGroupKey].ModPolicy = mspconfig.AdminsPolicyKey
	configUpdate.ReadSet.Groups[config.ApplicationGroupKey].Groups[org.Name] = cb.NewConfigGroup()
	configUpdate.ReadSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Values[config.MSPKey] = &cb.ConfigValue{}
	configUpdate.ReadSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Policies[mspconfig.ReadersPolicyKey] = &cb.ConfigPolicy{}
	configUpdate.ReadSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Policies[mspconfig.WritersPolicyKey] = &cb.ConfigPolicy{}
	configUpdate.ReadSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Policies[mspconfig.AdminsPolicyKey] = &cb.ConfigPolicy{}

	configUpdate.WriteSet.Groups[config.ApplicationGroupKey].Version = 1
	configUpdate.WriteSet.Groups[config.ApplicationGroupKey].ModPolicy = mspconfig.AdminsPolicyKey
	configUpdate.WriteSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Version = 1
	configUpdate.WriteSet.Groups[config.ApplicationGroupKey].Groups[org.Name].ModPolicy = mspconfig.AdminsPolicyKey
	configUpdate.WriteSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Values[config.MSPKey] = &cb.ConfigValue{}
	configUpdate.WriteSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Policies[mspconfig.ReadersPolicyKey] = &cb.ConfigPolicy{}
	configUpdate.WriteSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Policies[mspconfig.WritersPolicyKey] = &cb.ConfigPolicy{}
	configUpdate.WriteSet.Groups[config.ApplicationGroupKey].Groups[org.Name].Policies[mspconfig.AdminsPolicyKey] = &cb.ConfigPolicy{}

	configUpdateEnvelope := &cb.ConfigUpdateEnvelope{
		ConfigUpdate: utils.MarshalOrPanic(configUpdate),
	}

	update := &cb.Envelope{
		Payload: utils.MarshalOrPanic(&cb.Payload{
			Header: &cb.Header{
				ChannelHeader: utils.MarshalOrPanic(&cb.ChannelHeader{
					ChannelId: channelID,
					Type:      int32(cb.HeaderType_CONFIG_UPDATE),
				}),
			},
			Data: utils.MarshalOrPanic(configUpdateEnvelope),
		}),
	}

	err := ioutil.WriteFile(outputAnchorPeersUpdate, utils.MarshalOrPanic(update), 0644)
	return nil
}

//代码在common/configtx/tool/configtxgen/main.go
```