# Fabric 1.0源代码笔记 之 Tx（Transaction 交易）

## 1、Tx概述

Tx，即Transaction，交易或事务。

Tx代码分布目录结构如下：

* protos/common/common.pb.go，交易的封装即Envelope结构体。也包括Payload、Header、ChannelHeader和SignatureHeader。
* protos/utils目录，交易相关部分工具函数，包括txutils.go、proputils.go和commonutils.go。
* core/ledger/kvledger/txmgmt/rwsetutil目录，读写集相关结构体及方法。

## 2、交易的封装Envelope结构体

### 2.1、Envelope结构体

Envelope直译为信封，封装Payload和Signature。

```go
type Envelope struct { //用签名包装Payload，以便对信息做身份验证
	Payload []byte //Payload序列化
	Signature []byte //Payload header中指定的创建者签名
}
//代码在protos/common/common.pb.go
```

### 2.2、Payload相关结构体

Payload直译为有效载荷。Payload结构体：

```go
type Payload struct {
	Header *Header //Header
	Data []byte //Transaction序列化
}
//代码在protos/common/common.pb.go
```

Header结构体：

```go
type Header struct {
	ChannelHeader   []byte
	SignatureHeader []byte
}
//代码在protos/common/common.pb.go
```

ChannelHeader结构体：

```go
type ChannelHeader struct {
	Type int32
	Version int32 //消息协议版本
	Timestamp *google_protobuf.Timestamp //创建消息时的本地时间
	ChannelId string //消息绑定的ChannelId
	TxId string //TxId
	Epoch uint64 //纪元
	Extension []byte //可附加的扩展
}
//代码在protos/common/common.pb.go
```

补充HeaderType:

```go
type HeaderType int32

const (
	HeaderType_MESSAGE              HeaderType = 0
	HeaderType_CONFIG               HeaderType = 1
	HeaderType_CONFIG_UPDATE        HeaderType = 2
	HeaderType_ENDORSER_TRANSACTION HeaderType = 3
	HeaderType_ORDERER_TRANSACTION  HeaderType = 4
	HeaderType_DELIVER_SEEK_INFO    HeaderType = 5
	HeaderType_CHAINCODE_PACKAGE    HeaderType = 6
)
//代码在protos/common/common.pb.go
```

SignatureHeader结构体：

```go
type SignatureHeader struct {
	Creator []byte //消息的创建者, 指定为证书链
	Nonce []byte //可能只使用一次的任意数字，可用于检测重播攻击
}
//代码在protos/common/common.pb.go
```

### 2.3、Transaction相关结构体

Transaction结构体：

```go
type Transaction struct {
	Actions []*TransactionAction //Payload.Data是个TransactionAction数组，容纳每个交易
}
//代码在protos/peer/transaction.pb.go
```

TransactionAction结构体：

```go
type TransactionAction struct {
	Header []byte
	Payload []byte
}
//代码在protos/peer/transaction.pb.go
```

### 2.4、ChaincodeActionPayload相关结构体

ChaincodeActionPayload结构体：

```go
type ChaincodeActionPayload struct {
	ChaincodeProposalPayload []byte
	Action *ChaincodeEndorsedAction
}
//代码在protos/peer/transaction.pb.go
```

ChaincodeEndorsedAction结构体：

```go
type ChaincodeEndorsedAction struct {
	ProposalResponsePayload []byte //ProposalResponsePayload序列化
	Endorsements []*Endorsement
}
//代码在protos/peer/transaction.pb.go
```

ProposalResponsePayload结构体：

```go
type ProposalResponsePayload struct {
	ProposalHash []byte
	Extension []byte //ChaincodeAction序列化
}
//代码在protos/peer/proposal_response.pb.go
```

ChaincodeAction结构体：

```go
type ChaincodeAction struct {
	Results []byte
	Events []byte
	Response *Response
	ChaincodeId *ChaincodeID
}
//代码在protos/peer/proposal.pb.go
```

## 3、交易验证代码TxValidationFlags

TxValidationFlags是交易验证代码的数组，在commiter验证块时使用。

```go
type TxValidationFlags []uint8

//创建TxValidationFlags数组
func NewTxValidationFlags(size int) TxValidationFlags
//为指定的交易设置交易验证代码
func (obj TxValidationFlags) SetFlag(txIndex int, flag peer.TxValidationCode) 
//获取指定交易的交易验证代码
func (obj TxValidationFlags) Flag(txIndex int) peer.TxValidationCode 
//检查指定的交易是否有效
func (obj TxValidationFlags) IsValid(txIndex int) bool
//检查指定的交易是否无效
func (obj TxValidationFlags) IsInvalid(txIndex int) bool
//指定交易的交易验证代码与flag比较，相同为true
func (obj TxValidationFlags) IsSetTo(txIndex int, flag peer.TxValidationCode) bool
//代码在core/ledger/util/txvalidationflags.go
```

补充peer.TxValidationCode：

```go
type TxValidationCode int32

const (
	TxValidationCode_VALID                        TxValidationCode = 0
	TxValidationCode_NIL_ENVELOPE                 TxValidationCode = 1
	TxValidationCode_BAD_PAYLOAD                  TxValidationCode = 2
	TxValidationCode_BAD_COMMON_HEADER            TxValidationCode = 3
	TxValidationCode_BAD_CREATOR_SIGNATURE        TxValidationCode = 4
	TxValidationCode_INVALID_ENDORSER_TRANSACTION TxValidationCode = 5
	TxValidationCode_INVALID_CONFIG_TRANSACTION   TxValidationCode = 6
	TxValidationCode_UNSUPPORTED_TX_PAYLOAD       TxValidationCode = 7
	TxValidationCode_BAD_PROPOSAL_TXID            TxValidationCode = 8
	TxValidationCode_DUPLICATE_TXID               TxValidationCode = 9
	TxValidationCode_ENDORSEMENT_POLICY_FAILURE   TxValidationCode = 10
	TxValidationCode_MVCC_READ_CONFLICT           TxValidationCode = 11
	TxValidationCode_PHANTOM_READ_CONFLICT        TxValidationCode = 12
	TxValidationCode_UNKNOWN_TX_TYPE              TxValidationCode = 13
	TxValidationCode_TARGET_CHAIN_NOT_FOUND       TxValidationCode = 14
	TxValidationCode_MARSHAL_TX_ERROR             TxValidationCode = 15
	TxValidationCode_NIL_TXACTION                 TxValidationCode = 16
	TxValidationCode_EXPIRED_CHAINCODE            TxValidationCode = 17
	TxValidationCode_CHAINCODE_VERSION_CONFLICT   TxValidationCode = 18
	TxValidationCode_BAD_HEADER_EXTENSION         TxValidationCode = 19
	TxValidationCode_BAD_CHANNEL_HEADER           TxValidationCode = 20
	TxValidationCode_BAD_RESPONSE_PAYLOAD         TxValidationCode = 21
	TxValidationCode_BAD_RWSET                    TxValidationCode = 22
	TxValidationCode_ILLEGAL_WRITESET             TxValidationCode = 23
	TxValidationCode_INVALID_OTHER_REASON         TxValidationCode = 255
)
//代码在protos/peer/transaction.pb.go
```

## 4、交易相关部分工具函数（protos/utils包）

putils更详细内容，参考：[Fabric 1.0源代码笔记 之 putils（protos/utils工具包）](../putils/README.md)

## 5、rwsetutil（读写集）


