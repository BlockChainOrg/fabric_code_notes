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
tree crypto-config
crypto-config
├── ordererOrganizations
│   └── example.com
│       ├── ca
│       │   ├── 1366fb109b24c50e67c28b2ca4dca559eff79a85d53f833c8b9e89efdb4f4818_sk
│       │   └── ca.example.com-cert.pem
│       ├── msp
│       │   ├── admincerts
│       │   │   └── Admin@example.com-cert.pem
│       │   ├── cacerts
│       │   │   └── ca.example.com-cert.pem
│       │   └── tlscacerts
│       │       └── tlsca.example.com-cert.pem
│       ├── orderers
│       │   └── orderer.example.com
│       │       ├── msp
│       │       │   ├── admincerts
│       │       │   │   └── Admin@example.com-cert.pem
│       │       │   ├── cacerts
│       │       │   │   └── ca.example.com-cert.pem
│       │       │   ├── keystore
│       │       │   │   └── 71a37b3827738ec705ae92cc69adcaee1661d2445501b4189c576db9443c1edd_sk
│       │       │   ├── signcerts
│       │       │   │   └── orderer.example.com-cert.pem
│       │       │   └── tlscacerts
│       │       │       └── tlsca.example.com-cert.pem
│       │       └── tls
│       │           ├── ca.crt
│       │           ├── server.crt
│       │           └── server.key
│       ├── tlsca
│       │   ├── 1a1c0f88ee3c8c49c4a48c711ee7467675ce34d92733b02fbf221834eab4053b_sk
│       │   └── tlsca.example.com-cert.pem
│       └── users
│           └── Admin@example.com
│               ├── msp
│               │   ├── admincerts
│               │   │   └── Admin@example.com-cert.pem
│               │   ├── cacerts
│               │   │   └── ca.example.com-cert.pem
│               │   ├── keystore
│               │   │   └── 0830c91efdcf15869e4c8b1cb9de5774376eff5ef1b447f1c5642b8c4ebef9d5_sk
│               │   ├── signcerts
│               │   │   └── Admin@example.com-cert.pem
│               │   └── tlscacerts
│               │       └── tlsca.example.com-cert.pem
│               └── tls
│                   ├── ca.crt
│                   ├── server.crt
│                   └── server.key
└── peerOrganizations
    ├── org1.example.com
    │   ├── ca
    │   │   ├── ca.org1.example.com-cert.pem
    │   │   └── db8cc18ebcac9670df5ec1c1e7fcdc359d70a0b1d82adeab012eea9f5131850b_sk
    │   ├── msp
    │   │   ├── admincerts
    │   │   │   └── Admin@org1.example.com-cert.pem
    │   │   ├── cacerts
    │   │   │   └── ca.org1.example.com-cert.pem
    │   │   └── tlscacerts
    │   │       └── tlsca.org1.example.com-cert.pem
    │   ├── peers
    │   │   ├── peer0.org1.example.com
    │   │   │   ├── msp
    │   │   │   │   ├── admincerts
    │   │   │   │   │   └── Admin@org1.example.com-cert.pem
    │   │   │   │   ├── cacerts
    │   │   │   │   │   └── ca.org1.example.com-cert.pem
    │   │   │   │   ├── keystore
    │   │   │   │   │   └── 840f933728d2d1e2406768147f1c50d0f64fa0fd01901e5bb07f622462fd6ab6_sk
    │   │   │   │   ├── signcerts
    │   │   │   │   │   └── peer0.org1.example.com-cert.pem
    │   │   │   │   └── tlscacerts
    │   │   │   │       └── tlsca.org1.example.com-cert.pem
    │   │   │   └── tls
    │   │   │       ├── ca.crt
    │   │   │       ├── server.crt
    │   │   │       └── server.key
    │   │   └── peer1.org1.example.com
    │   │       ├── msp
    │   │       │   ├── admincerts
    │   │       │   │   └── Admin@org1.example.com-cert.pem
    │   │       │   ├── cacerts
    │   │       │   │   └── ca.org1.example.com-cert.pem
    │   │       │   ├── keystore
    │   │       │   │   └── 19b17095ed9a62378004727148aa5dfb70dfcd1275ce9a77d187fb84d1703971_sk
    │   │       │   ├── signcerts
    │   │       │   │   └── peer1.org1.example.com-cert.pem
    │   │       │   └── tlscacerts
    │   │       │       └── tlsca.org1.example.com-cert.pem
    │   │       └── tls
    │   │           ├── ca.crt
    │   │           ├── server.crt
    │   │           └── server.key
    │   ├── tlsca
    │   │   ├── 57dcbe74092cb46f126f1e9a58f9fd225d728a1b9b5b87383ece059476d48b40_sk
    │   │   └── tlsca.org1.example.com-cert.pem
    │   └── users
    │       ├── Admin@org1.example.com
    │       │   ├── msp
    │       │   │   ├── admincerts
    │       │   │   │   └── Admin@org1.example.com-cert.pem
    │       │   │   ├── cacerts
    │       │   │   │   └── ca.org1.example.com-cert.pem
    │       │   │   ├── keystore
    │       │   │   │   └── 2cb5be9ab061194b10d558302be57699516493dbe5b567e5ccadcd8157462601_sk
    │       │   │   ├── signcerts
    │       │   │   │   └── Admin@org1.example.com-cert.pem
    │       │   │   └── tlscacerts
    │       │   │       └── tlsca.org1.example.com-cert.pem
    │       │   └── tls
    │       │       ├── ca.crt
    │       │       ├── server.crt
    │       │       └── server.key
    │       └── User1@org1.example.com
    │           ├── msp
    │           │   ├── admincerts
    │           │   │   └── User1@org1.example.com-cert.pem
    │           │   ├── cacerts
    │           │   │   └── ca.org1.example.com-cert.pem
    │           │   ├── keystore
    │           │   │   └── 32c9696f3572dfc03eddb240ea4f0e8962a171ff86e60c42993b7f68eaaab123_sk
    │           │   ├── signcerts
    │           │   │   └── User1@org1.example.com-cert.pem
    │           │   └── tlscacerts
    │           │       └── tlsca.org1.example.com-cert.pem
    │           └── tls
    │               ├── ca.crt
    │               ├── server.crt
    │               └── server.key
    └── org2.example.com
        ├── ca
        │   ├── 8a2b52e8deda65a3a6e16c68d86aa8c4a25f0c35921f53ce2959e9f2bca956cc_sk
        │   └── ca.org2.example.com-cert.pem
        ├── msp
        │   ├── admincerts
        │   │   └── Admin@org2.example.com-cert.pem
        │   ├── cacerts
        │   │   └── ca.org2.example.com-cert.pem
        │   └── tlscacerts
        │       └── tlsca.org2.example.com-cert.pem
        ├── peers
        │   ├── peer0.org2.example.com
        │   │   ├── msp
        │   │   │   ├── admincerts
        │   │   │   │   └── Admin@org2.example.com-cert.pem
        │   │   │   ├── cacerts
        │   │   │   │   └── ca.org2.example.com-cert.pem
        │   │   │   ├── keystore
        │   │   │   │   └── 309fdb2f24b215f4cb097ade93b5b4c4dbda517fc646ddec572b3d8e24fa3b6c_sk
        │   │   │   ├── signcerts
        │   │   │   │   └── peer0.org2.example.com-cert.pem
        │   │   │   └── tlscacerts
        │   │   │       └── tlsca.org2.example.com-cert.pem
        │   │   └── tls
        │   │       ├── ca.crt
        │   │       ├── server.crt
        │   │       └── server.key
        │   └── peer1.org2.example.com
        │       ├── msp
        │       │   ├── admincerts
        │       │   │   └── Admin@org2.example.com-cert.pem
        │       │   ├── cacerts
        │       │   │   └── ca.org2.example.com-cert.pem
        │       │   ├── keystore
        │       │   │   └── 97b1fdcaab782d0f768dbd9ef867fafe246c27fd471782330cba0d0e36898516_sk
        │       │   ├── signcerts
        │       │   │   └── peer1.org2.example.com-cert.pem
        │       │   └── tlscacerts
        │       │       └── tlsca.org2.example.com-cert.pem
        │       └── tls
        │           ├── ca.crt
        │           ├── server.crt
        │           └── server.key
        ├── tlsca
        │   ├── acd26c09f5c89d891d3806c8d1d71e2c442ee9a58521d981980a1d45ab4ba666_sk
        │   └── tlsca.org2.example.com-cert.pem
        └── users
            ├── Admin@org2.example.com
            │   ├── msp
            │   │   ├── admincerts
            │   │   │   └── Admin@org2.example.com-cert.pem
            │   │   ├── cacerts
            │   │   │   └── ca.org2.example.com-cert.pem
            │   │   ├── keystore
            │   │   │   └── ea0f817adb699880824dbcefb7f88d439ee590c09e64a2f42ff39473c5cc1ea9_sk
            │   │   ├── signcerts
            │   │   │   └── Admin@org2.example.com-cert.pem
            │   │   └── tlscacerts
            │   │       └── tlsca.org2.example.com-cert.pem
            │   └── tls
            │       ├── ca.crt
            │       ├── server.crt
            │       └── server.key
            └── User1@org2.example.com
                ├── msp
                │   ├── admincerts
                │   │   └── User1@org2.example.com-cert.pem
                │   ├── cacerts
                │   │   └── ca.org2.example.com-cert.pem
                │   ├── keystore
                │   │   └── db11744d4c4a66ee04f783498d896099b275bd2cd2e2e2c188bde880041d8b1e_sk
                │   ├── signcerts
                │   │   └── User1@org2.example.com-cert.pem
                │   └── tlscacerts
                │       └── tlsca.org2.example.com-cert.pem
                └── tls
                    ├── ca.crt
                    ├── server.crt
                    └── server.key

109 directories, 101 files
```