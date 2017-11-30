# Fabric 1.0源代码笔记 之 Peer（7）EndorserServer（Endorser服务端）

## 1、EndorserServer概述

EndorserServer相关代码在core/endorser、core/policy（背书策略）目录下。

* endorser.go，Endorser结构体及方法。

## 2、Endorser结构体定义

```go
type Endorser struct {
	policyChecker policy.PolicyChecker
}
//代码在core/endorser/endorser.go
```