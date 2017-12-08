# Fabric 1.0源代码笔记 之 gossip（流言算法） #GossipServer（Gossip服务端）

## 1、GossipServer概述

GossipServer相关代码，分布在protos/gossip、gossip/comm目录下。目录结构如下：

* protos/gossip目录：
	* message.pb.go，GossipClient接口定义及实现，GossipServer接口定义。
* gossip/comm目录：
	* comm.go，Comm接口定义。
	* conn.go，connFactory接口定义，以及connectionStore结构体及方法。
	* comm_impl.go，commImpl结构体及方法（同时实现GossipServer接口/Comm接口/connFactory接口）。

## 2、GossipClient接口定义及实现

### 2.1、GossipClient接口定义

```go
type GossipClient interface {
	// GossipStream is the gRPC stream used for sending and receiving messages
	GossipStream(ctx context.Context, opts ...grpc.CallOption) (Gossip_GossipStreamClient, error)
	// Ping is used to probe a remote peer's aliveness
	Ping(ctx context.Context, in *Empty, opts ...grpc.CallOption) (*Empty, error)
}
//代码在protos/gossip/message.pb.go
```

### 2.2、GossipClient接口实现

```go
type gossipClient struct {
	cc *grpc.ClientConn
}

func NewGossipClient(cc *grpc.ClientConn) GossipClient {
	return &gossipClient{cc}
}

func (c *gossipClient) GossipStream(ctx context.Context, opts ...grpc.CallOption) (Gossip_GossipStreamClient, error) {
	stream, err := grpc.NewClientStream(ctx, &_Gossip_serviceDesc.Streams[0], c.cc, "/gossip.Gossip/GossipStream", opts...)
	if err != nil {
		return nil, err
	}
	x := &gossipGossipStreamClient{stream}
	return x, nil
}

func (c *gossipClient) Ping(ctx context.Context, in *Empty, opts ...grpc.CallOption) (*Empty, error) {
	out := new(Empty)
	err := grpc.Invoke(ctx, "/gossip.Gossip/Ping", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
//代码在protos/gossip/message.pb.go
```

### 2.3、Gossip_GossipStreamClient接口定义及实现

```go
type Gossip_GossipStreamClient interface {
	Send(*Envelope) error
	Recv() (*Envelope, error)
	grpc.ClientStream
}

type gossipGossipStreamClient struct {
	grpc.ClientStream
}

func (x *gossipGossipStreamClient) Send(m *Envelope) error {
	return x.ClientStream.SendMsg(m)
}

func (x *gossipGossipStreamClient) Recv() (*Envelope, error) {
	m := new(Envelope)
	if err := x.ClientStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
//代码在protos/gossip/message.pb.go
```

## 3、GossipServer接口定义

### 3.1、GossipServer接口定义

```go
type GossipServer interface {
	// GossipStream is the gRPC stream used for sending and receiving messages
	GossipStream(Gossip_GossipStreamServer) error
	// Ping is used to probe a remote peer's aliveness
	Ping(context.Context, *Empty) (*Empty, error)
}

func RegisterGossipServer(s *grpc.Server, srv GossipServer) {
	s.RegisterService(&_Gossip_serviceDesc, srv)
}

func _Gossip_GossipStream_Handler(srv interface{}, stream grpc.ServerStream) error {
	return srv.(GossipServer).GossipStream(&gossipGossipStreamServer{stream})
}

func _Gossip_Ping_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(Empty)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(GossipServer).Ping(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/gossip.Gossip/Ping",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(GossipServer).Ping(ctx, req.(*Empty))
	}
	return interceptor(ctx, in, info, handler)
}

var _Gossip_serviceDesc = grpc.ServiceDesc{
	ServiceName: "gossip.Gossip",
	HandlerType: (*GossipServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "Ping",
			Handler:    _Gossip_Ping_Handler,
		},
	},
	Streams: []grpc.StreamDesc{
		{
			StreamName:    "GossipStream",
			Handler:       _Gossip_GossipStream_Handler,
			ServerStreams: true,
			ClientStreams: true,
		},
	},
	Metadata: "gossip/message.proto",
}
//代码在protos/gossip/message.pb.go
```

### 3.2、Gossip_GossipStreamServer接口定义及实现

```go
type Gossip_GossipStreamServer interface {
	Send(*Envelope) error
	Recv() (*Envelope, error)
	grpc.ServerStream
}

type gossipGossipStreamServer struct {
	grpc.ServerStream
}

func (x *gossipGossipStreamServer) Send(m *Envelope) error {
	return x.ServerStream.SendMsg(m)
}

func (x *gossipGossipStreamServer) Recv() (*Envelope, error) {
	m := new(Envelope)
	if err := x.ServerStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
//代码在protos/gossip/message.pb.go
```

## 4、Comm接口/connFactory接口定义

### 4.1、Comm接口定义

```go
type Comm interface {
	//返回此实例的 PKI id
	GetPKIid() common.PKIidType
	//向节点发送消息
	Send(msg *proto.SignedGossipMessage, peers ...*RemotePeer)
	//探测远程节点是否有响应
	Probe(peer *RemotePeer) error
	//握手验证远程节点
	Handshake(peer *RemotePeer) (api.PeerIdentityType, error)
	Accept(common.MessageAcceptor) <-chan proto.ReceivedMessage
	//获取怀疑脱机节点的只读通道
	PresumedDead() <-chan common.PKIidType
	//关闭到某个节点的连接
	CloseConn(peer *RemotePeer)
	//关闭
	Stop()
}
//代码在gossip/comm/comm.go
```

### 4.2、connFactory接口定义

```go
type connFactory interface {
	createConnection(endpoint string, pkiID common.PKIidType) (*connection, error)
}
//代码在gossip/comm/conn.go
```

## 5、commImpl结构体及方法（同时实现GossipServer接口/Comm接口/connFactory接口）

### 5.1、commImpl结构体定义

```go
type commImpl struct {
	selfCertHash   []byte
	peerIdentity   api.PeerIdentityType
	idMapper       identity.Mapper
	logger         *logging.Logger
	opts           []grpc.DialOption
	secureDialOpts func() []grpc.DialOption
	connStore      *connectionStore
	PKIID          []byte
	deadEndpoints  chan common.PKIidType
	msgPublisher   *ChannelDeMultiplexer
	lock           *sync.RWMutex
	lsnr           net.Listener
	gSrv           *grpc.Server
	exitChan       chan struct{}
	stopWG         sync.WaitGroup
	subscriptions  []chan proto.ReceivedMessage
	port           int
	stopping       int32
}
//代码在gossip/comm/comm_impl.go
```

### 5.2、commImpl结构体方法

```go
func (c *commImpl) GossipStream(stream proto.Gossip_GossipStreamServer) error
func (c *commImpl) Ping(context.Context, *proto.Empty) (*proto.Empty, error)

func (c *commImpl) GetPKIid() common.PKIidType
func (c *commImpl) Send(msg *proto.SignedGossipMessage, peers ...*RemotePeer)
func (c *commImpl) Probe(remotePeer *RemotePeer) error
func (c *commImpl) Handshake(remotePeer *RemotePeer) (api.PeerIdentityType, error)
func (c *commImpl) Accept(acceptor common.MessageAcceptor) <-chan proto.ReceivedMessage
func (c *commImpl) PresumedDead() <-chan common.PKIidType
func (c *commImpl) CloseConn(peer *RemotePeer)
func (c *commImpl) Stop()

func SetDialTimeout(timeout time.Duration)
//创建并启动gRPC Server，以及注册GossipServer实例
func NewCommInstanceWithServer(port int, idMapper identity.Mapper, peerIdentity api.PeerIdentityType,
//将GossipServer实例注册至peerServer
func NewCommInstance(s *grpc.Server, cert *tls.Certificate, idStore identity.Mapper,
func extractRemoteAddress(stream stream) string
func readWithTimeout(stream interface{}, timeout time.Duration, address string) (*proto.SignedGossipMessage, error) 
//创建gRPC Server，grpc.NewServer(serverOpts...)
func createGRPCLayer(port int) (*grpc.Server, net.Listener, api.PeerSecureDialOpts, []byte)

func (c *commImpl) SetDialOpts(opts ...grpc.DialOption)
//创建与服务端连接
func (c *commImpl) createConnection(endpoint string, expectedPKIID common.PKIidType) (*connection, error)
func (c *commImpl) sendToEndpoint(peer *RemotePeer, msg *proto.SignedGossipMessage)
func (c *commImpl) isStopping() bool
func (c *commImpl) emptySubscriptions()
func (c *commImpl) authenticateRemotePeer(stream stream) (*proto.ConnectionInfo, error)
func (c *commImpl) disconnect(pkiID common.PKIidType)
func (c *commImpl) createConnectionMsg(pkiID common.PKIidType, certHash []byte, cert api.PeerIdentityType, signer proto.Signer) (*proto.SignedGossipMessage, error)
//代码在gossip/comm/comm_impl.go
```

#### 5.2.1、func NewCommInstanceWithServer(port int, idMapper identity.Mapper, peerIdentity api.PeerIdentityType,secureDialOpts api.PeerSecureDialOpts, dialOpts ...grpc.DialOption) (Comm, error)

创建并启动gRPC Server，以及注册GossipServer实例

```go
func NewCommInstanceWithServer(port int, idMapper identity.Mapper, peerIdentity api.PeerIdentityType,
	secureDialOpts api.PeerSecureDialOpts, dialOpts ...grpc.DialOption) (Comm, error) {

	var ll net.Listener
	var s *grpc.Server
	var certHash []byte

	if len(dialOpts) == 0 {
		//peer.gossip.dialTimeout，gRPC连接拨号的超时
		dialOpts = []grpc.DialOption{grpc.WithTimeout(util.GetDurationOrDefault("peer.gossip.dialTimeout", defDialTimeout))}
	}

	if port > 0 {
		//创建gRPC Server，grpc.NewServer(serverOpts...)
		s, ll, secureDialOpts, certHash = createGRPCLayer(port)
	}

	commInst := &commImpl{
		selfCertHash:   certHash,
		PKIID:          idMapper.GetPKIidOfCert(peerIdentity),
		idMapper:       idMapper,
		logger:         util.GetLogger(util.LoggingCommModule, fmt.Sprintf("%d", port)),
		peerIdentity:   peerIdentity,
		opts:           dialOpts,
		secureDialOpts: secureDialOpts,
		port:           port,
		lsnr:           ll,
		gSrv:           s,
		msgPublisher:   NewChannelDemultiplexer(),
		lock:           &sync.RWMutex{},
		deadEndpoints:  make(chan common.PKIidType, 100),
		stopping:       int32(0),
		exitChan:       make(chan struct{}, 1),
		subscriptions:  make([]chan proto.ReceivedMessage, 0),
	}
	commInst.connStore = newConnStore(commInst, commInst.logger)

	if port > 0 {
		commInst.stopWG.Add(1)
		go func() {
			defer commInst.stopWG.Done()
			s.Serve(ll) //启动gRPC Server
		}()
		//commInst注册到gRPC Server
		proto.RegisterGossipServer(s, commInst)
	}

	return commInst, nil
}

//代码在gossip/comm/comm_impl.go
```

#### 5.2.2、func NewCommInstance(s *grpc.Server, cert *tls.Certificate, idStore identity.Mapper,peerIdentity api.PeerIdentityType, secureDialOpts api.PeerSecureDialOpts,dialOpts ...grpc.DialOption) (Comm, error) 

将GossipServer实例注册至peerServer

```go
func NewCommInstance(s *grpc.Server, cert *tls.Certificate, idStore identity.Mapper,
	peerIdentity api.PeerIdentityType, secureDialOpts api.PeerSecureDialOpts,
	dialOpts ...grpc.DialOption) (Comm, error) {

	dialOpts = append(dialOpts, grpc.WithTimeout(util.GetDurationOrDefault("peer.gossip.dialTimeout", defDialTimeout)))
	//构造commImpl
	commInst, err := NewCommInstanceWithServer(-1, idStore, peerIdentity, secureDialOpts, dialOpts...)

	if cert != nil {
		inst := commInst.(*commImpl)
		inst.selfCertHash = certHashFromRawCert(cert.Certificate[0])
	}
	proto.RegisterGossipServer(s, commInst.(*commImpl))

	return commInst, nil
}

//代码在gossip/comm/comm_impl.go
```

//创建与服务端连接
#### 5.2.3、func (c *commImpl) createConnection(endpoint string, expectedPKIID common.PKIidType) (*connection, error)

```go
func (c *commImpl) createConnection(endpoint string, expectedPKIID common.PKIidType) (*connection, error) {
	var err error
	var cc *grpc.ClientConn
	var stream proto.Gossip_GossipStreamClient
	var pkiID common.PKIidType
	var connInfo *proto.ConnectionInfo
	var dialOpts []grpc.DialOption

	dialOpts = append(dialOpts, c.secureDialOpts()...)
	dialOpts = append(dialOpts, grpc.WithBlock())
	dialOpts = append(dialOpts, c.opts...)
	cc, err = grpc.Dial(endpoint, dialOpts...)

	cl := proto.NewGossipClient(cc)
	if _, err = cl.Ping(context.Background(), &proto.Empty{}); err != nil {
		cc.Close()
		return nil, err
	}

	ctx, cf := context.WithCancel(context.Background())
	stream, err = cl.GossipStream(ctx)
	connInfo, err = c.authenticateRemotePeer(stream)
	pkiID = connInfo.ID
	conn := newConnection(cl, cc, stream, nil)
	conn.pkiID = pkiID
	conn.info = connInfo
	conn.logger = c.logger
	conn.cancel = cf

	h := func(m *proto.SignedGossipMessage) {
		c.msgPublisher.DeMultiplex(&ReceivedMessageImpl{
			conn:                conn,
			lock:                conn,
			SignedGossipMessage: m,
			connInfo:            connInfo,
		})
	}
	conn.handler = h
	return conn, nil
}
//代码在gossip/comm/comm_impl.go
```

## 6、connectionStore和connection结构体及方法

### 6.1、connection结构体及方法

```go
type connection struct {
	cancel       context.CancelFunc
	info         *proto.ConnectionInfo
	outBuff      chan *msgSending
	logger       *logging.Logger                 // logger
	pkiID        common.PKIidType                // pkiID of the remote endpoint
	handler      handler                         // function to invoke upon a message reception
	conn         *grpc.ClientConn                // gRPC connection to remote endpoint
	cl           proto.GossipClient              // gRPC stub of remote endpoint
	clientStream proto.Gossip_GossipStreamClient // client-side stream to remote endpoint
	serverStream proto.Gossip_GossipStreamServer // server-side stream to remote endpoint
	stopFlag     int32                           // indicates whether this connection is in process of stopping
	stopChan     chan struct{}                   // a method to stop the server-side gRPC call from a different go-routine
	sync.RWMutex                                 // synchronizes access to shared variables
}

//构造connection
func newConnection(cl proto.GossipClient, c *grpc.ClientConn, cs proto.Gossip_GossipStreamClient, ss proto.Gossip_GossipStreamServer) *connection
//关闭connection
func (conn *connection) close()
//atomic.LoadInt32(&(conn.stopFlag)) == int32(1)
func (conn *connection) toDie() bool
//conn.outBuff <- m，其中m为msgSending{envelope: msg.Envelope,onErr: onErr,}
func (conn *connection) send(msg *proto.SignedGossipMessage, onErr func(error))
//go conn.readFromStream(errChan, msgChan)、go conn.writeToStream()，同时msg := <-msgChan，conn.handler(msg)
func (conn *connection) serviceConnection() error
//循环不间断从conn.outBuff取数据，然后stream.Send(m.envelope)
func (conn *connection) writeToStream()
//循环不间断envelope, err := stream.Recv()、msg, err := envelope.ToGossipMessage()、msgChan <- msg
func (conn *connection) readFromStream(errChan chan error, msgChan chan *proto.SignedGossipMessage)
//获取conn.serverStream
func (conn *connection) getStream() stream
//代码在gossip/comm/conn.go
```

### 6.2、connectionStore结构体及方法

```go
type connectionStore struct {
	logger           *logging.Logger          // logger
	isClosing        bool                     // whether this connection store is shutting down
	connFactory      connFactory              // creates a connection to remote peer
	sync.RWMutex                              // synchronize access to shared variables
	pki2Conn         map[string]*connection   // mapping between pkiID to connections
	destinationLocks map[string]*sync.RWMutex //mapping between pkiIDs and locks,
	// used to prevent concurrent connection establishment to the same remote endpoint
}

func newConnStore(connFactory connFactory, logger *logging.Logger) *connectionStore
func (cs *connectionStore) getConnection(peer *RemotePeer) (*connection, error)
func (cs *connectionStore) connNum() int
func (cs *connectionStore) closeConn(peer *RemotePeer)
func (cs *connectionStore) shutdown()
func (cs *connectionStore) onConnected(serverStream proto.Gossip_GossipStreamServer, connInfo *proto.ConnectionInfo) *connection
func (cs *connectionStore) registerConn(connInfo *proto.ConnectionInfo, serverStream proto.Gossip_GossipStreamServer) *connection
func (cs *connectionStore) closeByPKIid(pkiID common.PKIidType) 
//代码在gossip/comm/conn.go
```

