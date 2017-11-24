# Fabric 1.0源代码笔记 之 events（事件服务）

## 1、events概述

events代码分布在events/producer和events/consumer目录下，目录结构如下：

* events/producer目录：生产者，提供事件服务器。
	* producer.go
	* events.go
	* handler.go
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
func (ep *eventProcessor) start() 
func initializeEvents(bufferSize uint, tout time.Duration) 
func AddEventType(eventType pb.EventType) error 
func registerHandler(ie *pb.Interest, h *handler) error 
func deRegisterHandler(ie *pb.Interest, h *handler) error 
func Send(e *pb.Event) error 
//代码在events/producer/events.go
```

### 2.3、handlerList接口及实现

#### 2.3.1、handlerList接口定义（handler列表）

```go
type handlerList interface {
	add(ie *pb.Interest, h *handler) (bool, error)
	del(ie *pb.Interest, h *handler) (bool, error)
	foreach(ie *pb.Event, action func(h *handler))
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
```

## 3、consumer（消费者）