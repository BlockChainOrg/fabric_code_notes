# Fabric 1.0源代码笔记 之 scc（系统链码） #cscc（通道相关）

## 1、cscc概述

cscc代码在core/scc/cscc/configure.go。

## 2、PeerConfiger结构体

```go
type PeerConfiger struct {
	policyChecker policy.PolicyChecker
}
//代码在core/scc/cscc/configure.go
```

## 3、Init方法

```go
func (e *PeerConfiger) Init(stub shim.ChaincodeStubInterface) pb.Response {
	//初始化策略检查器，用于访问控制
	e.policyChecker = policy.NewPolicyChecker(
		peer.NewChannelPolicyManagerGetter(),
		mgmt.GetLocalMSP(),
		mgmt.NewLocalMSPPrincipalGetter(),
	)
	return shim.Success(nil)
}
//代码在core/scc/cscc/configure.go
```

## 4、Invoke方法

```go
func (e *PeerConfiger) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
	args := stub.GetArgs()
	fname := string(args[0]) //Invoke function
	sp, err := stub.GetSignedProposal() //获取SignedProposal

	switch fname {
	case JoinChain: //加入通道
		block, err := utils.GetBlockFromBlockBytes(args[1])
		cid, err := utils.GetChainIDFromBlock(block)
		err := validateConfigBlock(block)
		err = e.policyChecker.CheckPolicyNoChannel(mgmt.Admins, sp)
		return joinChain(cid, block)
	case GetConfigBlock:
		err = e.policyChecker.CheckPolicy(string(args[1]), policies.ChannelApplicationReaders, sp)
		return getConfigBlock(args[1])
	case GetChannels:
		err = e.policyChecker.CheckPolicyNoChannel(mgmt.Members, sp)
		return getChannels()

	}
}
//代码在core/scc/cscc/configure.go
```

## 5、其他方法

```go
func validateConfigBlock(block *common.Block) error
func joinChain(chainID string, block *common.Block) pb.Response
func getConfigBlock(chainID []byte) pb.Response
func getChannels() pb.Response
//代码在core/scc/cscc/configure.go
```
