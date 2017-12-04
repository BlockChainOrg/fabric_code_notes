# Fabric 1.0源代码笔记 之 Peer #BroadcastClient（Broadcast客户端）

## 1、BroadcastClient概述

BroadcastClient代码分布如下：

* peer/common/ordererclient.go，BroadcastClient接口定义及实现，及BroadcastClient工具函数。
* protos/orderer/ab.pb.go，AtomicBroadcastClient接口定义及实现，AtomicBroadcast_BroadcastClient接口定义及实现。

![](BroadcastServer_Broadcast.png)

## 2、BroadcastClient接口定义及实现

### 2.1、BroadcastClient工具函数

```go
//构造broadcastClient
func GetBroadcastClient(orderingEndpoint string, tlsEnabled bool, caFile string) (BroadcastClient, error)
//代码在peer/common/ordererclient.go
```

代码如下：

```go
func GetBroadcastClient(orderingEndpoint string, tlsEnabled bool, caFile string) (BroadcastClient, error) {
	var opts []grpc.DialOption
	if tlsEnabled { //检查TLS
		if caFile != "" {
			creds, err := credentials.NewClientTLSFromFile(caFile, "")
			opts = append(opts, grpc.WithTransportCredentials(creds))
		}
	} else {
		opts = append(opts, grpc.WithInsecure())
	}

	opts = append(opts, grpc.WithTimeout(3*time.Second))
	opts = append(opts, grpc.WithBlock())

	conn, err := grpc.Dial(orderingEndpoint, opts...)
	//NewAtomicBroadcastClient构造atomicBroadcastClient
	//.Broadcast构造atomicBroadcastBroadcastClient
	client, err := ab.NewAtomicBroadcastClient(conn).Broadcast(context.TODO())
	return &broadcastClient{conn: conn, client: client}, nil
}

//代码在peer/common/ordererclient.go
```

### 2.2、BroadcastClient接口定义及实现

```go
type BroadcastClient interface {
	//Send data to orderer
	Send(env *cb.Envelope) error
	Close() error
}

type broadcastClient struct {
	conn   *grpc.ClientConn
	client ab.AtomicBroadcast_BroadcastClient
}

//获取应答
func (s *broadcastClient) getAck() error {
	msg, err := s.client.Recv()
	if msg.Status != cb.Status_SUCCESS {
		//是否成功
	}
	return nil
}

//向orderer发送数据
func (s *broadcastClient) Send(env *cb.Envelope) error {
	err := s.client.Send(env) //发送cb.Envelope
	err := s.getAck() //获取应答
	return err
}

func (s *broadcastClient) Close() error { //关闭连接
	return s.conn.Close()
}

//代码在peer/common/ordererclient.go
```

## 3、AtomicBroadcastClient接口定义及实现

### 3.1、AtomicBroadcastClient工具函数

```go
//构造atomicBroadcastClient
func NewAtomicBroadcastClient(cc *grpc.ClientConn) AtomicBroadcastClient {
	return &atomicBroadcastClient{cc}
}
//代码在protos/orderer/ab.pb.go
```

### 3.2、AtomicBroadcastClient接口定义

```go
type AtomicBroadcastClient interface {
	//广播
	Broadcast(ctx context.Context, opts ...grpc.CallOption) (AtomicBroadcast_BroadcastClient, error)
	//投递
	Deliver(ctx context.Context, opts ...grpc.CallOption) (AtomicBroadcast_DeliverClient, error)
}
//代码在protos/orderer/ab.pb.go
```

### 3.3、AtomicBroadcastClient接口实现

AtomicBroadcastClient接口实现，即atomicBroadcastClient结构体及方法。

```go
type atomicBroadcastClient struct {
	cc *grpc.ClientConn
}

//构造atomicBroadcastBroadcastClient
func (c *atomicBroadcastClient) Broadcast(ctx context.Context, opts ...grpc.CallOption) (AtomicBroadcast_BroadcastClient, error) {
	stream, err := grpc.NewClientStream(ctx, &_AtomicBroadcast_serviceDesc.Streams[0], c.cc, "/orderer.AtomicBroadcast/Broadcast", opts...)
	x := &atomicBroadcastBroadcastClient{stream}
}
func (c *atomicBroadcastClient) Deliver(ctx context.Context, opts ...grpc.CallOption) (AtomicBroadcast_DeliverClient, error) {
	stream, err := grpc.NewClientStream(ctx, &_AtomicBroadcast_serviceDesc.Streams[1], c.cc, "/orderer.AtomicBroadcast/Deliver", opts...)
	x := &atomicBroadcastDeliverClient{stream}
}

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
//代码在protos/orderer/ab.pb.go
```

### 3.4、atomicBroadcastBroadcastClient

```go
type AtomicBroadcast_BroadcastClient interface {
	Send(*common.Envelope) error //发送
	Recv() (*BroadcastResponse, error) //接收
	grpc.ClientStream
}

type atomicBroadcastBroadcastClient struct {
	grpc.ClientStream
}

func (x *atomicBroadcastBroadcastClient) Send(m *common.Envelope) error {
	return x.ClientStream.SendMsg(m)
}

func (x *atomicBroadcastBroadcastClient) Recv() (*BroadcastResponse, error) {
	m := new(BroadcastResponse)
	if err := x.ClientStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
//代码在protos/orderer/ab.pb.go
```

### 3.5、atomicBroadcastDeliverClient

```go
type AtomicBroadcast_DeliverClient interface {
	Send(*common.Envelope) error //发送
	Recv() (*DeliverResponse, error) //接收
	grpc.ClientStream
}

type atomicBroadcastDeliverClient struct {
	grpc.ClientStream
}

func (x *atomicBroadcastDeliverClient) Send(m *common.Envelope) error {
	return x.ClientStream.SendMsg(m)
}

func (x *atomicBroadcastDeliverClient) Recv() (*DeliverResponse, error) {
	m := new(DeliverResponse)
	if err := x.ClientStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
//代码在protos/orderer/ab.pb.go
```