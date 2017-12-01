# Fabric 1.0源代码笔记 之 Chaincode（链码）（1）ChaincodeSupport（链码支持服务端）

## 1、ChaincodeSupport概述

ChaincodeSupport相关代码分布在protos/peer/chaincode_shim.pb.go和core/chaincode目录。

* protos/peer/chaincode_shim.pb.go，ChaincodeSupportServer接口定义。
* core/chaincode目录：
	* chaincode_support.go，ChaincodeSupportServer接口实现，即ChaincodeSupport结构体及方法，以及ChaincodeSupport服务端Register处理流程。
	* handler.go，HandleMessage接口定义及实现。

## 2、ChaincodeSupportServer接口定义

### 2.1、ChaincodeSupportServer接口定义

```go
type ChaincodeSupportServer interface {
	Register(ChaincodeSupport_RegisterServer) error
}
//代码在protos/peer/chaincode_shim.pb.go
```

### 2.2、gRPC相关实现

```go
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

func RegisterChaincodeSupportServer(s *grpc.Server, srv ChaincodeSupportServer) {
	s.RegisterService(&_ChaincodeSupport_serviceDesc, srv)
}

func _ChaincodeSupport_Register_Handler(srv interface{}, stream grpc.ServerStream) error {
	return srv.(ChaincodeSupportServer).Register(&chaincodeSupportRegisterServer{stream})
}
//代码在protos/peer/chaincode_shim.pb.go
```

## 3、ChaincodeSupportServer接口实现

### 3.1、ChaincodeSupport结构体定义

```go
type ChaincodeSupport struct {
	runningChaincodes *runningChaincodes
	peerAddress       string
	ccStartupTimeout  time.Duration
	peerNetworkID     string
	peerID            string
	peerTLSCertFile   string
	peerTLSKeyFile    string
	peerTLSSvrHostOrd string
	keepalive         time.Duration
	chaincodeLogLevel string
	shimLogLevel      string
	logFormat         string
	executetimeout    time.Duration
	userRunsCC        bool
	peerTLS           bool
}

//全局变量定义
var theChaincodeSupport *ChaincodeSupport
//获取theChaincodeSupport
func GetChain() *ChaincodeSupport
//构造ChaincodeSupport并赋值给theChaincodeSupport
func NewChaincodeSupport(getCCEndpoint func() (*pb.PeerEndpoint, error), userrunsCC bool, ccstartuptimeout time.Duration) *ChaincodeSupport
//代码在core/chaincode/chaincode_support.go
```

### 3.2、ChaincodeSupport结构体方法

```go
//调用chaincodeSupport.HandleChaincodeStream(stream.Context(), stream)
func (chaincodeSupport *ChaincodeSupport) Register(stream pb.ChaincodeSupport_RegisterServer) error
func getTxSimulator(context context.Context) ledger.TxSimulator
func getHistoryQueryExecutor(context context.Context) ledger.HistoryQueryExecutor
func GetChain() *ChaincodeSupport
func (chaincodeSupport *ChaincodeSupport) preLaunchSetup(chaincode string) chan bool
func (chaincodeSupport *ChaincodeSupport) chaincodeHasBeenLaunched(chaincode string) (*chaincodeRTEnv, bool)
func (chaincodeSupport *ChaincodeSupport) launchStarted(chaincode string) bool
func NewChaincodeSupport(getCCEndpoint func() (*pb.PeerEndpoint, error), userrunsCC bool, ccstartuptimeout time.Duration) *ChaincodeSupport
func getLogLevelFromViper(module string) string
func (d *DuplicateChaincodeHandlerError) Error() string
func newDuplicateChaincodeHandlerError(chaincodeHandler *Handler) error
func (chaincodeSupport *ChaincodeSupport) registerHandler(chaincodehandler *Handler) error
func (chaincodeSupport *ChaincodeSupport) deregisterHandler(chaincodehandler *Handler) error
func (chaincodeSupport *ChaincodeSupport) sendReady(context context.Context, cccid *ccprovider.CCContext, timeout time.Duration) error
func (chaincodeSupport *ChaincodeSupport) getArgsAndEnv(cccid *ccprovider.CCContext, cLang pb.ChaincodeSpec_Type) (args []string, envs []string, err error)
func (chaincodeSupport *ChaincodeSupport) launchAndWaitForRegister(ctxt context.Context, cccid *ccprovider.CCContext, cds *pb.ChaincodeDeploymentSpec, cLang pb.ChaincodeSpec_Type, builder api.BuildSpecFactory) error
func (chaincodeSupport *ChaincodeSupport) Stop(context context.Context, cccid *ccprovider.CCContext, cds *pb.ChaincodeDeploymentSpec) error
func (chaincodeSupport *ChaincodeSupport) Launch(context context.Context, cccid *ccprovider.CCContext, spec interface{}) (*pb.ChaincodeID, *pb.ChaincodeInput, error)
func (chaincodeSupport *ChaincodeSupport) getVMType(cds *pb.ChaincodeDeploymentSpec) (string, error)
//调用HandleChaincodeStream(chaincodeSupport, ctxt, stream)
func (chaincodeSupport *ChaincodeSupport) HandleChaincodeStream(ctxt context.Context, stream ccintf.ChaincodeStream) error
func createCCMessage(typ pb.ChaincodeMessage_Type, txid string, cMsg *pb.ChaincodeInput) (*pb.ChaincodeMessage, error)
func (chaincodeSupport *ChaincodeSupport) Execute(ctxt context.Context, cccid *ccprovider.CCContext, msg *pb.ChaincodeMessage, timeout time.Duration) (*pb.ChaincodeMessage, error)
func IsDevMode() bool
//代码在core/chaincode/chaincode_support.go
```

### 3.3、ChaincodeSupport服务端Register处理流程

#### 3.3.1、构造Handler

```go
//构造Handler
handler := newChaincodeSupportHandler(chaincodeSupport, stream)
//代码在core/chaincode/handler.go
```

handler := newChaincodeSupportHandler(chaincodeSupport, stream)代码如下：
此处使用了状态机包，github.com/looplab/fsm。

```go
func newChaincodeSupportHandler(chaincodeSupport *ChaincodeSupport, peerChatStream ccintf.ChaincodeStream) *Handler {
	v := &Handler{
		ChatStream: peerChatStream,
	}
	v.chaincodeSupport = chaincodeSupport
	v.nextState = make(chan *nextStateInfo)
	v.FSM = fsm.NewFSM(
		createdstate,
		fsm.Events{ //有限状态机
			{Name: pb.ChaincodeMessage_REGISTER.String(), Src: []string{createdstate}, Dst: establishedstate},
			{Name: pb.ChaincodeMessage_READY.String(), Src: []string{establishedstate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_PUT_STATE.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_DEL_STATE.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_INVOKE_CHAINCODE.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_COMPLETED.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_GET_STATE.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_GET_STATE_BY_RANGE.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_GET_QUERY_RESULT.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_GET_HISTORY_FOR_KEY.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_QUERY_STATE_NEXT.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_QUERY_STATE_CLOSE.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_ERROR.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_RESPONSE.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_INIT.String(), Src: []string{readystate}, Dst: readystate},
			{Name: pb.ChaincodeMessage_TRANSACTION.String(), Src: []string{readystate}, Dst: readystate},
		},
		fsm.Callbacks{
			"before_" + pb.ChaincodeMessage_REGISTER.String():           func(e *fsm.Event) { v.beforeRegisterEvent(e, v.FSM.Current()) },
			"before_" + pb.ChaincodeMessage_COMPLETED.String():          func(e *fsm.Event) { v.beforeCompletedEvent(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_GET_STATE.String():           func(e *fsm.Event) { v.afterGetState(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_GET_STATE_BY_RANGE.String():  func(e *fsm.Event) { v.afterGetStateByRange(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_GET_QUERY_RESULT.String():    func(e *fsm.Event) { v.afterGetQueryResult(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_GET_HISTORY_FOR_KEY.String(): func(e *fsm.Event) { v.afterGetHistoryForKey(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_QUERY_STATE_NEXT.String():    func(e *fsm.Event) { v.afterQueryStateNext(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_QUERY_STATE_CLOSE.String():   func(e *fsm.Event) { v.afterQueryStateClose(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_PUT_STATE.String():           func(e *fsm.Event) { v.enterBusyState(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_DEL_STATE.String():           func(e *fsm.Event) { v.enterBusyState(e, v.FSM.Current()) },
			"after_" + pb.ChaincodeMessage_INVOKE_CHAINCODE.String():    func(e *fsm.Event) { v.enterBusyState(e, v.FSM.Current()) },
			"enter_" + establishedstate:                                 func(e *fsm.Event) { v.enterEstablishedState(e, v.FSM.Current()) },
			"enter_" + readystate:                                       func(e *fsm.Event) { v.enterReadyState(e, v.FSM.Current()) },
			"enter_" + endstate:                                         func(e *fsm.Event) { v.enterEndState(e, v.FSM.Current()) },
		},
	)

	v.policyChecker = policy.NewPolicyChecker(
		peer.NewChannelPolicyManagerGetter(),
		mgmt.GetLocalMSP(),
		mgmt.NewLocalMSPPrincipalGetter(),
	)

	return v
}
//代码在core/chaincode/handler.go
```

#### 3.3.2、



## 4、HandleMessage接口定义及实现

### 4.1、HandleMessage接口定义

```go
type MessageHandler interface {
	//处理ChaincodeMessage
	HandleMessage(msg *pb.ChaincodeMessage) error
	//发送ChaincodeMessage
	SendMessage(msg *pb.ChaincodeMessage) error
}
//代码在core/chaincode/handler.go
```

### 4.2、HandleMessage接口实现

HandleMessage接口实现，即结构体Handler，定义如下：

```go
type Handler struct {
	sync.RWMutex
	serialLock  sync.Mutex
	ChatStream  ccintf.ChaincodeStream
	FSM         *fsm.FSM
	ChaincodeID *pb.ChaincodeID
	ccInstance  *sysccprovider.ChaincodeInstance
	chaincodeSupport *ChaincodeSupport
	registered       bool
	readyNotify      chan bool
	txCtxs map[string]*transactionContext
	txidMap map[string]bool
	nextState chan *nextStateInfo
	policyChecker policy.PolicyChecker
}
//代码在core/chaincode/handler.go
```

涉及方法如下：

```go
func shorttxid(txid string) string
func (handler *Handler) decomposeRegisteredName(cid *pb.ChaincodeID)
func getChaincodeInstance(ccName string) *sysccprovider.ChaincodeInstance
func (handler *Handler) getCCRootName() string
func (handler *Handler) serialSend(msg *pb.ChaincodeMessage) error
func (handler *Handler) serialSendAsync(msg *pb.ChaincodeMessage, errc chan error)
func (handler *Handler) createTxContext(ctxt context.Context, chainID string, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal) (*transactionContext, error)
func (handler *Handler) getTxContext(txid string) *transactionContext
func (handler *Handler) deleteTxContext(txid string)
func (handler *Handler) putQueryIterator(txContext *transactionContext, txid string,
func (handler *Handler) getQueryIterator(txContext *transactionContext, txid string) commonledger.ResultsIterator
func (handler *Handler) deleteQueryIterator(txContext *transactionContext, txid string)
func (handler *Handler) checkACL(signedProp *pb.SignedProposal, proposal *pb.Proposal, ccIns *sysccprovider.ChaincodeInstance) error
func (handler *Handler) deregister() error
func (handler *Handler) triggerNextState(msg *pb.ChaincodeMessage, send bool)
func (handler *Handler) triggerNextStateSync(msg *pb.ChaincodeMessage)
func (handler *Handler) waitForKeepaliveTimer() <-chan time.Time
func (handler *Handler) processStream() error
func HandleChaincodeStream(chaincodeSupport *ChaincodeSupport, ctxt context.Context, stream ccintf.ChaincodeStream) error
//构造Handler
func newChaincodeSupportHandler(chaincodeSupport *ChaincodeSupport, peerChatStream ccintf.ChaincodeStream) *Handler
func (handler *Handler) createTXIDEntry(txid string) bool
func (handler *Handler) deleteTXIDEntry(txid string)
func (handler *Handler) notifyDuringStartup(val bool)
func (handler *Handler) beforeRegisterEvent(e *fsm.Event, state string)
func (handler *Handler) notify(msg *pb.ChaincodeMessage)
func (handler *Handler) beforeCompletedEvent(e *fsm.Event, state string)
func (handler *Handler) afterGetState(e *fsm.Event, state string)
func (handler *Handler) isValidTxSim(txid string, fmtStr string, args ...interface{}) (*transactionContext, *pb.ChaincodeMessage)
func (handler *Handler) handleGetState(msg *pb.ChaincodeMessage)
func (handler *Handler) afterGetStateByRange(e *fsm.Event, state string)
func (handler *Handler) handleGetStateByRange(msg *pb.ChaincodeMessage)
func getQueryResponse(handler *Handler, txContext *transactionContext, iter commonledger.ResultsIterator,
func (handler *Handler) afterQueryStateNext(e *fsm.Event, state string)
func (handler *Handler) handleQueryStateNext(msg *pb.ChaincodeMessage)
func (handler *Handler) afterQueryStateClose(e *fsm.Event, state string)
func (handler *Handler) handleQueryStateClose(msg *pb.ChaincodeMessage)
func (handler *Handler) afterGetQueryResult(e *fsm.Event, state string)
func (handler *Handler) handleGetQueryResult(msg *pb.ChaincodeMessage)
func (handler *Handler) afterGetHistoryForKey(e *fsm.Event, state string)
func (handler *Handler) handleGetHistoryForKey(msg *pb.ChaincodeMessage)
func (handler *Handler) enterBusyState(e *fsm.Event, state string)
func (handler *Handler) enterEstablishedState(e *fsm.Event, state string)
func (handler *Handler) enterReadyState(e *fsm.Event, state string)
func (handler *Handler) enterEndState(e *fsm.Event, state string)
func (handler *Handler) setChaincodeProposal(signedProp *pb.SignedProposal, prop *pb.Proposal, msg *pb.ChaincodeMessage) error
func (handler *Handler) ready(ctxt context.Context, chainID string, txid string, signedProp *pb.SignedProposal, prop *pb.Proposal) (chan *pb.ChaincodeMessage, error)
func (handler *Handler) HandleMessage(msg *pb.ChaincodeMessage) error
func filterError(errFromFSMEvent error) error
func (handler *Handler) sendExecuteMessage(ctxt context.Context, chainID string, msg *pb.ChaincodeMessage, signedProp *pb.SignedProposal, prop *pb.Proposal) (chan *pb.ChaincodeMessage, error)
func (handler *Handler) isRunning() bool
```