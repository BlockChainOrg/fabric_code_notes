# Fabric 1.0源代码笔记 之 configtx（配置交易） #genesis（系统通道创世区块）

## 1、genesis概述

genesis，即创世区块，此处特指系统通道的创世区块。
相关代码在common/genesis/genesis.go，即Factory接口及实现。

## 2、Factory接口定义

```go
type Factory interface {
	Block(channelID string) (*cb.Block, error)
}
//代码在common/genesis/genesis.go
```

## 3、Factory接口实现

```go
msgVersion = int32(1)
epoch = 0

type factory struct {
	template configtx.Template
}

func NewFactoryImpl(template configtx.Template) Factory {
	return &factory{template: template}
}

func (f *factory) Block(channelID string) (*cb.Block, error) {
	configEnv, err := f.template.Envelope(channelID)
	configUpdate := &cb.ConfigUpdate{}
	err = proto.Unmarshal(configEnv.ConfigUpdate, configUpdate)

	payloadChannelHeader := utils.MakeChannelHeader(cb.HeaderType_CONFIG, msgVersion, channelID, epoch)
	payloadSignatureHeader := utils.MakeSignatureHeader(nil, utils.CreateNonceOrPanic())
	utils.SetTxID(payloadChannelHeader, payloadSignatureHeader)
	payloadHeader := utils.MakePayloadHeader(payloadChannelHeader, payloadSignatureHeader)
	payload := &cb.Payload{Header: payloadHeader, Data: utils.MarshalOrPanic(&cb.ConfigEnvelope{Config: &cb.Config{ChannelGroup: configUpdate.WriteSet}})}
	envelope := &cb.Envelope{Payload: utils.MarshalOrPanic(payload), Signature: nil}

	block := cb.NewBlock(0, nil)
	block.Data = &cb.BlockData{Data: [][]byte{utils.MarshalOrPanic(envelope)}}
	block.Header.DataHash = block.Data.Hash()
	block.Metadata.Metadata[cb.BlockMetadataIndex_LAST_CONFIG] = utils.MarshalOrPanic(&cb.Metadata{
		Value: utils.MarshalOrPanic(&cb.LastConfig{Index: 0}),
	})
	return block, nil
}

//代码在common/genesis/genesis.go
```


