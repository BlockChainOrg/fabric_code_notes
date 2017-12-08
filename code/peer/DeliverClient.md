# Fabric 1.0源代码笔记 之 Peer #DeliverClient（Deliver客户端）

## 1、DeliverClient概述

DeliverClient代码分布如下：

* peer/channel/deliverclient.go，deliverClientIntf接口定义及实现，以及DeliverClient工具函数。
* protos/orderer/ab.pb.go，AtomicBroadcast_DeliverClient接口定义和实现。

## 2、deliverClientIntf接口定义及实现

### 2.1、DeliverClient工具函数

```go
//构造deliverClient
func newDeliverClient(conn *grpc.ClientConn, client ab.AtomicBroadcast_DeliverClient, chainID string) *deliverClient
//代码在peer/channel/deliverclient.go
```

### 2.2、deliverClientIntf接口定义及实现

```go
type deliverClientIntf interface {
	getSpecifiedBlock(num uint64) (*common.Block, error)
	getOldestBlock() (*common.Block, error)
	getNewestBlock() (*common.Block, error)
	Close() error
}

type deliverClient struct {
	conn    *grpc.ClientConn
	client  ab.AtomicBroadcast_DeliverClient
	chainID string
}

//构造查询Envelope
func seekHelper(chainID string, position *ab.SeekPosition) *common.Envelope
//r.client.Send(seekHelper(r.chainID, &ab.SeekPosition{Type: &ab.SeekPosition_Specified{Specified: &ab.SeekSpecified{Number: blockNumber}}}))
func (r *deliverClient) seekSpecified(blockNumber uint64) error
//r.client.Send(seekHelper(r.chainID, &ab.SeekPosition{Type: &ab.SeekPosition_Oldest{Oldest: &ab.SeekOldest{}}}))
func (r *deliverClient) seekOldest() error
//return r.client.Send(seekHelper(r.chainID, &ab.SeekPosition{Type: &ab.SeekPosition_Newest{Newest: &ab.SeekNewest{}}}))
func (r *deliverClient) seekNewest() error
//r.client.Recv()读取块
func (r *deliverClient) readBlock() (*common.Block, error)
//r.seekSpecified(num)和r.readBlock()
func (r *deliverClient) getSpecifiedBlock(num uint64) (*common.Block, error)
//r.seekOldest()和r.readBlock()
func (r *deliverClient) getOldestBlock() (*common.Block, error)
//r.seekNewest()和r.readBlock()
func (r *deliverClient) getNewestBlock() (*common.Block, error)
//r.conn.Close()
func (r *deliverClient) Close() error
//cf.DeliverClient.getSpecifiedBlock(0)获取创世区块
func getGenesisBlock(cf *ChannelCmdFactory) (*common.Block, error)
//代码在peer/channel/deliverclient.go
```

func seekHelper(chainID string, position *ab.SeekPosition) *common.Envelope代码如下：

```go
func seekHelper(chainID string, position *ab.SeekPosition) *common.Envelope {
	seekInfo := &ab.SeekInfo{
		Start:    position,
		Stop:     position,
		Behavior: ab.SeekInfo_BLOCK_UNTIL_READY,
	}

	msgVersion := int32(0)
	epoch := uint64(0)
	env, err := utils.CreateSignedEnvelope(common.HeaderType_CONFIG_UPDATE, chainID, localmsp.NewSigner(), seekInfo, msgVersion, epoch)
	return env
}
//代码在peer/channel/deliverclient.go
```
