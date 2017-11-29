# Fabric 1.0源代码笔记 之 Peer（5）EndorserClient（Endorser客户端）

## 1、EndorserClient概述

EndorserClient相关代码分布如下：

* protos/peer/peer.pb.go，EndorserClient接口及实现。
* peer/common/common.go，EndorserClient相关工具函数。

## 2、EndorserClient接口定义

```go
type EndorserClient interface {
	//处理Proposal
	ProcessProposal(ctx context.Context, in *SignedProposal, opts ...grpc.CallOption) (*ProposalResponse, error)
}
//代码在protos/peer/peer.pb.go
```

## 3、EndorserClient接口实现

EndorserClient接口实现，即endorserClient结构体及方法。

```go
type endorserClient struct {
	cc *grpc.ClientConn
}

func NewEndorserClient(cc *grpc.ClientConn) EndorserClient {
	return &endorserClient{cc}
}

func (c *endorserClient) ProcessProposal(ctx context.Context, in *SignedProposal, opts ...grpc.CallOption) (*ProposalResponse, error) {
	out := new(ProposalResponse)
	err := grpc.Invoke(ctx, "/protos.Endorser/ProcessProposal", in, out, c.cc, opts...)
	return out, nil
}
//代码在protos/peer/peer.pb.go
```

## 4、EndorserClient工具函数

```go
//获取Endorser客户端
func GetEndorserClient() (pb.EndorserClient, error) {
	clientConn, err := peer.NewPeerClientConnection()
	endorserClient := pb.NewEndorserClient(clientConn)
	return endorserClient, nil
}
//代码在peer/common/common.go
```