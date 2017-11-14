# Fabric源码之旅(2)-BCCSP
Peer节点启动流程，涉及BCCSP，本文专门详解BCCSP。
## 1、BCCSP概要
BCCSP，全称Blockchain Cryptographic Service Provider，即区块链加密服务提供者，为Fabric提供加密标准和算法的实现，包括哈希、签名、校验、加解密等。
BCCSP通过MSP（即Membership Service Provider成员关系服务提供者）给核心功能和客户端SDK提供加密算法相关服务。
另外BCCSP支持可插拔，提供多种CSP，支持自定义CSP。目前支持sw和pkcs11两种实现。

代码在bccsp目录，bccsp主要目录结构如下：
bccsp.go，主要是接口声明，定义了BCCSP和Key接口，以及众多Opts接口，如KeyGenOpts、KeyDerivOpts、KeyImportOpts、HashOpts、SignerOpts、EncrypterOpts和DecrypterOpts。
keystore.go，定义了KeyStore接口，即Key的管理和存储接口。如果Key不是暂时的，则存储在实现了该接口的对象中，否则不存储。
*opts.go，bccsp所使用到的各种技术选项的实现。
[factory]目录，即bccsp工厂包，通过bccsp工厂返回bccsp实例，比如sw或pkcs11，如果自定义bccsp实现，也需加添加到factory中。
[sw]目录，为the software-based implementation of the BCCSP，即基于软件的BCCSP实现，通过调用go原生支持的密码算法实现，并提供keystore来保存密钥。
[pkcs11]目录，为bccsp的pkcs11实现，通过调用pkcs11接口实现相关加密操作，仅支持ecdsa、rsa以及aes算法，密码保存在pkcs11通过pin口令保护的数据库或者硬件设备中。
[utils]目录，为工具函数包。
[signer]目录，实现go crypto标准库的Signer接口。
补充：bccsp_test.go和[mocks]目录，可忽略。
## 2、BCCSP接口定义
BCCSP接口（区块链加密服务提供者）定义如下：
```go
type BCCSP interface {
	KeyGen(opts KeyGenOpts) (k Key, err error) //生成Key
	KeyDeriv(k Key, opts KeyDerivOpts) (dk Key, err error) //派生Key
	KeyImport(raw interface{}, opts KeyImportOpts) (k Key, err error) //导入Key
	GetKey(ski []byte) (k Key, err error) //获取Key
	Hash(msg []byte, opts HashOpts) (hash []byte, err error) //哈希msg
	GetHash(opts HashOpts) (h hash.Hash, err error) //获取哈希实例
	Sign(k Key, digest []byte, opts SignerOpts) (signature []byte, err error) //签名
	Verify(k Key, signature, digest []byte, opts SignerOpts) (valid bool, err error) //校验签名
	Encrypt(k Key, plaintext []byte, opts EncrypterOpts) (ciphertext []byte, err error) //加密
	Decrypt(k Key, ciphertext []byte, opts DecrypterOpts) (plaintext []byte, err error) //解密
}
//代码在bccsp/bccsp.go
```
Key接口（密钥）定义如下：
```go
type Key interface {
	Bytes() ([]byte, error) //Key转换成字节形式
	SKI() []byte //SKI，全称Subject Key Identifier，主题密钥标识符
	Symmetric() bool //是否对称密钥，是为true，否则为false
	Private() bool //是否为私钥，是为true，否则为false
	PublicKey() (Key, error) //返回非对称密钥中的公钥，如果为对称密钥则返回错误
}
//代码在bccsp/bccsp.go
```
KeyStore接口（密钥存储）定义如下：
```go
type KeyStore interface {
	ReadOnly() bool //密钥库是否只读，只读时StoreKey将失败
	GetKey(ski []byte) (k Key, err error) //如果SKI通过，返回Key
　　StoreKey(k Key) (err error) //将Key存储到密钥库中
}
//代码在bccsp/keystore.go
```
## 3、Opts接口定义
KeyGenOpts接口（密钥生成选项）定义如下：
```go
//KeyGen(opts KeyGenOpts) (k Key, err error)
type KeyGenOpts interface {
	Algorithm() string //获取密钥生成算法的标识符
	Ephemeral() bool //要生成的密钥是否为暂时的，如果为长期密钥，需要通过SKI来完成存储和索引
}
//代码在bccsp/bccsp.go
```
KeyDerivOpts接口（密钥派生选项）定义如下：
```go
//KeyDeriv(k Key, opts KeyDerivOpts) (dk Key, err error)
type KeyDerivOpts interface {
	Algorithm() string //获取密钥派生算法标识符
	Ephemeral() bool //要派生的密钥是否为暂时的
}
//代码在bccsp/bccsp.go
KeyImportOpts接口（导入选项）定义如下：
```go
//KeyImport(raw interface{}, opts KeyImportOpts) (k Key, err error)
type KeyImportOpts interface {
	Algorithm() string //获取密钥导入算法标识符
	Ephemeral() bool //要生成的密钥是否为暂时的
}
//代码在bccsp/bccsp.go
```
