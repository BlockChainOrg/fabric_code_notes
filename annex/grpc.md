# Fabric 1.0源代码笔记 之 附录-gRPC（Fabric中注册的gRPC Service）

Peer节点中注册的gRPC Service，包括：

* Events Service（事件服务）：Chat
* Admin Service（管理服务）：GetStatus、StartServer、GetModuleLogLevel、SetModuleLogLevel、RevertLogLevels
* Endorser Service（背书服务）：ProcessProposal
* ChaincodeSupport Service（链码支持服务）：Register

Orderer节点中注册的gRPC Service，包括：

* AtomicBroadcast Service（广播服务）：Broadcast、Deliver

## 1、Peer节点中注册的gRPC Service

### 1.1、Events Service（事件服务）

#### 1.1.1、Events Service客户端

```go
type EventsClient interface {
	// event chatting using Event
	Chat(ctx context.Context, opts ...grpc.CallOption) (Events_ChatClient, error)
}

type eventsClient struct {
	cc *grpc.ClientConn
}

func NewEventsClient(cc *grpc.ClientConn) EventsClient {
	return &eventsClient{cc}
}

func (c *eventsClient) Chat(ctx context.Context, opts ...grpc.CallOption) (Events_ChatClient, error) {
	stream, err := grpc.NewClientStream(ctx, &_Events_serviceDesc.Streams[0], c.cc, "/protos.Events/Chat", opts...)
	if err != nil {
		return nil, err
	}
	x := &eventsChatClient{stream}
	return x, nil
}
//代码在protos/peer/events.pb.go
```

#### 1.1.2、Events Service服务端

```go
type EventsServer interface {
	Chat(Events_ChatServer) error
}

func RegisterEventsServer(s *grpc.Server, srv EventsServer) {
	s.RegisterService(&_Events_serviceDesc, srv)
}

func _Events_Chat_Handler(srv interface{}, stream grpc.ServerStream) error {
	return srv.(EventsServer).Chat(&eventsChatServer{stream})
}

var _Events_serviceDesc = grpc.ServiceDesc{
	ServiceName: "protos.Events",
	HandlerType: (*EventsServer)(nil),
	Methods:     []grpc.MethodDesc{},
	Streams: []grpc.StreamDesc{
		{
			StreamName:    "Chat",
			Handler:       _Events_Chat_Handler,
			ServerStreams: true,
			ClientStreams: true,
		},
	},
	Metadata: "peer/events.proto",
}
//代码在protos/peer/events.pb.go
```

### 1.2、Admin Service（管理服务）

#### 1.2.1、Admin Service客户端

```go
type AdminClient interface {
	// Return the serve status.
	GetStatus(ctx context.Context, in *google_protobuf.Empty, opts ...grpc.CallOption) (*ServerStatus, error)
	StartServer(ctx context.Context, in *google_protobuf.Empty, opts ...grpc.CallOption) (*ServerStatus, error)
	GetModuleLogLevel(ctx context.Context, in *LogLevelRequest, opts ...grpc.CallOption) (*LogLevelResponse, error)
	SetModuleLogLevel(ctx context.Context, in *LogLevelRequest, opts ...grpc.CallOption) (*LogLevelResponse, error)
	RevertLogLevels(ctx context.Context, in *google_protobuf.Empty, opts ...grpc.CallOption) (*google_protobuf.Empty, error)
}

type adminClient struct {
	cc *grpc.ClientConn
}

func NewAdminClient(cc *grpc.ClientConn) AdminClient {
	return &adminClient{cc}
}

func (c *adminClient) GetStatus(ctx context.Context, in *google_protobuf.Empty, opts ...grpc.CallOption) (*ServerStatus, error) {
	out := new(ServerStatus)
	err := grpc.Invoke(ctx, "/protos.Admin/GetStatus", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

func (c *adminClient) StartServer(ctx context.Context, in *google_protobuf.Empty, opts ...grpc.CallOption) (*ServerStatus, error) {
	out := new(ServerStatus)
	err := grpc.Invoke(ctx, "/protos.Admin/StartServer", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

func (c *adminClient) GetModuleLogLevel(ctx context.Context, in *LogLevelRequest, opts ...grpc.CallOption) (*LogLevelResponse, error) {
	out := new(LogLevelResponse)
	err := grpc.Invoke(ctx, "/protos.Admin/GetModuleLogLevel", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

func (c *adminClient) SetModuleLogLevel(ctx context.Context, in *LogLevelRequest, opts ...grpc.CallOption) (*LogLevelResponse, error) {
	out := new(LogLevelResponse)
	err := grpc.Invoke(ctx, "/protos.Admin/SetModuleLogLevel", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

func (c *adminClient) RevertLogLevels(ctx context.Context, in *google_protobuf.Empty, opts ...grpc.CallOption) (*google_protobuf.Empty, error) {
	out := new(google_protobuf.Empty)
	err := grpc.Invoke(ctx, "/protos.Admin/RevertLogLevels", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
//代码在protos/peer/admin.pb.go
```

#### 1.2.2、Admin Service服务端

```go
type AdminServer interface {
	GetStatus(context.Context, *google_protobuf.Empty) (*ServerStatus, error)
	StartServer(context.Context, *google_protobuf.Empty) (*ServerStatus, error)
	GetModuleLogLevel(context.Context, *LogLevelRequest) (*LogLevelResponse, error)
	SetModuleLogLevel(context.Context, *LogLevelRequest) (*LogLevelResponse, error)
	RevertLogLevels(context.Context, *google_protobuf.Empty) (*google_protobuf.Empty, error)
}

func RegisterAdminServer(s *grpc.Server, srv AdminServer) {
	s.RegisterService(&_Admin_serviceDesc, srv)
}

func _Admin_GetStatus_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(google_protobuf.Empty)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(AdminServer).GetStatus(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/protos.Admin/GetStatus",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(AdminServer).GetStatus(ctx, req.(*google_protobuf.Empty))
	}
	return interceptor(ctx, in, info, handler)
}

func _Admin_StartServer_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(google_protobuf.Empty)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(AdminServer).StartServer(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/protos.Admin/StartServer",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(AdminServer).StartServer(ctx, req.(*google_protobuf.Empty))
	}
	return interceptor(ctx, in, info, handler)
}

func _Admin_GetModuleLogLevel_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(LogLevelRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(AdminServer).GetModuleLogLevel(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/protos.Admin/GetModuleLogLevel",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(AdminServer).GetModuleLogLevel(ctx, req.(*LogLevelRequest))
	}
	return interceptor(ctx, in, info, handler)
}

func _Admin_SetModuleLogLevel_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(LogLevelRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(AdminServer).SetModuleLogLevel(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/protos.Admin/SetModuleLogLevel",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(AdminServer).SetModuleLogLevel(ctx, req.(*LogLevelRequest))
	}
	return interceptor(ctx, in, info, handler)
}

func _Admin_RevertLogLevels_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(google_protobuf.Empty)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(AdminServer).RevertLogLevels(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/protos.Admin/RevertLogLevels",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(AdminServer).RevertLogLevels(ctx, req.(*google_protobuf.Empty))
	}
	return interceptor(ctx, in, info, handler)
}

var _Admin_serviceDesc = grpc.ServiceDesc{
	ServiceName: "protos.Admin",
	HandlerType: (*AdminServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "GetStatus",
			Handler:    _Admin_GetStatus_Handler,
		},
		{
			MethodName: "StartServer",
			Handler:    _Admin_StartServer_Handler,
		},
		{
			MethodName: "GetModuleLogLevel",
			Handler:    _Admin_GetModuleLogLevel_Handler,
		},
		{
			MethodName: "SetModuleLogLevel",
			Handler:    _Admin_SetModuleLogLevel_Handler,
		},
		{
			MethodName: "RevertLogLevels",
			Handler:    _Admin_RevertLogLevels_Handler,
		},
	},
	Streams:  []grpc.StreamDesc{},
	Metadata: "peer/admin.proto",
}
//代码在protos/peer/admin.pb.go
```

### 1.3、Endorser Service（背书服务）

#### 1.3.1、Endorser Service客户端

```go
type EndorserClient interface {
	ProcessProposal(ctx context.Context, in *SignedProposal, opts ...grpc.CallOption) (*ProposalResponse, error)
}

type endorserClient struct {
	cc *grpc.ClientConn
}

func NewEndorserClient(cc *grpc.ClientConn) EndorserClient {
	return &endorserClient{cc}
}

func (c *endorserClient) ProcessProposal(ctx context.Context, in *SignedProposal, opts ...grpc.CallOption) (*ProposalResponse, error) {
	out := new(ProposalResponse)
	err := grpc.Invoke(ctx, "/protos.Endorser/ProcessProposal", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
//代码在protos/peer/peer.pb.go
```

#### 1.3.2、Endorser Service服务端

```go
type EndorserServer interface {
	ProcessProposal(context.Context, *SignedProposal) (*ProposalResponse, error)
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
//代码在protos/peer/peer.pb.go
```

### 1.4、ChaincodeSupport Service（链码支持服务）

#### 1.4.1、ChaincodeSupport Service客户端

```go
type ChaincodeSupportClient interface {
	Register(ctx context.Context, opts ...grpc.CallOption) (ChaincodeSupport_RegisterClient, error)
}

type chaincodeSupportClient struct {
	cc *grpc.ClientConn
}

func NewChaincodeSupportClient(cc *grpc.ClientConn) ChaincodeSupportClient {
	return &chaincodeSupportClient{cc}
}

func (c *chaincodeSupportClient) Register(ctx context.Context, opts ...grpc.CallOption) (ChaincodeSupport_RegisterClient, error) {
	stream, err := grpc.NewClientStream(ctx, &_ChaincodeSupport_serviceDesc.Streams[0], c.cc, "/protos.ChaincodeSupport/Register", opts...)
	if err != nil {
		return nil, err
	}
	x := &chaincodeSupportRegisterClient{stream}
	return x, nil
}
//代码在protos/peer/peer.pb.go
```

#### 1.4.2、ChaincodeSupport Service服务端

```go
type ChaincodeSupportServer interface {
	Register(ChaincodeSupport_RegisterServer) error
}

func RegisterChaincodeSupportServer(s *grpc.Server, srv ChaincodeSupportServer) {
	s.RegisterService(&_ChaincodeSupport_serviceDesc, srv)
}

func _ChaincodeSupport_Register_Handler(srv interface{}, stream grpc.ServerStream) error {
	return srv.(ChaincodeSupportServer).Register(&chaincodeSupportRegisterServer{stream})
}

var _ChaincodeSupport_serviceDesc = grpc.ServiceDesc{
	ServiceName: "protos.ChaincodeSupport",
	HandlerType: (*ChaincodeSupportServer)(nil),
	Methods:     []grpc.MethodDesc{},
	Streams: []grpc.StreamDesc{
		{
			StreamName:    "Register",
			Handler:       _ChaincodeSupport_Register_Handler,
			ServerStreams: true,
			ClientStreams: true,
		},
	},
	Metadata: "peer/chaincode_shim.proto",
}
//代码在protos/peer/peer.pb.go
```

## 2、Orderer节点中注册的gRPC Service

### 2.1、AtomicBroadcast Service（广播服务）

#### 2.1.1、AtomicBroadcast Service客户端

```go
type AtomicBroadcastClient interface {
	// broadcast receives a reply of Acknowledgement for each common.Envelope in order, indicating success or type of failure
	Broadcast(ctx context.Context, opts ...grpc.CallOption) (AtomicBroadcast_BroadcastClient, error)
	// deliver first requires an Envelope of type DELIVER_SEEK_INFO with Payload data as a mashaled SeekInfo message, then a stream of block replies is received.
	Deliver(ctx context.Context, opts ...grpc.CallOption) (AtomicBroadcast_DeliverClient, error)
}

type atomicBroadcastClient struct {
	cc *grpc.ClientConn
}

func NewAtomicBroadcastClient(cc *grpc.ClientConn) AtomicBroadcastClient {
	return &atomicBroadcastClient{cc}
}

func (c *atomicBroadcastClient) Broadcast(ctx context.Context, opts ...grpc.CallOption) (AtomicBroadcast_BroadcastClient, error) {
	stream, err := grpc.NewClientStream(ctx, &_AtomicBroadcast_serviceDesc.Streams[0], c.cc, "/orderer.AtomicBroadcast/Broadcast", opts...)
	if err != nil {
		return nil, err
	}
	x := &atomicBroadcastBroadcastClient{stream}
	return x, nil
}

func (c *atomicBroadcastClient) Deliver(ctx context.Context, opts ...grpc.CallOption) (AtomicBroadcast_DeliverClient, error) {
	stream, err := grpc.NewClientStream(ctx, &_AtomicBroadcast_serviceDesc.Streams[1], c.cc, "/orderer.AtomicBroadcast/Deliver", opts...)
	if err != nil {
		return nil, err
	}
	x := &atomicBroadcastDeliverClient{stream}
	return x, nil
}
//代码在protos/orderer/ab.pb.go
```

#### 2.1.2、AtomicBroadcast Service服务端

```go
type AtomicBroadcastServer interface {
	// broadcast receives a reply of Acknowledgement for each common.Envelope in order, indicating success or type of failure
	Broadcast(AtomicBroadcast_BroadcastServer) error
	// deliver first requires an Envelope of type DELIVER_SEEK_INFO with Payload data as a mashaled SeekInfo message, then a stream of block replies is received.
	Deliver(AtomicBroadcast_DeliverServer) error
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