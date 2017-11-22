# Fabric 1.0源代码笔记 之 putils（protos/utils工具包）

## 1、putils概述

putils，即protos/utils工具包，代码分布在：protos/utils目录下。
包括：txutils.go、proputils.go、commonutils.go、blockutils.go。

## 2、txutils

```go
//TransactionAction.Payload => ChaincodeActionPayload
//ChaincodeActionPayload.Action.ProposalResponsePayload => ProposalResponsePayload
//ProposalResponsePayload.Extension => ChaincodeAction
//返回ChaincodeActionPayload和ChaincodeAction
func GetPayloads(txActions *peer.TransactionAction) (*peer.ChaincodeActionPayload, *peer.ChaincodeAction, error)
//[]byte反序列化为Envelope
func GetEnvelopeFromBlock(data []byte) (*common.Envelope, error)
func CreateSignedEnvelope(txType common.HeaderType, channelID string, signer crypto.LocalSigner, dataMsg proto.Message, msgVersion int32, epoch uint64) (*common.Envelope, error) 
func CreateSignedTx(proposal *peer.Proposal, signer msp.SigningIdentity, resps ...*peer.ProposalResponse) (*common.Envelope, error) {
func CreateProposalResponse(hdrbytes []byte, payl []byte, response *peer.Response, results []byte, events []byte, ccid *peer.ChaincodeID, visibility []byte, signingEndorser msp.SigningIdentity) (*peer.ProposalResponse, error)
func CreateProposalResponseFailure(hdrbytes []byte, payl []byte, response *peer.Response, results []byte, events []byte, ccid *peer.ChaincodeID, visibility []byte) (*peer.ProposalResponse, error)
func GetSignedProposal(prop *peer.Proposal, signer msp.SigningIdentity) (*peer.SignedProposal, error)
func GetSignedEvent(evt *peer.Event, signer msp.SigningIdentity) (*peer.SignedEvent, error)
func MockSignedEndorserProposalOrPanic(chainID string, cs *peer.ChaincodeSpec, creator, signature []byte) (*peer.SignedProposal, *peer.Proposal)
func MockSignedEndorserProposal2OrPanic(chainID string, cs *peer.ChaincodeSpec, signer msp.SigningIdentity) (*peer.SignedProposal, *peer.Proposal)
func GetBytesProposalPayloadForTx(payload *peer.ChaincodeProposalPayload, visibility []byte) ([]byte, error)
func GetProposalHash2(header *common.Header, ccPropPayl []byte) ([]byte, error)
func GetProposalHash1(header *common.Header, ccPropPayl []byte, visibility []byte) ([]byte, error)
//代码在protos/utils/txutils.go
```

## 3、proputils

```go
func GetChaincodeInvocationSpec(prop *peer.Proposal) (*peer.ChaincodeInvocationSpec, error)
func GetChaincodeProposalContext(prop *peer.Proposal) ([]byte, map[string][]byte, error)
func GetHeader(bytes []byte) (*common.Header, error)
func GetNonce(prop *peer.Proposal) ([]byte, error)
func GetChaincodeHeaderExtension(hdr *common.Header) (*peer.ChaincodeHeaderExtension, error)
func GetProposalResponse(prBytes []byte) (*peer.ProposalResponse, error)
func GetChaincodeDeploymentSpec(code []byte) (*peer.ChaincodeDeploymentSpec, error)
func GetChaincodeAction(caBytes []byte) (*peer.ChaincodeAction, error)
func GetResponse(resBytes []byte) (*peer.Response, error)
func GetChaincodeEvents(eBytes []byte) (*peer.ChaincodeEvent, error)
func GetProposalResponsePayload(prpBytes []byte) (*peer.ProposalResponsePayload, error)
func GetProposal(propBytes []byte) (*peer.Proposal, error)
//e.Payload反序列化为Payload
func GetPayload(e *common.Envelope) (*common.Payload, error)
//[]byte反序列化为Transaction
func GetTransaction(txBytes []byte) (*peer.Transaction, error)
func GetChaincodeActionPayload(capBytes []byte) (*peer.ChaincodeActionPayload, error)
func GetChaincodeProposalPayload(bytes []byte) (*peer.ChaincodeProposalPayload, error)
func GetSignatureHeader(bytes []byte) (*common.SignatureHeader, error)
func CreateChaincodeProposal(typ common.HeaderType, chainID string, cis *peer.ChaincodeInvocationSpec, creator []byte) (*peer.Proposal, string, error)
func CreateChaincodeProposalWithTransient(typ common.HeaderType, chainID string, cis *peer.ChaincodeInvocationSpec, creator []byte, transientMap map[string][]byte) (*peer.Proposal, string, error)
func CreateChaincodeProposalWithTxIDNonceAndTransient(txid string, typ common.HeaderType, chainID string, cis *peer.ChaincodeInvocationSpec, nonce, creator []byte, transientMap map[string][]byte) (*peer.Proposal, string, error)
func GetBytesProposalResponsePayload(hash []byte, response *peer.Response, result []byte, event []byte, ccid *peer.ChaincodeID) ([]byte, error)
func GetBytesChaincodeProposalPayload(cpp *peer.ChaincodeProposalPayload) ([]byte, error)
func GetBytesResponse(res *peer.Response) ([]byte, error)
func GetBytesChaincodeEvent(event *peer.ChaincodeEvent) ([]byte, error)
func GetBytesChaincodeActionPayload(cap *peer.ChaincodeActionPayload) ([]byte, error)
func GetBytesProposalResponse(pr *peer.ProposalResponse) ([]byte, error)
func GetBytesProposal(prop *peer.Proposal) ([]byte, error)
func GetBytesHeader(hdr *common.Header) ([]byte, error)
func GetBytesSignatureHeader(hdr *common.SignatureHeader) ([]byte, error)
func GetBytesTransaction(tx *peer.Transaction) ([]byte, error)
func GetBytesPayload(payl *common.Payload) ([]byte, error)
func GetBytesEnvelope(env *common.Envelope) ([]byte, error)
func GetActionFromEnvelope(envBytes []byte) (*peer.ChaincodeAction, error)
func CreateProposalFromCIS(typ common.HeaderType, chainID string, cis *peer.ChaincodeInvocationSpec, creator []byte) (*peer.Proposal, string, error)
func CreateInstallProposalFromCDS(ccpack proto.Message, creator []byte) (*peer.Proposal, string, error)
func CreateDeployProposalFromCDS(chainID string, cds *peer.ChaincodeDeploymentSpec, creator []byte, policy []byte, escc []byte, vscc []byte) (*peer.Proposal, string, error)
func CreateUpgradeProposalFromCDS(chainID string, cds *peer.ChaincodeDeploymentSpec, creator []byte, policy []byte, escc []byte, vscc []byte) (*peer.Proposal, string, error)
func createProposalFromCDS(chainID string, msg proto.Message, creator []byte, policy []byte, escc []byte, vscc []byte, propType string) (*peer.Proposal, string, error)
func ComputeProposalTxID(nonce, creator []byte) (string, error)
func CheckProposalTxID(txid string, nonce, creator []byte) error
func ComputeProposalBinding(proposal *peer.Proposal) ([]byte, error)
func computeProposalBindingInternal(nonce, creator []byte, epoch uint64) ([]byte, error)
//代码在protos/utils/proputils.go
```

## 4、commonutils

```go
func MarshalOrPanic(pb proto.Message) []byte
func Marshal(pb proto.Message) ([]byte, error)
func CreateNonceOrPanic() []byte
func CreateNonce() ([]byte, error)
func UnmarshalPayloadOrPanic(encoded []byte) *cb.Payload
func UnmarshalPayload(encoded []byte) (*cb.Payload, error)
func UnmarshalEnvelopeOrPanic(encoded []byte) *cb.Envelope
func UnmarshalEnvelope(encoded []byte) (*cb.Envelope, error)
func UnmarshalEnvelopeOfType(envelope *cb.Envelope, headerType cb.HeaderType, message proto.Message) (*cb.ChannelHeader, error)
func ExtractEnvelopeOrPanic(block *cb.Block, index int) *cb.Envelope
func ExtractEnvelope(block *cb.Block, index int) (*cb.Envelope, error)
func ExtractPayloadOrPanic(envelope *cb.Envelope) *cb.Payload
func ExtractPayload(envelope *cb.Envelope) (*cb.Payload, error)
func MakeChannelHeader(headerType cb.HeaderType, version int32, chainID string, epoch uint64) *cb.ChannelHeader
func MakeSignatureHeader(serializedCreatorCertChain []byte, nonce []byte) *cb.SignatureHeader
func SetTxID(channelHeader *cb.ChannelHeader, signatureHeader *cb.SignatureHeader) error
func MakePayloadHeader(ch *cb.ChannelHeader, sh *cb.SignatureHeader) *cb.Header
func NewSignatureHeaderOrPanic(signer crypto.LocalSigner) *cb.SignatureHeader
func SignOrPanic(signer crypto.LocalSigner, msg []byte) []byte
//[]byte反序列化为ChannelHeader
func UnmarshalChannelHeader(bytes []byte) (*cb.ChannelHeader, error)
func UnmarshalChaincodeID(bytes []byte) (*pb.ChaincodeID, error)
func IsConfigBlock(block *cb.Block) bool
//代码在protos/utils/commonutils.go
```

## 5、blockutils

```go
func GetChainIDFromBlockBytes(bytes []byte) (string, error)
func GetChainIDFromBlock(block *cb.Block) (string, error)
func GetMetadataFromBlock(block *cb.Block, index cb.BlockMetadataIndex) (*cb.Metadata, error)
func GetMetadataFromBlockOrPanic(block *cb.Block, index cb.BlockMetadataIndex) *cb.Metadata
func GetLastConfigIndexFromBlock(block *cb.Block) (uint64, error)
func GetLastConfigIndexFromBlockOrPanic(block *cb.Block) uint64
func GetBlockFromBlockBytes(blockBytes []byte) (*cb.Block, error)
func CopyBlockMetadata(src *cb.Block, dst *cb.Block)
func InitBlockMetadata(block *cb.Block)
//代码在protos/utils/blockutils.go
```