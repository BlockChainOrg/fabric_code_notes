# Fabric 1.0源代码笔记 之 consenter（共识插件）

## 1、consenter概述

consenter，即共识插件，负责接受交易信息进行排序，以及对交易进行切割并打包，打包后返回批量交易。
Orderer包含三种共识插件：
* solo，单节点的排序功能，用于试验。
* kafka，基于kafka集群实现的排序，可用于生产环境。
* SBFT，支持拜占庭容错的排序实现，尚未完成开发。

consenter代码分布在orderer/multichain、orderer/solo、orderer/kafka、orderer/common/blockcutter、orderer/common/filter目录下。目录结构如下：

* orderer/multichain目录：
	* chainsupport.go，Consenter和Chain接口定义。
* orderer/solo目录，solo版本共识插件。
* orderer/kafka目录，kafka版本共识插件。
* orderer/common/blockcutter目录，block cutter相关实现，即Receiver接口定义及实现。
* orderer/common/filter目录，过滤器相关实现。

## 2、Consenter和Chain接口定义

```go
type Consenter interface { //共识插件接口
	//获取共识插件对应的Chain实例
	HandleChain(support ConsenterSupport, metadata *cb.Metadata) (Chain, error)
}

type Chain interface {
	//接受消息
	Enqueue(env *cb.Envelope) bool
	Errored() <-chan struct{}
	Start() //开始
	Halt() //挂起
}
//代码在orderer/multichain/chainsupport.go
```

## 3、solo版本共识插件

### 3.1、Consenter接口实现

```go
type consenter struct{}

//构造consenter
func New() multichain.Consenter {
	return &consenter{}
}

//获取solo共识插件对应的Chain实例
func (solo *consenter) HandleChain(support multichain.ConsenterSupport, metadata *cb.Metadata) (multichain.Chain, error) {
	return newChain(support), nil
}
//代码在orderer/solo/consensus.go
```

### 3.2、Chain接口实现

```go
type chain struct {
	support  multichain.ConsenterSupport
	sendChan chan *cb.Envelope //交易数据通道
	exitChan chan struct{} //退出信号
}

//构造chain
func newChain(support multichain.ConsenterSupport) *chain
//go ch.main()
func (ch *chain) Start()
//关闭通道，close(ch.exitChan)
func (ch *chain) Halt()
//Envelope写入通道ch.sendChan
func (ch *chain) Enqueue(env *cb.Envelope) bool
//获取ch.exitChan
func (ch *chain) Errored() <-chan struct{}
//goroutine
func (ch *chain) main()
//代码在orderer/solo/consensus.go
```

### 3.3、main()实现

```go
func (ch *chain) main() {
	var timer <-chan time.Time //超时通道

	for {
		select {
		case msg := <-ch.sendChan: //接受交易消息
			batches, committers, ok, _ := ch.support.BlockCutter().Ordered(msg)
			if ok && len(batches) == 0 && timer == nil {
				timer = time.After(ch.support.SharedConfig().BatchTimeout())
				continue
			}
			for i, batch := range batches {
				block := ch.support.CreateNextBlock(batch) //每个批处理创建一个块
				ch.support.WriteBlock(block, committers[i], nil) //写入块
			}
			if len(batches) > 0 {
				timer = nil
			}
		case <-timer:
			//clear the timer
			timer = nil

			batch, committers := ch.support.BlockCutter().Cut()
			if len(batch) == 0 {
				logger.Warningf("Batch timer expired with no pending requests, this might indicate a bug")
				continue
			}
			logger.Debugf("Batch timer expired, creating block")
			block := ch.support.CreateNextBlock(batch)
			ch.support.WriteBlock(block, committers, nil)
		case <-ch.exitChan: //退出信号
			logger.Debugf("Exiting")
			return
		}
	}
}
//代码在orderer/solo/consensus.go
```

## 4、kafka版本共识插件



## 5、blockcutter

### 5.1、Receiver接口定义

```go
type Receiver interface {
	//交易信息排序
	Ordered(msg *cb.Envelope) (messageBatches [][]*cb.Envelope, committers [][]filter.Committer, validTx bool, pending bool)
	//返回当前批处理，并启动一个新批次
	Cut() ([]*cb.Envelope, []filter.Committer)
}
//代码在orderer/common/blockcutter/blockcutter.go
```

### 5.2、Receiver接口实现

```go
type receiver struct {
	sharedConfigManager   config.Orderer
	filters               *filter.RuleSet
	pendingBatch          []*cb.Envelope
	pendingBatchSizeBytes uint32
	pendingCommitters     []filter.Committer
}

//构造receiver
func NewReceiverImpl(sharedConfigManager config.Orderer, filters *filter.RuleSet) Receiver
//交易信息排序
func (r *receiver) Ordered(msg *cb.Envelope) (messageBatches [][]*cb.Envelope, committerBatches [][]filter.Committer, validTx bool, pending bool)
//返回当前批处理，并启动一个新批次
func (r *receiver) Cut() ([]*cb.Envelope, []filter.Committer)
//获取消息长度
func messageSizeBytes(message *cb.Envelope) uint32
//代码在orderer/common/blockcutter/blockcutter.go
```

func (r *receiver) Ordered(msg *cb.Envelope) (messageBatches [][]*cb.Envelope, committerBatches [][]filter.Committer, validTx bool, pending bool)代码如下：

```go
func (r *receiver) Ordered(msg *cb.Envelope) (messageBatches [][]*cb.Envelope, committerBatches [][]filter.Committer, validTx bool, pending bool) {
	committer, err := r.filters.Apply(msg) //执行过滤器
		if err != nil {
		logger.Debugf("Rejecting message: %s", err)
		return
	}
	
	validTx = true
	messageSizeBytes := messageSizeBytes(msg)
	//孤立的块，或者交易大小超限，将被隔离
	if committer.Isolated() || messageSizeBytes > r.sharedConfigManager.BatchSize().PreferredMaxBytes {
		if committer.Isolated() {
			//发现消息要求被隔离, 切割成自己的批
			logger.Debugf("Found message which requested to be isolated, cutting into its own batch")
		} else {
			//当前消息 大于默认批处理大小, 将被隔离
			logger.Debugf("The current message, with %v bytes, is larger than the preferred batch size of %v bytes and will be isolated.", messageSizeBytes, r.sharedConfigManager.BatchSize().PreferredMaxBytes)
		}
		//剪切批处理
		if len(r.pendingBatch) > 0 {
			messageBatch, committerBatch := r.Cut()
			messageBatches = append(messageBatches, messageBatch)
			committerBatches = append(committerBatches, committerBatch)
		}

		//创建新批次
		messageBatches = append(messageBatches, []*cb.Envelope{msg})
		committerBatches = append(committerBatches, []filter.Committer{committer})

		return
	}

	//混合块且大小未超限
	messageWillOverflowBatchSizeBytes := r.pendingBatchSizeBytes+messageSizeBytes > r.sharedConfigManager.BatchSize().PreferredMaxBytes

	if messageWillOverflowBatchSizeBytes { //添加当前消息后，批处理将溢出
		messageBatch, committerBatch := r.Cut()
		messageBatches = append(messageBatches, messageBatch)
		committerBatches = append(committerBatches, committerBatch)
	}

	//消息添加到批处理
	r.pendingBatch = append(r.pendingBatch, msg)
	r.pendingBatchSizeBytes += messageSizeBytes
	r.pendingCommitters = append(r.pendingCommitters, committer)
	pending = true

	if uint32(len(r.pendingBatch)) >= r.sharedConfigManager.BatchSize().MaxMessageCount {
		//批次大小满足, 切割批次
		messageBatch, committerBatch := r.Cut()
		messageBatches = append(messageBatches, messageBatch)
		committerBatches = append(committerBatches, committerBatch)
		pending = false
	}

	return
}
//代码在orderer/common/blockcutter/blockcutter.go
```

## 6、filter相关实现（过滤器）

filter更详细内容，参考：[Fabric 1.0源代码笔记 之 consenter（共识插件） #filter（过滤器）](filter.md)