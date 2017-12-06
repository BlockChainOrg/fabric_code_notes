# Fabric 1.0源代码笔记 之 cryptogen（生成组织关系和身份证书）

## 1、cryptogen概述

cryptogen，用于生成组织关系和身份证书。
命令为：cryptogen generate --config=./crypto-config.yaml --output ./crypto-config

## 2、crypto-config.yaml文件示例

```bash
OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    Specs:
      - Hostname: orderer
PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: Org2
    Domain: org2.example.com
    Template:
      Count: 2
    Users:
      Count: 1
```

## 3、组织关系和身份证书目录结构

```bash
tree -L 4 crypto-config
crypto-config
├── ordererOrganizations
│   └── example.com
│       ├── ca
│       │   ├── 1366fb109b24c50e67c28b2ca4dca559eff79a85d53f833c8b9e89efdb4f4818_sk
│       │   └── ca.example.com-cert.pem
│       ├── msp
│       │   ├── admincerts
│       │   ├── cacerts
│       │   └── tlscacerts
│       ├── orderers
│       │   └── orderer.example.com
│       ├── tlsca
│       │   ├── 1a1c0f88ee3c8c49c4a48c711ee7467675ce34d92733b02fbf221834eab4053b_sk
│       │   └── tlsca.example.com-cert.pem
│       └── users
│           └── Admin@example.com
└── peerOrganizations
    ├── org1.example.com
    │   ├── ca
    │   │   ├── ca.org1.example.com-cert.pem
    │   │   └── db8cc18ebcac9670df5ec1c1e7fcdc359d70a0b1d82adeab012eea9f5131850b_sk
    │   ├── msp
    │   │   ├── admincerts
    │   │   ├── cacerts
    │   │   └── tlscacerts
    │   ├── peers
    │   │   ├── peer0.org1.example.com
    │   │   └── peer1.org1.example.com
    │   ├── tlsca
    │   │   ├── 57dcbe74092cb46f126f1e9a58f9fd225d728a1b9b5b87383ece059476d48b40_sk
    │   │   └── tlsca.org1.example.com-cert.pem
    │   └── users
    │       ├── Admin@org1.example.com
    │       └── User1@org1.example.com
    └── org2.example.com
        ├── ca
        │   ├── 8a2b52e8deda65a3a6e16c68d86aa8c4a25f0c35921f53ce2959e9f2bca956cc_sk
        │   └── ca.org2.example.com-cert.pem
        ├── msp
        │   ├── admincerts
        │   ├── cacerts
        │   └── tlscacerts
        ├── peers
        │   ├── peer0.org2.example.com
        │   └── peer1.org2.example.com
        ├── tlsca
        │   ├── acd26c09f5c89d891d3806c8d1d71e2c442ee9a58521d981980a1d45ab4ba666_sk
        │   └── tlsca.org2.example.com-cert.pem
        └── users
            ├── Admin@org2.example.com
            └── User1@org2.example.com

39 directories, 12 files
```