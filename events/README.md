# Fabric 1.0源代码笔记 之 events（事件服务）

## 1、events概述

events代码分布在events/producer和events/consumer目录下，目录结构如下：

* events/producer目录：生产者，提供事件服务器。
	* producer.go，EventsServer结构体及方法。
	* events.go，eventProcessor结构体及方法，handlerList接口及实现。
	* handler.go，handler结构体及方法。
* events/consumer目录：消费者，获取事件。

## 2、producer（生产者）

### 2.1、EventsServer结构体及方法

EventsServer结构体定义：（空）

```go
type EventsServer struct {
}

//全局事件服务器globalEventsServer
var globalEventsServer *EventsServer
//代码在events/producer/producer.go
```

涉及方法如下：

```go
//globalEventsServer创建并初始化
func NewEventsServer(bufferSize uint, timeout time.Duration) *EventsServer
//接收消息并处理
func (p *EventsServer) Chat(stream pb.Events_ChatServer) error
//代码在events/producer/producer.go
```

### 2.2、eventProcessor结构体及方法（事件处理器）

eventProcessor结构体定义：

```go
type eventProcessor struct {
	sync.RWMutex
	eventConsumers map[pb.EventType]handlerList
	eventChannel chan *pb.Event
	timeout time.Duration //生产者发送事件的超时时间
}

//全局事件处理器gEventProcessor
var gEventProcessor *eventProcessor
//代码在events/producer/events.go
```

涉及方法如下：

```go
func (ep *eventProcessor) start() //启动并处理事件，从ep.eventChannel通道接收消息并处理
//初始化gEventProcessor，并添加消息类型，并发启动start()
func initializeEvents(bufferSize uint, tout time.Duration) 
//添加消息类型至gEventProcessor.eventConsumers，并初始化handlerList
func AddEventType(eventType pb.EventType) error 
//绑定Interest和handler
func registerHandler(ie *pb.Interest, h *handler) error 
//取消绑定Interest和handler
func deRegisterHandler(ie *pb.Interest, h *handler) error 
func Send(e *pb.Event) error //Event发送至gEventProcessor.eventChannel
//代码在events/producer/events.go
```

### 2.3、handlerList接口及实现

#### 2.3.1、handlerList接口定义（handler列表）

```go
type handlerList interface {
	add(ie *pb.Interest, h *handler) (bool, error) //添加
	del(ie *pb.Interest, h *handler) (bool, error) //删除
	foreach(ie *pb.Event, action func(h *handler)) //遍历
}
//代码在events/producer/events.go
```

#### 2.3.2、handlerList接口实现

```go
type genericHandlerList struct {
	sync.RWMutex
	handlers map[*handler]bool
}

type chaincodeHandlerList struct {
	sync.RWMutex
	handlers map[string]map[string]map[*handler]bool
}
//代码在events/producer/events.go
```

### 2.4、handler结构体及方法

handler结构体定义：

```go
type handler struct {
	ChatStream       pb.Events_ChatServer
	interestedEvents map[string]*pb.Interest
}
//代码在events/producer/handler.go
```

补充pb.Events_ChatServer和pb.Interest（关注）定义：

```go
type Events_ChatServer interface {
	Send(*Event) error
	Recv() (*SignedEvent, error)
	grpc.ServerStream //type ServerStream interface
}

Interest struct {
	EventType EventType //type EventType int32
	RegInfo isInterest_RegInfo //type isInterest_RegInfo interface
	ChainID string
}

type EventType int32

const (
	EventType_REGISTER  EventType = 0
	EventType_BLOCK     EventType = 1
	EventType_CHAINCODE EventType = 2
	EventType_REJECTION EventType = 3
)
//代码在protos/peer/events.pb.go
```

handler结构体方法如下：

```go
func newEventHandler(stream pb.Events_ChatServer) (*handler, error) //构造handler
func (d *handler) Stop() error //停止handler，取消所有handler注册
func getInterestKey(interest pb.Interest) string //获取interest.EventType
func (d *handler) register(iMsg []*pb.Interest) error //逐一绑定Interest和handler
func (d *handler) deregister(iMsg []*pb.Interest) error //逐一取消绑定Interest和handler
func (d *handler) deregisterAll() //取消所有handler绑定
func (d *handler) HandleMessage(msg *pb.SignedEvent) error //处理消息
func (d *handler) SendMessage(msg *pb.Event) error //通过流向远程PEER发送消息
func validateEventMessage(signedEvt *pb.SignedEvent) (*pb.Event, error) //验证事件消息
//代码在events/producer/handler.go
```

补充pb.Event和pb.SignedEvent：

```go
type Event struct {
	Event isEvent_Event //type isEvent_Event interface
	Creator []byte //事件创建者
}

SignedEvent struct {
	Signature []byte //在事件字节上的签名
	EventBytes []byte //事件对象序列化，即type Event struct
}
//代码在protos/peer/events.pb.go
```

## 3、consumer（消费者）