# Fabric 1.0源代码笔记 之 gossip（流言算法） #deliverclient（deliver客户端）

## 1、deliverclient概述

deliverclient代码分布在gossip/service、core/deliverservice目录下，目录结构如下：

* gossip/service目录：
	* gossip_service.go，DeliveryServiceFactory接口定义及实现。
* core/deliverservice目录：
	* deliveryclient.go，DeliverService接口定义及实现。
	* client.go，broadcastClient结构体，实现AtomicBroadcast_BroadcastClient接口。
	* requester.go，blocksRequester结构体及方法。
	* blocksprovider目录，BlocksProvider接口定义及实现。
	
## 2、DeliveryServiceFactory接口定义及实现

```go
type DeliveryServiceFactory interface {
	Service(g GossipService, endpoints []string, msc api.MessageCryptoService) (deliverclient.DeliverService, error)
}

type deliveryFactoryImpl struct {
}

// Returns an instance of delivery client
func (*deliveryFactoryImpl) Service(g GossipService, endpoints []string, mcs api.MessageCryptoService) (deliverclient.DeliverService, error) {
	return deliverclient.NewDeliverService(&deliverclient.Config{
		CryptoSvc:   mcs,
		Gossip:      g,
		Endpoints:   endpoints,
		ConnFactory: deliverclient.DefaultConnectionFactory,
		ABCFactory:  deliverclient.DefaultABCFactory,
	})
}
//代码在gossip/service/gossip_service.go
```

## 3、DeliverService接口定义及实现

### 3.1、DeliverService接口定义

用于与orderer沟通获取新区块，并发送给committer。

```go
type DeliverService interface {
	//从orderer接收通道的新块，接收完毕调用终结器
	StartDeliverForChannel(chainID string, ledgerInfo blocksprovider.LedgerInfo, finalizer func()) error
	//停止从orderer接收通道的新快
	StopDeliverForChannel(chainID string) error
	//终止所有通道Deliver
	Stop()
}
//代码在core/deliverservice/deliveryclient.go
```

### 3.2、Config结构体定义

```go
type Config struct {
	//创建grpc.ClientConn
	ConnFactory func(channelID string) func(endpoint string) (*grpc.ClientConn, error)
	//创建AtomicBroadcastClient
	ABCFactory func(*grpc.ClientConn) orderer.AtomicBroadcastClient
	//执行加密操作
	CryptoSvc api.MessageCryptoService
	//GossipServiceAdapter
	Gossip blocksprovider.GossipServiceAdapter
	//排序服务的地址和端口
	Endpoints []string
}
//代码在core/deliverservice/deliveryclient.go
```

### 3.3、DeliverService接口实现

DeliverService接口实现，即deliverServiceImpl结构体及方法。

```go
type deliverServiceImpl struct {
	conf           *Config //配置
	blockProviders map[string]blocksprovider.BlocksProvider
	lock           sync.RWMutex
	stopping       bool //是否停止
}

//构造deliverServiceImpl
func NewDeliverService(conf *Config) (DeliverService, error)
//校验Config
func (d *deliverServiceImpl) validateConfiguration() error
//从orderer接收通道的新块，接收完毕调用终结器
func (d *deliverServiceImpl) StartDeliverForChannel(chainID string, ledgerInfo blocksprovider.LedgerInfo, finalizer func()) error
//停止从orderer接收通道的新快
func (d *deliverServiceImpl) StopDeliverForChannel(chainID string) error
//所有通道调取client.Stop()
func (d *deliverServiceImpl) Stop()
//构造blocksRequester，并返回blocksRequester.client
func (d *deliverServiceImpl) newClient(chainID string, ledgerInfoProvider blocksprovider.LedgerInfo) *broadcastClient
func DefaultConnectionFactory(channelID string) func(endpoint string) (*grpc.ClientConn, error)
func DefaultABCFactory(conn *grpc.ClientConn) orderer.AtomicBroadcastClient
//代码在core/deliverservice/deliveryclient.go
```

#### 3.3.1、func (d *deliverServiceImpl) StartDeliverForChannel(chainID string, ledgerInfo blocksprovider.LedgerInfo, finalizer func()) error

```go
func (d *deliverServiceImpl) StartDeliverForChannel(chainID string, ledgerInfo blocksprovider.LedgerInfo, finalizer func()) error {
	d.lock.Lock()
	defer d.lock.Unlock()
	if d.stopping {
		return errors.New(errMsg)
	}
	if _, exist := d.blockProviders[chainID]; exist {
		errMsg := fmt.Sprintf("Delivery service - block provider already exists for %s found, can't start delivery", chainID)
		return errors.New(errMsg)
	} else {
		client := d.newClient(chainID, ledgerInfo)
		d.blockProviders[chainID] = blocksprovider.NewBlocksProvider(chainID, client, d.conf.Gossip, d.conf.CryptoSvc)
		go func() {
			d.blockProviders[chainID].DeliverBlocks()
			finalizer()
		}()
	}
	return nil
}
//代码在core/deliverservice/deliveryclient.go
```

#### 3.3.2、func (d *deliverServiceImpl) StopDeliverForChannel(chainID string) error

```go
func (d *deliverServiceImpl) StopDeliverForChannel(chainID string) error {
	d.lock.Lock()
	defer d.lock.Unlock()
	if d.stopping {
		return errors.New(errMsg)
	}
	if client, exist := d.blockProviders[chainID]; exist {
		client.Stop()
		delete(d.blockProviders, chainID)
		logger.Debug("This peer will stop pass blocks from orderer service to other peers")
	} else {
		errMsg := fmt.Sprintf("Delivery service - no block provider for %s found, can't stop delivery", chainID)
		return errors.New(errMsg)
	}
	return nil
}
//代码在core/deliverservice/deliveryclient.go
```

####3.3.3、func (d *deliverServiceImpl) newClient(chainID string, ledgerInfoProvider blocksprovider.LedgerInfo) *broadcastClient

```go
func (d *deliverServiceImpl) newClient(chainID string, ledgerInfoProvider blocksprovider.LedgerInfo) *broadcastClient {
	requester := &blocksRequester{
		chainID: chainID,
	}
	broadcastSetup := func(bd blocksprovider.BlocksDeliverer) error {
		return requester.RequestBlocks(ledgerInfoProvider)
	}
	backoffPolicy := func(attemptNum int, elapsedTime time.Duration) (time.Duration, bool) {
		if elapsedTime.Nanoseconds() > reConnectTotalTimeThreshold.Nanoseconds() {
			return 0, false
		}
		sleepIncrement := float64(time.Millisecond * 500)
		attempt := float64(attemptNum)
		return time.Duration(math.Min(math.Pow(2, attempt)*sleepIncrement, reConnectBackoffThreshold)), true
	}
	connProd := comm.NewConnectionProducer(d.conf.ConnFactory(chainID), d.conf.Endpoints)
	bClient := NewBroadcastClient(connProd, d.conf.ABCFactory, broadcastSetup, backoffPolicy)
	requester.client = bClient
	return bClient
}
//代码在core/deliverservice/deliveryclient.go
```

## 4、blocksRequester结构体及方法

```go
type blocksRequester struct {
	chainID string
	client  blocksprovider.BlocksDeliverer
}

func (b *blocksRequester) RequestBlocks(ledgerInfoProvider blocksprovider.LedgerInfo) error
func (b *blocksRequester) seekOldest() error
func (b *blocksRequester) seekLatestFromCommitter(height uint64) error
//代码在core/deliverservice/requester.go
```

### 4.1、func (b *blocksRequester) RequestBlocks(ledgerInfoProvider blocksprovider.LedgerInfo) error

```go
func (b *blocksRequester) RequestBlocks(ledgerInfoProvider blocksprovider.LedgerInfo) error {
	height, err := ledgerInfoProvider.LedgerHeight()
	if height > 0 {
		err := b.seekLatestFromCommitter(height)
	} else {
		err := b.seekOldest()
	}

	return nil
}
//代码在core/deliverservice/requester.go
```

## 5、BlocksProvider接口定义及实现

### 5.1、BlocksProvider接口定义

```go
type BlocksProvider interface {
	DeliverBlocks()
	Stop()
}
//代码在core/deliverservice/blocksprovider/blocksprovider.go
```

### 5.2、BlocksProvider接口实现

BlocksProvider接口实现，即blocksProviderImpl结构体及方法。

```go
type blocksProviderImpl struct {
	chainID string
	client streamClient
	gossip GossipServiceAdapter
	mcs api.MessageCryptoService
	done int32
	wrongStatusThreshold int
}

//构造blocksProviderImpl
func NewBlocksProvider(chainID string, client streamClient, gossip GossipServiceAdapter, mcs api.MessageCryptoService) BlocksProvider 
//从orderer获取块，并在peer分发
func (b *blocksProviderImpl) DeliverBlocks() 
func (b *blocksProviderImpl) Stop() 
func (b *blocksProviderImpl) isDone() bool 
func createGossipMsg(chainID string, payload *gossip_proto.Payload) *gossip_proto.GossipMessage 
func createPayload(seqNum uint64, marshaledBlock []byte) *gossip_proto.Payload 
//代码在core/deliverservice/blocksprovider/blocksprovider.go
```

#### 5.2.1、func (b *blocksProviderImpl) DeliverBlocks() 

```go
func (b *blocksProviderImpl) DeliverBlocks() {
	errorStatusCounter := 0
	statusCounter := 0
	defer b.client.Close()
	for !b.isDone() {
		msg, err := b.client.Recv()
		switch t := msg.Type.(type) {
		case *orderer.DeliverResponse_Status: //状态
			//状态相关
		case *orderer.DeliverResponse_Block: //如果是块，接收并分发给其他节点
			errorStatusCounter = 0
			statusCounter = 0
			seqNum := t.Block.Header.Number

			marshaledBlock, err := proto.Marshal(t.Block)
			b.mcs.VerifyBlock(gossipcommon.ChainID(b.chainID), seqNum, marshaledBlock)
			numberOfPeers := len(b.gossip.PeersOfChannel(gossipcommon.ChainID(b.chainID)))
			payload := createPayload(seqNum, marshaledBlock)
			gossipMsg := createGossipMsg(b.chainID, payload)
			b.gossip.AddPayload(b.chainID, payload)
			b.gossip.Gossip(gossipMsg) //分发给其他节点
		default:
			logger.Warningf("[%s] Received unknown: ", b.chainID, t)
			return
		}
	}
}
//代码在core/deliverservice/blocksprovider/blocksprovider.go
```