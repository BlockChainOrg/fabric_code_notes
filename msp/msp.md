# Fabric 1.0源码之旅(3)-MSP

Peer节点启动流程，涉及MSP，本文专门讲解MSP。

## 1、MSP概要

MSP，全称Membership Service Provider，即成员关系服务提供者，作用为管理Fabric中的众多参与者。
MSP的核心代码在msp目录下，其他相关代码分布在common/config/msp、protos/msp下。目录结构如下：

* msp目录
	* msp.go，定义接口MSP、MSPManager、Identity、SigningIdentity、IdentityDeserializer。
	* mspimpl.go，实现MSP接口，即bccspmsp。
	* mspmgrimpl.go，实现MSPManager接口，即mspManagerImpl。
	* identities.go，实现Identity、SigningIdentity接口，即identity和signingidentity。
	* configbuilder.go，提供读取证书文件并将其组装成MSP等接口所需的数据结构，以及转换配置结构体（FactoryOpts->MSPConfig）等工具函数。
	* cert.go，证书相关结构体及方法。
	* mgmt目录
		* mgmt.go，msp相关管理方法实现。
		* principal.go，MSPPrincipalGetter接口及其实现，即localMSPPrincipalGetter。
		* deserializer.go，DeserializersManager接口及其实现，即mspDeserializersManager。
* common/config/msp目录
	* config.go，定义了MSPConfigHandler及其方法，用于配置MSP和configtx工具。
* protos/msp目录，msp相关Protocol Buffer原型文件。



## 6、本文使用到的网络内容

* [fabric源码解析12——peer的MSP服务](http://blog.csdn.net/idsuf698987/article/details/77103011)