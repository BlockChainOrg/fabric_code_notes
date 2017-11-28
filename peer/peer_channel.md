# Fabric 1.0源代码笔记 之 Peer（3）peer channel命令及子命令实现

## 1、peer channel create子命令实现（创建通道）

### 1.1、初始化Orderer客户端

```go
const (
	EndorserRequired    EndorserRequirement = true
	EndorserNotRequired EndorserRequirement = false
	OrdererRequired     OrdererRequirement  = true
	OrdererNotRequired  OrdererRequirement  = false
)

cf, err = InitCmdFactory(EndorserNotRequired, OrdererRequired)
//代码在peer/channel/create.go
```

cf, err = InitCmdFactory(EndorserNotRequired, OrdererRequired)代码如下：

```go
type ChannelCmdFactory struct {
	EndorserClient   pb.EndorserClient //EndorserClient
	Signer           msp.SigningIdentity //Signer
	BroadcastClient  common.BroadcastClient //BroadcastClient
	DeliverClient    deliverClientIntf //DeliverClient
	BroadcastFactory BroadcastClientFactory //BroadcastClientFactory，type BroadcastClientFactory func() (common.BroadcastClient, error)
}

func InitCmdFactory(isEndorserRequired EndorserRequirement, isOrdererRequired OrdererRequirement) (*ChannelCmdFactory, error) {
	var err error
	cmdFact := &ChannelCmdFactory{}
	cmdFact.Signer, err = common.GetDefaultSignerFnc() //GetDefaultSignerFnc = GetDefaultSigner

	cmdFact.BroadcastFactory = func() (common.BroadcastClient, error) {
		return common.GetBroadcastClientFnc(orderingEndpoint, tls, caFile) //GetBroadcastClientFnc = GetBroadcastClient
	}

	//peer channel join或list需要endorser，本处暂未使用
	if isEndorserRequired {
		cmdFact.EndorserClient, err = common.GetEndorserClientFnc()
	}

	//peer channel create或fetch需要orderer
	if isOrdererRequired {
		var opts []grpc.DialOption
		if tls {
			if caFile != "" {
				creds, err := credentials.NewClientTLSFromFile(caFile, "")
				opts = append(opts, grpc.WithTransportCredentials(creds))
			}
		} else {
			opts = append(opts, grpc.WithInsecure())
		}
		conn, err := grpc.Dial(orderingEndpoint, opts...)
		client, err := ab.NewAtomicBroadcastClient(conn).Deliver(context.TODO())
		cmdFact.DeliverClient = newDeliverClient(conn, client, chainID) //构造deliverClient
	}
	return cmdFact, nil
}

//代码在peer/channel/channel.go
```

### 1.2、发送创建区块链的交易

```go
err = sendCreateChainTransaction(cf)
//代码在peer/channel/create.go
```

sendCreateChainTransaction(cf)代码如下：

```go
func sendCreateChainTransaction(cf *ChannelCmdFactory) error {
	var err error
	var chCrtEnv *cb.Envelope

	if channelTxFile != "" { //peer channel create -f指定通道交易配置文件
		chCrtEnv, err = createChannelFromConfigTx(channelTxFile) //获取创世区块
	} else {
		chCrtEnv, err = createChannelFromDefaults(cf) //没有指定通道交易配置文件
	}

	chCrtEnv, err = sanityCheckAndSignConfigTx(chCrtEnv) //检查和签名交易配置
	var broadcastClient common.BroadcastClient
	broadcastClient, err = cf.BroadcastFactory()
	defer broadcastClient.Close()
	err = broadcastClient.Send(chCrtEnv)
	return err
}
//代码在peer/channel/create.go
```

### 1.3、获取创世区块并写入文件

```go
var block *cb.Block
block, err = getGenesisBlock(cf) //获取创世区块
b, err := proto.Marshal(block) //块序列化
err = ioutil.WriteFile(file, b, 0644) //写入文件
//代码在peer/channel/create.go
```

## 2、peer channel join子命令实现（加入通道）

### 2.1、初始化Endorser客户端

```go
cf, err = InitCmdFactory(EndorserRequired, OrdererNotRequired)
//代码在peer/channel/join.go
```

### 2.2、生成ChaincodeInvocationSpec消息（cscc.JoinChain）

```go
spec, err := getJoinCCSpec()
invocation := &pb.ChaincodeInvocationSpec{ChaincodeSpec: spec}
//代码在peer/channel/join.go
```

spec, err := getJoinCCSpec()代码如下：

```go
func getJoinCCSpec() (*pb.ChaincodeSpec, error) {
	gb, err := ioutil.ReadFile(genesisBlockPath)
	input := &pb.ChaincodeInput{Args: [][]byte{[]byte(cscc.JoinChain), gb}}
	spec := &pb.ChaincodeSpec{
		Type:        pb.ChaincodeSpec_Type(pb.ChaincodeSpec_Type_value["GOLANG"]),
		ChaincodeId: &pb.ChaincodeID{Name: "cscc"},
		Input:       input,
	}
	return spec, nil
}
//代码在peer/channel/join.go
```

### 2.3、创建Proposal并签名

```go
creator, err := cf.Signer.Serialize()
//从CIS创建Proposal，CIS即ChaincodeInvocationSpec
//调用CreateChaincodeProposal(typ, chainID, cis, creator)
//而后调用CreateChaincodeProposalWithTransient(typ, chainID, cis, creator, nil)
prop, _, err = putils.CreateProposalFromCIS(pcommon.HeaderType_CONFIG, "", invocation, creator)
var signedProp *pb.SignedProposal
signedProp, err = putils.GetSignedProposal(prop, cf.Signer)
//代码在peer/channel/join.go
```

CreateChaincodeProposalWithTransient(typ, chainID, cis, creator, nil)代码如下：

```go
func CreateChaincodeProposalWithTransient(typ common.HeaderType, chainID string, cis *peer.ChaincodeInvocationSpec, creator []byte, transientMap map[string][]byte) (*peer.Proposal, str    ing, error) {
	//创建随机Nonce
	nonce, err := crypto.GetRandomNonce()
	//计算交易ID
	//digest, err := factory.GetDefault().Hash(append(nonce, creator...),&bccsp.SHA256Opts{})
	txid, err := ComputeProposalTxID(nonce, creator)
	return CreateChaincodeProposalWithTxIDNonceAndTransient(txid, typ, chainID, cis, nonce, creator, transientMap)
}
//代码在protos/utils/proputils.go
```

CreateChaincodeProposalWithTxIDNonceAndTransient(txid, typ, chainID, cis, nonce, creator, transientMap)代码如下：

```go
func CreateChaincodeProposalWithTxIDNonceAndTransient(txid string, typ common.HeaderType, chainID string, cis *peer.ChaincodeInvocationSpec, nonce, creator []byte, transientMap map[string][]byte) (*peer.Proposal, string, error) {
	ccHdrExt := &peer.ChaincodeHeaderExtension{ChaincodeId: cis.ChaincodeSpec.ChaincodeId}
	ccHdrExtBytes, err := proto.Marshal(ccHdrExt)
	cisBytes, err := proto.Marshal(cis)
	ccPropPayload := &peer.ChaincodeProposalPayload{Input: cisBytes, TransientMap: transientMap}
	ccPropPayloadBytes, err := proto.Marshal(ccPropPayload)
	var epoch uint64 = 0
	timestamp := util.CreateUtcTimestamp()
	hdr := &common.Header{ChannelHeader: MarshalOrPanic(&common.ChannelHeader{
		Type:      int32(typ),
		TxId:      txid,
		Timestamp: timestamp,
		ChannelId: chainID,
		Extension: ccHdrExtBytes,
		Epoch:     epoch}),
		SignatureHeader: MarshalOrPanic(&common.SignatureHeader{Nonce: nonce, Creator: creator})}
	hdrBytes, err := proto.Marshal(hdr)
	return &peer.Proposal{Header: hdrBytes, Payload: ccPropPayloadBytes}, txid, nil
}
//代码在protos/utils/proputils.go
```

### 2.3、提交并处理Proposal

```go
var proposalResp *pb.ProposalResponse
proposalResp, err = cf.EndorserClient.ProcessProposal(context.Background(), signedProp)
//代码在peer/channel/join.go
```