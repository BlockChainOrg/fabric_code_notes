# Fabric 1.0源代码笔记 之 Peer（4）peer chaincode命令及子命令实现

## 1、peer chaincode install子命令实现（安装链码）

### 1.1、初始化Endorser客户端

```go
cf, err = InitCmdFactory(true, false)
//代码在peer/chaincode/install.go
```

cf, err = InitCmdFactory(true, false)代码如下：

```go
func InitCmdFactory(isEndorserRequired, isOrdererRequired bool) (*ChaincodeCmdFactory, error) {
	var err error
	var endorserClient pb.EndorserClient
	if isEndorserRequired {
		endorserClient, err = common.GetEndorserClientFnc() //func GetEndorserClient() (pb.EndorserClient, error)
	}
	signer, err := common.GetDefaultSignerFnc()
	var broadcastClient common.BroadcastClient
	if isOrdererRequired {
		//此处未用到，暂略
	}
	return &ChaincodeCmdFactory{
		EndorserClient:  endorserClient,
		Signer:          signer,
		BroadcastClient: broadcastClient,
	}, nil
}
//代码在peer/chaincode/common.go
```
