# Fabric 1.0源代码笔记 之 Peer（7）EndorserServer（Endorser服务端）

## 1、EndorserServer概述

EndorserServer相关代码在protos/peer、core/endorser（背书策略）目录下。

* protos/peer/peer.pb.go，EndorserServer接口定义。
* core/endorser/endorser.go，EndorserServer接口实现，即Endorser结构体及方法。

## 2、EndorserServer接口定义

### 2.1、EndorserServer接口定义

```go
type EndorserServer interface {
	ProcessProposal(context.Context, *SignedProposal) (*ProposalResponse, error)
}
//代码在protos/peer/peer.pb.go
```

### 2.2、gRPC相关实现

```go
var _Endorser_serviceDesc = grpc.ServiceDesc{
	ServiceName: "protos.Endorser",
	HandlerType: (*EndorserServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "ProcessProposal",
			Handler:    _Endorser_ProcessProposal_Handler,
		},
	},
	Streams:  []grpc.StreamDesc{},
	Metadata: "peer/peer.proto",
}

func RegisterEndorserServer(s *grpc.Server, srv EndorserServer) {
	s.RegisterService(&_Endorser_serviceDesc, srv)
}

func _Endorser_ProcessProposal_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(SignedProposal)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(EndorserServer).ProcessProposal(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/protos.Endorser/ProcessProposal",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(EndorserServer).ProcessProposal(ctx, req.(*SignedProposal))
	}
	return interceptor(ctx, in, info, handler)
}
//代码在protos/peer/peer.pb.go
```

## 3、EndorserServer接口实现

### 3.1、Endorser结构体及方法

```go
type Endorser struct {
	policyChecker policy.PolicyChecker //策略检查器
}
//代码在core/endorser/endorser.go
```

涉及方法如下：

```go
//构造Endorser
func NewEndorserServer() pb.EndorserServer
//执行提案
func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error)

//检查SignedProposal是否符合通道策略，调用e.policyChecker.CheckPolicy()
func (e *Endorser) checkACL(signedProp *pb.SignedProposal, chdr *common.ChannelHeader, shdr *common.SignatureHeader, hdrext *pb.ChaincodeHeaderExtension) error
//检查Escc和Vscc，暂未实现
func (*Endorser) checkEsccAndVscc(prop *pb.Proposal) error
//获取账本的交易模拟器，调用peer.GetLedger(ledgername).NewTxSimulator()
func (*Endorser) getTxSimulator(ledgername string) (ledger.TxSimulator, error)
//获取账本历史交易查询器，调用peer.GetLedger(ledgername).NewHistoryQueryExecutor()
func (*Endorser) getHistoryQueryExecutor(ledgername string) (ledger.HistoryQueryExecutor, error)
//执行链码
func (e *Endorser) callChaincode(ctxt context.Context, chainID string, version string, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal, cis *pb.ChaincodeInvocationSpec, cid *pb.ChaincodeID, txsim ledger.TxSimulator) (*pb.Response, *pb.ChaincodeEvent, error)
func (e *Endorser) disableJavaCCInst(cid *pb.ChaincodeID, cis *pb.ChaincodeInvocationSpec) error
//通过调用chaincode来模拟提案
func (e *Endorser) simulateProposal(ctx context.Context, chainID string, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal, cid *pb.ChaincodeID, txsim ledger.TxSimulator) (*ccprovider.ChaincodeData, *pb.Response, []byte, *pb.ChaincodeEvent, error)
//从LSCC获取链码数据
func (e *Endorser) getCDSFromLSCC(ctx context.Context, chainID string, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal, chaincodeID string, txsim ledger.TxSimulator) (*ccprovider.ChaincodeData, error)
//提案背书
func (e *Endorser) endorseProposal(ctx context.Context, chainID string, txid string, signedProp *pb.SignedProposal, proposal *pb.Proposal, response *pb.Response, simRes []byte, event *pb.ChaincodeEvent, visibility []byte, ccid *pb.ChaincodeID, txsim ledger.TxSimulator, cd *ccprovider.ChaincodeData) (*pb.ProposalResponse, error)
//提交模拟交易，仅测试使用
func (e *Endorser) commitTxSimulation(proposal *pb.Proposal, chainID string, signer msp.SigningIdentity, pResp *pb.ProposalResponse, blockNumber uint64) error
//代码在core/endorser/endorser.go
```

### 3.2、func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error)方法实现

```go

//代码在core/endorser/endorser.go
```