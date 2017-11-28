# Fabric 1.0源代码笔记 之 Orderer（2）localconfig（Orderer配置文件定义）

## 1、配置文件定义

```bash
General: #通用配置
    LedgerType: file #账本类型，包括ram、json和file，其中ram保存在内存中，生产环境推荐使用file
    ListenAddress: 127.0.0.1 #服务绑定的监听地址
    ListenPort: 7050 #服务绑定的监听端口
    TLS: #启用TLS时的相关配置
        Enabled: false #是否启用TLS
        PrivateKey: tls/server.key #Orderer签名私钥
        Certificate: tls/server.crt #Orderer身份证书
        RootCAs: #信任的根证书
          - tls/ca.crt
        ClientAuthEnabled: false #是否对客户端也进行认证
        ClientRootCAs:
    LogLevel: info #日志级别
    LogFormat: '%{color}%{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}'
    GenesisMethod: provisional #初始区块的提供方式，支持provisional或file，前者基于GenesisProfile指定的configtx.yaml中Profile生成，后者基于指定的初始区块文件
    GenesisProfile: SampleInsecureSolo #provisional方式生成初始区块时采用的Profile
    GenesisFile: genesisblock #使用现成的初始区块文件时，文件的路径
    LocalMSPDir: msp #本地msp文件的路径
    LocalMSPID: DEFAULT #MSP的ID
    Profile: #是否启用go profiling
        Enabled: false
        Address: 0.0.0.0:6060
    BCCSP: #密码库机制等，可以为SW（软件实现）或PKCS11（硬件安全模块）
        Default: SW
        SW:
            Hash: SHA2 #哈希算法类型
            Security: 256
            FileKeyStore: #本地私钥文件路径，默认指向<mspConfigPath>/keystore
                KeyStore:
FileLedger: #基于文件的账本的配置
    Location: /var/hyperledger/production/orderer #存放区块文件的位置，一般为/var/hyperledger/production/orderer/目录
    Prefix: hyperledger-fabric-ordererledger #如果不指定Location，则在临时目录下创建账本时使用的目录名称
RAMLedger: #基于内存的账本最多保留的区块个数
    HistorySize: 1000
Kafka: #Orderer使用Kafka集群作为后端时，Kafka的配置
    Retry: #Kafka未就绪时Orderer的重试配置，orderer会利用sarama客户端为channel创建一个producer、一个consumer，分别向Kafka写和读数据
        ShortInterval: 5s #操作失败后的快速重试阶段的间隔
        ShortTotal: 10m #快速重试阶段最多重试多长时间
        LongInterval: 5m #快速重试阶段仍然失败后进入慢重试阶段，慢重试阶段的时间间隔
        LongTotal: 12h #慢重试阶段最多重试多长时间
        NetworkTimeouts: #sarama网络超时时间
            DialTimeout: 10s
            ReadTimeout: 10s
            WriteTimeout: 10s
        Metadata: #Kafka集群leader选举中的metadata请求参数
            RetryBackoff: 250ms
            RetryMax: 3
        Producer: #发送消息到Kafka集群的超时
            RetryBackoff: 100ms
            RetryMax: 3
        Consumer: #从Kafka集群读取消息的超时
            RetryBackoff: 2s
    Verbose: false #是否开启Kafka客户端的调试日志
    TLS: #Kafka集群的连接启用TLS时的相关配置
      Enabled: false #是否启用TLS，默认不开启
      PrivateKey: #Orderer证明身份用的签名私钥
      Certificate: #Kafka身份证书
      RootCAs: #验证Kafka证书时的CA证书
    Version: #Kafka版本号
#代码在/etc/hyperledger/fabric/orderer.yaml
```

## 2、TopLevel结构体定义

```go
type TopLevel struct {
	General    General
	FileLedger FileLedger
	RAMLedger  RAMLedger
	Kafka      Kafka
}

type General struct {
	LedgerType     string
	ListenAddress  string
	ListenPort     uint16
	TLS            TLS
	GenesisMethod  string
	GenesisProfile string
	GenesisFile    string
	Profile        Profile
	LogLevel       string
	LogFormat      string
	LocalMSPDir    string
	LocalMSPID     string
	BCCSP          *bccsp.FactoryOpts
}

type TLS struct {
	Enabled           bool
	PrivateKey        string
	Certificate       string
	RootCAs           []string
	ClientAuthEnabled bool
	ClientRootCAs     []string
}

type Profile struct {
	Enabled bool
	Address string
}

type FileLedger struct {
	Location string
	Prefix   string
}

type RAMLedger struct {
	HistorySize uint
}

type Kafka struct {
	Retry   Retry
	Verbose bool
	Version sarama.KafkaVersion
	TLS     TLS
}

type Retry struct {
	ShortInterval   time.Duration
	ShortTotal      time.Duration
	LongInterval    time.Duration
	LongTotal       time.Duration
	NetworkTimeouts NetworkTimeouts
	Metadata        Metadata
	Producer        Producer
	Consumer        Consumer
}

type NetworkTimeouts struct {
	DialTimeout  time.Duration
	ReadTimeout  time.Duration
	WriteTimeout time.Duration
}

type Metadata struct {
	RetryMax     int
	RetryBackoff time.Duration
}

type Producer struct {
	RetryMax     int
	RetryBackoff time.Duration
}

type Consumer struct {
	RetryBackoff time.Duration
}
//代码在orderer/localconfig/config.go
```