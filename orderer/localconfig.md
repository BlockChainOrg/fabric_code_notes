# Fabric 1.0源代码笔记 之 Orderer（2）localconfig（Orderer配置文件定义）

## 1、配置文件定义

```bash
General:
    LedgerType: file
    ListenAddress: 127.0.0.1
    ListenPort: 7050
    TLS:
        Enabled: false
        PrivateKey: tls/server.key
        Certificate: tls/server.crt
        RootCAs:
          - tls/ca.crt
        ClientAuthEnabled: false
        ClientRootCAs:
    LogLevel: info
    LogFormat: '%{color}%{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}'
    GenesisMethod: provisional
    GenesisProfile: SampleInsecureSolo
    GenesisFile: genesisblock
    LocalMSPDir: msp
    LocalMSPID: DEFAULT
    Profile:
        Enabled: false
        Address: 0.0.0.0:6060
    BCCSP:
        Default: SW
        SW:
            Hash: SHA2
            Security: 256
            FileKeyStore:
                KeyStore:
FileLedger:
    Location: /var/hyperledger/production/orderer
    Prefix: hyperledger-fabric-ordererledger
RAMLedger:
    HistorySize: 1000
Kafka:
    Retry:
        ShortInterval: 5s
        ShortTotal: 10m
        LongInterval: 5m
        LongTotal: 12h
        NetworkTimeouts:
            DialTimeout: 10s
            ReadTimeout: 10s
            WriteTimeout: 10s
        Metadata:
            RetryBackoff: 250ms
            RetryMax: 3
        Producer:
            RetryBackoff: 100ms
            RetryMax: 3
        Consumer:
            RetryBackoff: 2s
    Verbose: false
    TLS:
      Enabled: false
      PrivateKey:
      Certificate:
      RootCAs:
    Version:
#代码在/etc/hyperledger/fabric/orderer.yaml
```








补充TopLevel结构体定义：

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


//代码在orderer/localconfig/config.go
```