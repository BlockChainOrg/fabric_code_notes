# Fabric 1.0源代码笔记 之 scc（系统链码） #cscc（通道相关）

## 1、cscc概述

cscc代码在core/scc/cscc/configure.go。

## 2、PeerConfiger结构体

```go
type PeerConfiger struct {
	policyChecker policy.PolicyChecker
}
//代码在core/scc/cscc/configure.go
```

## 3、Init方法

```go
func (e *PeerConfiger) Init(stub shim.ChaincodeStubInterface) pb.Response {
	//初始化策略检查器，用于访问控制
	e.policyChecker = policy.NewPolicyChecker(
		peer.NewChannelPolicyManagerGetter(),
		mgmt.GetLocalMSP(),
		mgmt.NewLocalMSPPrincipalGetter(),
	)
	return shim.Success(nil)
}
//代码在core/scc/cscc/configure.go
```

## 4、Invoke方法

```go
func (e *PeerConfiger) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
	//args[0]为JoinChain或GetChannels
	args := stub.GetArgs()
	fname := string(args[0]) //Invoke function
	sp, err := stub.GetSignedProposal() //获取SignedProposal

	switch fname {
	case JoinChain: //加入通道
		//此处args[1]为创世区块
		block, err := utils.GetBlockFromBlockBytes(args[1])
		cid, err := utils.GetChainIDFromBlock(block)
		err := validateConfigBlock(block)
		err = e.policyChecker.CheckPolicyNoChannel(mgmt.Admins, sp)
		return joinChain(cid, block)
	case GetConfigBlock:
		err = e.policyChecker.CheckPolicy(string(args[1]), policies.ChannelApplicationReaders, sp)
		return getConfigBlock(args[1])
	case GetChannels:
		err = e.policyChecker.CheckPolicyNoChannel(mgmt.Members, sp)
		return getChannels()

	}
}
//代码在core/scc/cscc/configure.go
```

## 5、其他方法

```go
//校验创世区块
func validateConfigBlock(block *common.Block) error
func joinChain(chainID string, block *common.Block) pb.Response
func getConfigBlock(chainID []byte) pb.Response
func getChannels() pb.Response
//代码在core/scc/cscc/configure.go
```

### 5.1、joinChain

```go
func joinChain(chainID string, block *common.Block) pb.Response {
	err := peer.CreateChainFromBlock(block) //创建chain
	peer.InitChain(chainID)
	err := producer.SendProducerBlockEvent(block)
	return shim.Success(nil)
}
//代码在core/scc/cscc/configure.go
```

#### 5.1.1、创建Chain（或channel）

peer.CreateChainFromBlock(block)代码如下：

```go
func CreateChainFromBlock(cb *common.Block) error {
	cid, err := utils.GetChainIDFromBlock(cb) //获取ChainID
	var l ledger.PeerLedger
	l, err = ledgermgmt.CreateLedger(cb) //创建Ledger
	return createChain(cid, l, cb)
}
//代码在core/peer/peer.go
```

createChain(cid, l, cb)代码如下：

```go
func createChain(cid string, ledger ledger.PeerLedger, cb *common.Block) error {
	envelopeConfig, err := utils.ExtractEnvelope(cb, 0) //获取配置Envelope
	configtxInitializer := configtx.NewInitializer() //type initializer struct
	gossipEventer := service.GetGossipService().NewConfigEventer() //获取gossipServiceInstance

	gossipCallbackWrapper := func(cm configtxapi.Manager) {
		ac, ok := configtxInitializer.ApplicationConfig()
		if !ok {
			// TODO, handle a missing ApplicationConfig more gracefully
			ac = nil
		}
		gossipEventer.ProcessConfigUpdate(&chainSupport{
			Manager:     cm,
			Application: ac,
		})
		service.GetGossipService().SuspectPeers(func(identity api.PeerIdentityType) bool {
			// TODO: this is a place-holder that would somehow make the MSP layer suspect
			// that a given certificate is revoked, or its intermediate CA is revoked.
			// In the meantime, before we have such an ability, we return true in order
			// to suspect ALL identities in order to validate all of them.
			return true
		})
	}

	trustedRootsCallbackWrapper := func(cm configtxapi.Manager) {
		updateTrustedRoots(cm)
	}

	configtxManager, err := configtx.NewManagerImpl(
		envelopeConfig,
		configtxInitializer,
		[]func(cm configtxapi.Manager){gossipCallbackWrapper, trustedRootsCallbackWrapper},
	)
	if err != nil {
		return err
	}

	// TODO remove once all references to mspmgmt are gone from peer code
	mspmgmt.XXXSetMSPManager(cid, configtxManager.MSPManager())

	ac, ok := configtxInitializer.ApplicationConfig()
	if !ok {
		ac = nil
	}
	cs := &chainSupport{
		Manager:     configtxManager,
		Application: ac, // TODO, refactor as this is accessible through Manager
		ledger:      ledger,
	}

	c := committer.NewLedgerCommitterReactive(ledger, txvalidator.NewTxValidator(cs), func(block *common.Block) error {
		chainID, err := utils.GetChainIDFromBlock(block)
		if err != nil {
			return err
		}
		return SetCurrConfigBlock(block, chainID)
	})

	ordererAddresses := configtxManager.ChannelConfig().OrdererAddresses()
	if len(ordererAddresses) == 0 {
		return errors.New("No ordering service endpoint provided in configuration block")
	}
	service.GetGossipService().InitializeChannel(cs.ChainID(), c, ordererAddresses)

	chains.Lock()
	defer chains.Unlock()
	chains.list[cid] = &chain{
		cs:        cs,
		cb:        cb,
		committer: c,
	}
	return nil
}
//代码在core/peer/peer.go
```

补充initializer：

```go
type initializer struct {
	*resources
	ppr *policyProposerRoot
}
//代码在common/configtx/initializer.go
```

