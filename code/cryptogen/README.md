# Fabric 1.0源代码笔记 之 cryptogen（生成组织关系和身份证书）

## 1、cryptogen概述

cryptogen，用于生成组织关系和身份证书。
命令为：cryptogen generate --config=./crypto-config.yaml

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