# Fabric 1.0源代码笔记 之 Orderer #BroadcastServer（Broadcast服务端）

## 1、BroadcastServer概述

BroadcastServer相关代码在protos/orderer、orderer目录下。

protos/orderer/ab.pb.go，AtomicBroadcastServer接口定义。
orderer/server.go，go，AtomicBroadcastServer接口实现。

![](BroadcastServer_Broadcast.png)

## 2、AtomicBroadcastServer接口定义

### 2.1、AtomicBroadcastServer接口定义

```go
type AtomicBroadcastServer interface {
	Broadcast(AtomicBroadcast_BroadcastServer) error
	Deliver(AtomicBroadcast_DeliverServer) error
}
//代码在protos/orderer/ab.pb.go
···

### 2.2、gRPC相关实现

```go
var _AtomicBroadcast_serviceDesc = grpc.ServiceDesc{
	ServiceName: "orderer.AtomicBroadcast",
	HandlerType: (*AtomicBroadcastServer)(nil),
	Methods:     []grpc.MethodDesc{},
	Streams: []grpc.StreamDesc{
		{
			StreamName:    "Broadcast",
			Handler:       _AtomicBroadcast_Broadcast_Handler,
			ServerStreams: true,
			ClientStreams: true,
		},
		{
			StreamName:    "Deliver",
			Handler:       _AtomicBroadcast_Deliver_Handler,
			ServerStreams: true,
			ClientStreams: true,
		},
	},
	Metadata: "orderer/ab.proto",
}

func RegisterAtomicBroadcastServer(s *grpc.Server, srv AtomicBroadcastServer) {
	s.RegisterService(&_AtomicBroadcast_serviceDesc, srv)
}

func _AtomicBroadcast_Broadcast_Handler(srv interface{}, stream grpc.ServerStream) error {
	return srv.(AtomicBroadcastServer).Broadcast(&atomicBroadcastBroadcastServer{stream})
}

func _AtomicBroadcast_Deliver_Handler(srv interface{}, stream grpc.ServerStream) error {
	return srv.(AtomicBroadcastServer).Deliver(&atomicBroadcastDeliverServer{stream})
}
//代码在protos/orderer/ab.pb.go
```

## 3、AtomicBroadcastServer接口实现

### 3.1、server结构体

server结构体：

```go
type server struct {
	bh broadcast.Handler
	dh deliver.Handler
}

type broadcastSupport struct {
	multichain.Manager
	broadcast.ConfigUpdateProcessor
}
//代码在orderer/server.go
```

broadcast.Handler：

```go
type Handler interface {
	Handle(srv ab.AtomicBroadcast_BroadcastServer) error
}

type handlerImpl struct {
	sm SupportManager
}

func NewHandlerImpl(sm SupportManager) Handler {
	return &handlerImpl{
		sm: sm,
	}
}

type SupportManager interface {
	ConfigUpdateProcessor
	GetChain(chainID string) (Support, bool)
}

type ConfigUpdateProcessor interface { //处理通道配置更新
	Process(envConfigUpdate *cb.Envelope) (*cb.Envelope, error)
}
//代码在orderer/common/broadcast/broadcast.go
```

deliver.Handler：

```go
type Handler interface {
	Handle(srv ab.AtomicBroadcast_DeliverServer) error
}

type deliverServer struct {
	sm SupportManager
}

type SupportManager interface {
	GetChain(chainID string) (Support, bool)
}
//代码在orderer/common/deliver/deliver.go
```

### 3.2、server结构体相关方法

```go
//构建server结构体
func NewServer(ml multichain.Manager, signer crypto.LocalSigner) ab.AtomicBroadcastServer
//s.bh.Handle(srv)
func (s *server) Broadcast(srv ab.AtomicBroadcast_BroadcastServer) error
//s.dh.Handle(srv)
func (s *server) Deliver(srv ab.AtomicBroadcast_DeliverServer) error 
//代码在orderer/server.go
```

func NewServer(ml multichain.Manager, signer crypto.LocalSigner) ab.AtomicBroadcastServer代码如下：

```go
func NewServer(ml multichain.Manager, signer crypto.LocalSigner) ab.AtomicBroadcastServer {
	s := &server{
		dh: deliver.NewHandlerImpl(deliverSupport{Manager: ml}),
		bh: broadcast.NewHandlerImpl(broadcastSupport{
			Manager:               ml,
			ConfigUpdateProcessor: configupdate.New(ml.SystemChannelID(), configUpdateSupport{Manager: ml}, signer),
		}),
	}
	return s
}
//代码在orderer/server.go
```

### 3.3、Broadcast服务端Broadcast处理流程

Broadcast服务端Broadcast处理流程，即broadcast.handlerImpl.Handle方法。

#### 3.3.1、接收Envelope消息，并获取Payload和ChannelHeader

```go
msg, err := srv.Recv() //接收Envelope消息
payload, err := utils.UnmarshalPayload(msg.Payload) //反序列化获取Payload
chdr, err := utils.UnmarshalChannelHeader(payload.Header.ChannelHeader) //反序列化获取ChannelHeader
//代码在orderer/common/broadcast/broadcast.go
```

#### 3.3.2、如果消息类型为channel配置或更新，则使用multichain.Manager处理消息

```
if chdr.Type == int32(cb.HeaderType_CONFIG_UPDATE) { //如果是channel配置或更新
	msg, err = bh.sm.Process(msg) //configupdate.Processor.Process方法
}
//代码在orderer/common/broadcast/broadcast.go
```

msg, err = bh.sm.Process(msg)代码如下：

```go
func (p *Processor) Process(envConfigUpdate *cb.Envelope) (*cb.Envelope, error) {
	channelID, err := channelID(envConfigUpdate) //获取ChannelHeader.ChannelId
	//multichain.Manager.GetChain方法，获取chainSupport，以及chain是否存在
	support, ok := p.manager.GetChain(channelID)
	if ok {
		//已存在的channel配置，调取multichain.Manager.ProposeConfigUpdate方法
		return p.existingChannelConfig(envConfigUpdate, channelID, support)
	}
	//新channel配置，调取multichain.Manager.NewChannelConfig方法
	return p.newChannelConfig(channelID, envConfigUpdate)
}
//代码在orderer/configupdate/configupdate.go
```

#### 3.3.3、其他消息类型或channel消息处理后，接受消息并加入排序

```go
support, ok := bh.sm.GetChain(chdr.ChannelId) //获取chainSupport
_, filterErr := support.Filters().Apply(msg) //filter.RuleSet.Apply方法
//调取Chain.Enqueue方法，接受消息，加入排序
support.Enqueue(msg)
//代码在orderer/common/broadcast/broadcast.go
```

#### 3.3.4、向客户端发送响应信息

```go
err = srv.Send(&ab.BroadcastResponse{Status: cb.Status_SUCCESS})
//代码在orderer/common/broadcast/broadcast.go
```

### 3.4、Broadcast服务端Deliver处理流程

暂略。
