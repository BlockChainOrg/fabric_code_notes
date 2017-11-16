# Fabric 1.0源码之旅(2)-BCCSP

Peer节点启动流程，涉及BCCSP，本文专门详解BCCSP。

## 1、BCCSP概要

BCCSP，全称Blockchain Cryptographic Service Provider，即区块链加密服务提供者，为Fabric提供加密标准和算法的实现，包括哈希、签名、校验、加解密等。
BCCSP通过MSP（即Membership Service Provider成员关系服务提供者）给核心功能和客户端SDK提供加密算法相关服务。
另外BCCSP支持可插拔，提供多种CSP，支持自定义CSP。目前支持sw和pkcs11两种实现。

代码在bccsp目录，bccsp主要目录结构如下：

* bccsp.go，主要是接口声明，定义了BCCSP和Key接口，以及众多Opts接口，如KeyGenOpts、KeyDerivOpts、KeyImportOpts、HashOpts、SignerOpts、EncrypterOpts和DecrypterOpts。
* keystore.go，定义了KeyStore接口，即Key的管理和存储接口。如果Key不是暂时的，则存储在实现了该接口的对象中，否则不存储。
* *opts.go，bccsp所使用到的各种技术选项的实现。
* [factory]目录，即bccsp工厂包，通过bccsp工厂返回bccsp实例，比如sw或pkcs11，如果自定义bccsp实现，也需加添加到factory中。
* [sw]目录，为the software-based implementation of the BCCSP，即基于软件的BCCSP实现，通过调用go原生支持的密码算法实现，并提供keystore来保存密钥。
* [pkcs11]目录，为bccsp的pkcs11实现，通过调用pkcs11接口实现相关加密操作，仅支持ecdsa、rsa以及aes算法，密码保存在pkcs11通过pin口令保护的数据库或者硬件设备中。
* [utils]目录，为工具函数包。
* [signer]目录，实现go crypto标准库的Signer接口。
补充：bccsp_test.go和mocks目录，可忽略。

## 2、接口定义

### 2.1、BCCSP接口定义

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

### 2.2、Opts接口定义

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
```

KeyImportOpts接口（导入选项）定义如下：

```go
//KeyImport(raw interface{}, opts KeyImportOpts) (k Key, err error)
type KeyImportOpts interface {
	Algorithm() string //获取密钥导入算法标识符
	Ephemeral() bool //要生成的密钥是否为暂时的
}
//代码在bccsp/bccsp.go
```

HashOpts接口（哈希选项）定义如下：

```go
//Hash(msg []byte, opts HashOpts) (hash []byte, err error)
type HashOpts interface {
	Algorithm() string //获取哈希算法标识符
}
//代码在bccsp/bccsp.go
```

SignerOpts接口（签名选项）定义如下：

```go
//Sign(k Key, digest []byte, opts SignerOpts) (signature []byte, err error)
//即go标准库crypto.SignerOpts接口
type SignerOpts interface {
	crypto.SignerOpts
}
//代码在bccsp/bccsp.go
```

另外EncrypterOpts接口（加密选项）和DecrypterOpts接口（解密选项）均为空接口。

```go
type EncrypterOpts interface{}
type DecrypterOpts interface{}
//代码在bccsp/bccsp.go
```

## 3、SW实现方式

### 3.1、sw目录结构

SW实现方式是默认实现方式，代码在bccsp/sw。主要目录结构如下：

* impl.go，bccsp的SW实现。
* internals.go，签名者、校验者、加密者、解密者等接口定义，包括：KeyGenerator、KeyDeriver、KeyImporter、Hasher、Signer、Verifier、Encryptor和Decryptor。
* conf.go，bccsp的sw实现的配置定义。
------
* aes.go，AES类型的加密者（aescbcpkcs7Encryptor）和解密者（aescbcpkcs7Decryptor）接口实现。AES为一种对称加密算法。
* ecdsa.go，ECDSA类型的签名者（ecdsaSigner）和校验者（ecdsaPrivateKeyVerifier和ecdsaPublicKeyKeyVerifier）接口实现。ECDSA即椭圆曲线算法。
* rsa.go，RSA类型的签名者（rsaSigner）和校验者（rsaPrivateKeyVerifier和rsaPublicKeyKeyVerifier）接口实现。RSA为另一种非对称加密算法。
------
* aeskey.go，AES类型的Key接口实现。
* ecdsakey.go，ECDSA类型的Key接口实现，包括ecdsaPrivateKey和ecdsaPublicKey。
* rsakey.go，RSA类型的Key接口实现，包括rsaPrivateKey和rsaPublicKey。
------
* dummyks.go，dummy类型的KeyStore接口实现，即dummyKeyStore，用于暂时性的Key，保存在内存中，系统关闭即消失。
* fileks.go，file类型的KeyStore接口实现，即fileBasedKeyStore，用于长期的Key，保存在文件中。
------
* keygen.go，KeyGenerator接口实现，包括aesKeyGenerator、ecdsaKeyGenerator和rsaKeyGenerator。
* keyderiv.go，KeyDeriver接口实现，包括aesPrivateKeyKeyDeriver、ecdsaPrivateKeyKeyDeriver和ecdsaPublicKeyKeyDeriver。
* keyimport.go，KeyImporter接口实现，包括aes256ImportKeyOptsKeyImporter、ecdsaPKIXPublicKeyImportOptsKeyImporter、ecdsaPrivateKeyImportOptsKeyImporter、
　　ecdsaGoPublicKeyImportOptsKeyImporter、rsaGoPublicKeyImportOptsKeyImporter、hmacImportKeyOptsKeyImporter和x509PublicKeyImportOptsKeyImporter。
* hash.go，Hasher接口实现，即hasher。

### 3.2、SW bccsp配置

即代码bccsp/sw/conf.go，config数据结构定义：
elliptic.Curve为椭圆曲线接口，使用了crypto/elliptic包。有关椭圆曲线，参考http://8btc.com/thread-1240-1-1.html。
SHA，全称Secure Hash Algorithm，即安全哈希算法，参考https://www.cnblogs.com/kabi/p/5871421.html。

```go
type config struct {
	ellipticCurve elliptic.Curve //指定椭圆曲线，elliptic.P256()和elliptic.P384()分别为P-256曲线和P-384曲线
	hashFunction  func() hash.Hash //指定哈希函数，如SHA-2（SHA-256、SHA-384、SHA-512等）和SHA-3
	aesBitLength  int //指定AES密钥长度
	rsaBitLength  int //指定RSA密钥长度
}
//代码在bccsp/sw/conf.go
```

func (conf *config) setSecurityLevel(securityLevel int, hashFamily string) (err error)为设置安全级别和哈希系列（包括SHA2和SHA3）。
如果hashFamily为"SHA2"或"SHA3"，将分别调取conf.setSecurityLevelSHA2(securityLevel)或conf.setSecurityLevelSHA3(securityLevel)。

func (conf *config) setSecurityLevelSHA2(level int) (err error)代码如下：

```go
switch level {
case 256:
	conf.ellipticCurve = elliptic.P256() //P-256曲线
	conf.hashFunction = sha256.New //SHA-256
	conf.rsaBitLength = 2048 //指定AES密钥长度2048
	conf.aesBitLength = 32 //指定RSA密钥长度32
case 384:
	conf.ellipticCurve = elliptic.P384() //P-384曲线
	conf.hashFunction = sha512.New384 //SHA-384
	conf.rsaBitLength = 3072 //指定AES密钥长度3072
	conf.aesBitLength = 32 //指定RSA密钥长度32
//...
}
//代码在bccsp/sw/conf.go
```

func (conf *config) setSecurityLevelSHA3(level int) (err error)代码如下：

```go
switch level {
case 256:
	conf.ellipticCurve = elliptic.P256() //P-256曲线
	conf.hashFunction = sha3.New256 //SHA3-256
	conf.rsaBitLength = 2048 //指定AES密钥长度2048
	conf.aesBitLength = 32 //指定RSA密钥长度32
case 384:
	conf.ellipticCurve = elliptic.P384() //P-384曲线
	conf.hashFunction = sha3.New384 //SHA3-384
	conf.rsaBitLength = 3072 //指定AES密钥长度3072
	conf.aesBitLength = 32 //指定RSA密钥长度32
//...
}
//代码在bccsp/sw/conf.go
```

### 3.3、SW bccsp实例结构体定义

```go
type impl struct {
	conf *config //bccsp实例的配置
	ks   bccsp.KeyStore //KeyStore对象，用于存储和获取Key

	keyGenerators map[reflect.Type]KeyGenerator //KeyGenerator映射
	keyDerivers   map[reflect.Type]KeyDeriver //KeyDeriver映射
	keyImporters  map[reflect.Type]KeyImporter //KeyImporter映射
	encryptors    map[reflect.Type]Encryptor //加密者映射
	decryptors    map[reflect.Type]Decryptor //解密者映射
	signers       map[reflect.Type]Signer //签名者映射
	verifiers     map[reflect.Type]Verifier //校验者映射
	hashers       map[reflect.Type]Hasher //Hasher映射
}
//代码在bccsp/sw/impl.go
```

涉及如下方法： 

```go
func New(securityLevel int, hashFamily string, keyStore bccsp.KeyStore) (bccsp.BCCSP, error) //生成sw实例
func (csp *impl) KeyGen(opts bccsp.KeyGenOpts) (k bccsp.Key, err error) //生成Key
func (csp *impl) KeyDeriv(k bccsp.Key, opts bccsp.KeyDerivOpts) (dk bccsp.Key, err error) //派生Key
func (csp *impl) KeyImport(raw interface{}, opts bccsp.KeyImportOpts) (k bccsp.Key, err error) //导入Key
func (csp *impl) GetKey(ski []byte) (k bccsp.Key, err error) //获取Key
func (csp *impl) Hash(msg []byte, opts bccsp.HashOpts) (digest []byte, err error) //哈希msg
func (csp *impl) GetHash(opts bccsp.HashOpts) (h hash.Hash, err error) //获取哈希实例
func (csp *impl) Sign(k bccsp.Key, digest []byte, opts bccsp.SignerOpts) (signature []byte, err error) //签名
func (csp *impl) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error) //校验签名
func (csp *impl) Encrypt(k bccsp.Key, plaintext []byte, opts bccsp.EncrypterOpts) (ciphertext []byte, err error) //加密
func (csp *impl) Decrypt(k bccsp.Key, ciphertext []byte, opts bccsp.DecrypterOpts) (plaintext []byte, err error) //解密
//代码在bccsp/sw/impl.go
```

func New(securityLevel int, hashFamily string, keyStore bccsp.KeyStore) (bccsp.BCCSP, error)作用为：
设置securityLevel和hashFamily，设置keyStore、encryptors、decryptors、signers、verifiers和hashers，之后设置keyGenerators、keyDerivers和keyImporters。

func (csp *impl) KeyGen(opts bccsp.KeyGenOpts) (k bccsp.Key, err error)作用为：
按opts查找keyGenerator是否在csp.keyGenerators[]中，如果在则调取keyGenerator.KeyGen(opts)生成Key。如果opts.Ephemeral()不是暂时的，调取csp.ks.StoreKey存储Key。

func (csp *impl) KeyDeriv(k bccsp.Key, opts bccsp.KeyDerivOpts) (dk bccsp.Key, err error)作用为：
按k的类型查找keyDeriver是否在csp.keyDerivers[]中，如果在则调取keyDeriver.KeyDeriv(k, opts)派生Key。如果opts.Ephemeral()不是暂时的，调取csp.ks.StoreKey存储Key。

```go
func (csp *impl) KeyImport(raw interface{}, opts bccsp.KeyImportOpts) (k bccsp.Key, err error)
func (csp *impl) Hash(msg []byte, opts bccsp.HashOpts) (digest []byte, err error)
func (csp *impl) GetHash(opts bccsp.HashOpts) (h hash.Hash, err error)
func (csp *impl) Sign(k bccsp.Key, digest []byte, opts bccsp.SignerOpts) (signature []byte, err error)
func (csp *impl) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error)
func (csp *impl) Encrypt(k bccsp.Key, plaintext []byte, opts bccsp.EncrypterOpts) (ciphertext []byte, err error)
func (csp *impl) Decrypt(k bccsp.Key, ciphertext []byte, opts bccsp.DecrypterOpts) (plaintext []byte, err error)
//与上述方法实现方式相似。
```

func (csp *impl) GetKey(ski []byte) (k bccsp.Key, err error)作用为：按ski调取csp.ks.GetKey(ski)获取Key。

### 3.4、AES算法相关代码实现

参考：https://studygolang.com/articles/7302。
AES，Advanced Encryption Standard，即高级加密标准，是一种对称加密算法。
AES属于块密码工作模式。块密码工作模式，允许使用同一个密码块对于多于一块的数据进行加密。
块密码只能加密长度等于密码块长度的单块数据，若要加密变长数据，则数据必须先划分为一些单独的数据块。
通常而言最后一块数据，也需要使用合适的填充方式将数据扩展到符合密码块大小的长度。

Fabric中使用的填充方式为：pkcs7Padding，即填充字符串由一个字节序列组成，每个字节填充该字节序列的长度。 代码如下：
另外pkcs7UnPadding为其反操作。

```go
func pkcs7Padding(src []byte) []byte {
	padding := aes.BlockSize - len(src)%aes.BlockSize //计算填充长度
	padtext := bytes.Repeat([]byte{byte(padding)}, padding) //bytes.Repeat构建长度为padding的字节序列，内容为padding
	return append(src, padtext...)
}
//代码在bccsp/sw/aes.go
```

AES常见模式有ECB、CBC等。其中ECB，对于相同的数据块都会加密为相同的密文块，这种模式不能提供严格的数据保密性。
而CBC模式，每个数据块都会和前一个密文块异或后再加密，这种模式中每个密文块都会依赖前一个数据块。同时为了保证每条消息的唯一性，在第一块中需要使用初始化向量。
Fabric使用了CBC模式，代码如下：

```go
//AES加密
func aesCBCEncrypt(key, s []byte) ([]byte, error) {
	block, err := aes.NewCipher(key) //生成加密块

	//随机一个块大小作为初始化向量
	ciphertext := make([]byte, aes.BlockSize+len(s))
	iv := ciphertext[:aes.BlockSize]
	if _, err := io.ReadFull(rand.Reader, iv); err != nil {
		return nil, err
	}

	mode := cipher.NewCBCEncrypter(block, iv) //创建CBC模式加密器
	mode.CryptBlocks(ciphertext[aes.BlockSize:], s) //执行加密操作

	return ciphertext, nil
}
//代码在bccsp/sw/aes.go
```

```go
//AES解密
func aesCBCDecrypt(key, src []byte) ([]byte, error) {
	block, err := aes.NewCipher(key) //生成加密块

	iv := src[:aes.BlockSize] //初始化向量
	src = src[aes.BlockSize:] //实际数据

	mode := cipher.NewCBCDecrypter(block, iv) //创建CBC模式解密器
	mode.CryptBlocks(src, src) //执行解密操作

	return src, nil
}
//代码在bccsp/sw/aes.go
```

pkcs7Padding和aesCBCEncrypt整合后代码如下：

```go
//AES加密
func AESCBCPKCS7Encrypt(key, src []byte) ([]byte, error) {
	tmp := pkcs7Padding(src)
	return aesCBCEncrypt(key, tmp)
}
//AES解密
func AESCBCPKCS7Decrypt(key, src []byte) ([]byte, error) {
	pt, err := aesCBCDecrypt(key, src)
	return pkcs7UnPadding(pt)
}
//代码在bccsp/sw/aes.go
```

### 3.5、RSA算法相关代码实现

签名相关代码如下：

```go
type rsaSigner struct{}
func (s *rsaSigner) Sign(k bccsp.Key, digest []byte, opts bccsp.SignerOpts) (signature []byte, err error) {
	//...
	return k.(*rsaPrivateKey).privKey.Sign(rand.Reader, digest, opts) //签名
}
//代码在bccsp/sw/rsa.go
```

校验签名相关代码如下：

```go
type rsaPrivateKeyVerifier struct{}
func (v *rsaPrivateKeyVerifier) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error) {
	/...
	rsa.VerifyPSS(&(k.(*rsaPrivateKey).privKey.PublicKey), (opts.(*rsa.PSSOptions)).Hash, digest, signature, opts.(*rsa.PSSOptions)) //验签
	/...	
}
```

```go
type rsaPublicKeyKeyVerifier struct{}
func (v *rsaPublicKeyKeyVerifier) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error) {
	/...
	err := rsa.VerifyPSS(k.(*rsaPublicKey).pubKey, (opts.(*rsa.PSSOptions)).Hash, digest, signature, opts.(*rsa.PSSOptions)) //验签
	/...
}
//代码在bccsp/sw/rsa.go
```

另附rsaPrivateKey和rsaPublicKey定义如下：

```go
type rsaPrivateKey struct {
	privKey *rsa.PrivateKey
}
type rsaPublicKey struct {
	pubKey *rsa.PublicKey
}
//代码在bccsp/sw/rsakey.go
```

### 3.6、椭圆曲线算法相关代码实现

代码在bccsp/sw/ecdsa.go
椭圆曲线算法，相关内容参考：[Fabric1.0源码之旅附录(1)-椭圆曲线算法](../elliptic_curve_algorithm/elliptic_curve_algorithm.md)

### 3.7、文件类型KeyStore接口实现

虚拟类型KeyStore接口实现dummyKeyStore，无任何实际操作，忽略。
文件类型KeyStore接口实现fileBasedKeyStore，数据结构定义如下：

```go
type fileBasedKeyStore struct {
	path string //路径
	readOnly bool //是否只读
	isOpen   bool //是否打开
	pwd []byte //密码
	m sync.Mutex //锁
}
//代码在bccsp/sw/fileks.go
```
fileBasedKeyStore是一个基于文件夹的密钥库，每个Key都存储在分散的文件中，文件名包含密钥的SKI。
密钥库可以用密码初始化，这个密码可以用于加密和解密存储密钥的文件。为了避免覆盖，密钥库可以设置为只读。


涉及方法如下：

```go
func NewFileBasedKeyStore(pwd []byte, path string, readOnly bool) (bccsp.KeyStore, error) //创建fileBasedKeyStore，并调用Init完成初始化
func (ks *fileBasedKeyStore) Init(pwd []byte, path string, readOnly bool) error //初始化路径、密码、是否只读，以及创建并打开KeyStore
func (ks *fileBasedKeyStore) ReadOnly() bool //密钥库是否只读，只读时StoreKey将失败
func (ks *fileBasedKeyStore) GetKey(ski []byte) (k bccsp.Key, err error) //如果SKI通过，返回Key。通过ski可以获取文件后缀，key、sk、pk分别为普通key、私钥、公钥
func (ks *fileBasedKeyStore) StoreKey(k bccsp.Key) (err error) //将Key存储到密钥库中
//代码在bccsp/sw/fileks.go
```

func (ks *fileBasedKeyStore) StoreKey(k bccsp.Key) (err error)代码如下：

```go
switch k.(type) {
case *ecdsaPrivateKey:
	kk := k.(*ecdsaPrivateKey)
	err = ks.storePrivateKey(hex.EncodeToString(k.SKI()), kk.privKey) //ECDSA私钥
case *ecdsaPublicKey:
	kk := k.(*ecdsaPublicKey)
	err = ks.storePublicKey(hex.EncodeToString(k.SKI()), kk.pubKey) //ECDSA公钥
case *rsaPrivateKey:
	kk := k.(*rsaPrivateKey)
	err = ks.storePrivateKey(hex.EncodeToString(k.SKI()), kk.privKey) //RSA私钥
case *rsaPublicKey:
	kk := k.(*rsaPublicKey)
	err = ks.storePublicKey(hex.EncodeToString(k.SKI()), kk.pubKey) //RSA公钥
case *aesPrivateKey:
	kk := k.(*aesPrivateKey)
	err = ks.storeKey(hex.EncodeToString(k.SKI()), kk.privKey) //AES私钥
//...
//代码在bccsp/sw/fileks.go
```

## 4、pkcs11实现方式

pkcs11包，即HSM基础的bccsp（the hsm-based BCCSP implementation），HSM是Hardware Security Modules，即硬件安全模块。
pckcs11是硬件基础的加密服务实现，sw是软件基础的加密服务实现。这个硬件基础的实现以 https://github.com/miekg/pkcs11 这个库为基础。

PKCS#11称为Cyptoki，定义了一套独立于技术的程序设计接口，USBKey安全应用需要实现的接口。
在密码系统中，PKCS#11是公钥加密标准（PKCS, Public-Key Cryptography Standards）中的一份子，由RSA实验室(RSA Laboratories)发布，它为加密令牌定义了一组平台无关的API ，如硬件安全模块和智能卡。
pkcs11包主要内容是PKCS11标准的实现及椭圆曲线算法中以low-S算法为主导的go实现。同时也通过利用RSA的一些特性和算法，丰富了PKCS11加密体系。

### 4.1、pkcs11目录结构

* impl.go，bccsp的pkcs11实现。
* conf.go，bccsp的pkcs11实现的配置定义，实现代码与sw的配置定义接近，即实现设置安全级别和哈希系列。
* pkcs11.go，以miekg/pkcs11包为基础，包装了各种pkcs11功能。
* ecdsa.go，ECDSA算法的签名和验签的实现。
* ecdsakey.go，ECDSA类型的Key接口实现，包括ecdsaPrivateKey和ecdsaPublicKey。

### 4.2、pkcs11实例结构体定义和实现

```go
type impl struct {
	bccsp.BCCSP //结构体中内嵌接口，参考https://studygolang.com/articles/6934

	conf *config //pkcs11实例的配置
	ks   bccsp.KeyStore //KeyStore对象，用于存储和获取Key

	ctx      *pkcs11.Ctx //pkcs11上下文
	sessions chan pkcs11.SessionHandle //即type SessionHandle uint，会话标识符通道，默认数量10
	slot     uint	//安全硬件外设连接插槽标识号

	lib          string	//pkcs11库文件所在路径
	noPrivImport bool	//是否禁止导入私钥
	softVerify   bool	//是否使用软件方式校验签名
}
//代码在bccsp/pkcs11/impl.go
```

涉及方法如下：

```go
func New(opts PKCS11Opts, keyStore bccsp.KeyStore) (bccsp.BCCSP, error) //生成pkcs11实例
func (csp *impl) KeyGen(opts bccsp.KeyGenOpts) (k bccsp.Key, err error) //生成Key
func (csp *impl) KeyDeriv(k bccsp.Key, opts bccsp.KeyDerivOpts) (dk bccsp.Key, err error) //派生Key
func (csp *impl) KeyImport(raw interface{}, opts bccsp.KeyImportOpts) (k bccsp.Key, err error) //导入Key
func (csp *impl) GetKey(ski []byte) (k bccsp.Key, err error) //获取Key
func (csp *impl) Sign(k bccsp.Key, digest []byte, opts bccsp.SignerOpts) (signature []byte, err error) //签名
func (csp *impl) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error) //校验签名
func (csp *impl) Encrypt(k bccsp.Key, plaintext []byte, opts bccsp.EncrypterOpts) (ciphertext []byte, err error) //加密
func (csp *impl) Decrypt(k bccsp.Key, ciphertext []byte, opts bccsp.DecrypterOpts) (plaintext []byte, err error) //解密
func FindPKCS11Lib() (lib, pin, label string) //从环境变量PKCS11_LIB、PKCS11_PIN、PKCS11_LABEL中获取lib、pin、label，否则取默认使用libsofthsm2.so、98765432、ForFabric
//代码在bccsp/pkcs11/impl.go
```

func New(opts PKCS11Opts, keyStore bccsp.KeyStore) (bccsp.BCCSP, error)核心代码如下：

```go
conf := &config{}
err := conf.setSecurityLevel(opts.SecLevel, opts.HashFamily) //初始化conf
swCSP, err := sw.New(opts.SecLevel, opts.HashFamily, keyStore) //创建sw实例

lib := opts.Library
pin := opts.Pin
label := opts.Label
ctx, slot, session, err := loadLib(lib, pin, label) //加载动态库，寻找slot，打开会话并登陆会话

sessions := make(chan pkcs11.SessionHandle, sessionCacheSize)
csp := &impl{swCSP, conf, keyStore, ctx, sessions, slot, lib, opts.Sensitive, opts.SoftVerify}
csp.returnSession(*session)
return csp, nil
//代码在bccsp/pkcs11/impl.go
```

loadLib(lib, pin, label)代码如下：
* pkcs11.New(lib)根据lib路径加载动态库（如openCryptoki的动态库），并建立pkcs11实例ctx。ctx相当于fabric与安全硬件模块通信的桥梁：bccsp<–>ctx<–>驱动lib<–>安全硬件模块，只要驱动lib是按照pkcs11标准开发。
* ctx.Initialize()进行初始化PKCS#11库。
* 从ctx.GetSlotList(true)返回的列表中获取由label指定的插槽标识slot。注：这里的槽可以简单的理解为电脑主机上供安全硬件模块插入的槽，如USB插口，可能不止一个，每一个在系统内核中都有名字和标识号。
* 尝试10次调用ctx.OpenSession打开一个会话session。会话就是通过通信路径与安全硬件模块建立连接，可以简单的理解为pkcs11的chan。
* 登陆会话ctx.Login。
* 返回ctx，slot，会话对象session，用于赋值给impl实例成员ctx，slot，把session发送到sessions里。
pkcs11库的使用参考：Cryptoki库概述https://docs.oracle.com/cd/E19253-01/819-7056/6n91eac56/index.html

```go
var slot uint = 0
ctx := pkcs11.New(lib) //根据lib路径加载动态库（如openCryptoki的动态库），并建立pkcs11实例ctx
ctx.Initialize() //初始化 PKCS #11 库
slots, err := ctx.GetSlotList(true) //可用插槽的列表

found := false
for _, s := range slots {
	info, err := ctx.GetTokenInfo(s) //获取有关特定令牌的信息
	if label == info.Label {
		found = true
		slot = s
		break
	}
}

var session pkcs11.SessionHandle
for i := 0; i < 10; i++ { //尝试10次调用ctx.OpenSession打开一个会话session
	session, err = ctx.OpenSession(slot, pkcs11.CKF_SERIAL_SESSION|pkcs11.CKF_RW_SESSION)
	if err != nil {
		//...
	} else {
		break
	}
}
err = ctx.Login(session, pkcs11.CKU_USER, pin) //登陆会话ctx.Login
return ctx, slot, &session, nil //返回ctx，slot，会话对象session
//代码在bccsp/pkcs11/pkcs11.go
```

补充type PKCS11Opts struct定义如下：

```go
type PKCS11Opts struct {
	//...
	//Keystore选项
	Ephemeral     bool //是否暂存的
	FileKeystore  *FileKeystoreOpts //FileKeystore
	DummyKeystore *DummyKeystoreOpts //DummyKeystore

	// PKCS11 options
	Library    string //库文件路径
	Label      string //插槽标识
	Pin        string //登录密码
	Sensitive  bool						
	SoftVerify bool						
}
//代码在bccsp/pkcs11/conf.go
```

如下方法优先判断Opts或Key类型，如果为pkcs11支持的ecdsa类型，将调取pkcs11包的实现，否则调取sw包作为默认实现。

```go
func (csp *impl) KeyGen(opts bccsp.KeyGenOpts) (k bccsp.Key, err error) //生成Key
func (csp *impl) KeyDeriv(k bccsp.Key, opts bccsp.KeyDerivOpts) (dk bccsp.Key, err error) //派生Key
func (csp *impl) KeyImport(raw interface{}, opts bccsp.KeyImportOpts) (k bccsp.Key, err error) //导入Key
func (csp *impl) GetKey(ski []byte) (k bccsp.Key, err error) //获取Key
func (csp *impl) Sign(k bccsp.Key, digest []byte, opts bccsp.SignerOpts) (signature []byte, err error) //签名
func (csp *impl) Verify(k bccsp.Key, signature, digest []byte, opts bccsp.SignerOpts) (valid bool, err error) //校验签名
//代码在bccsp/pkcs11/impl.go
```

如下加解密方法，将直接调取sw包的默认实现。

```go
func (csp *impl) Encrypt(k bccsp.Key, plaintext []byte, opts bccsp.EncrypterOpts) (ciphertext []byte, err error) //加密
func (csp *impl) Decrypt(k bccsp.Key, ciphertext []byte, opts bccsp.DecrypterOpts) (plaintext []byte, err error) //解密
//代码在bccsp/pkcs11/impl.go
```

### 4.3、pkcs11对第三方包github.com/miekg/pkcs11的封装

```go
func (csp *impl) getSession() (session pkcs11.SessionHandle) //在cache为空或者完全为使用状态的时候，通过OpenSession来获取session
func (csp *impl) returnSession(session pkcs11.SessionHandle) //关闭Session
func (csp *impl) getECKey(ski []byte) (pubKey *ecdsa.PublicKey, isPriv bool, err error) //通过SKI, 查找EC（椭圆曲线） key
func (csp *impl) generateECKey(curve asn1.ObjectIdentifier, ephemeral bool) (ski []byte, pubKey *ecdsa.PublicKey, err error) //生成EC key
func (csp *impl) signP11ECDSA(ski []byte, msg []byte) (R, S *big.Int, err error) //签名
func (csp *impl) verifyP11ECDSA(ski []byte, msg []byte, R, S *big.Int, byteSize int) (valid bool, err error) //校验签名
func (csp *impl) importECKey(curve asn1.ObjectIdentifier, privKey, ecPt []byte, ephemeral bool, keyType bool) (ski []byte, err error) //导入EC key
//loadLib 加载lib文件，初始化数据，通过GetSlotList来解析slot数据，通过GetTokenInfo获取token信息，通过pkcs11.SessionHandle方法来获取session
func loadLib(lib, pin, label string) (*pkcs11.Ctx, uint, *pkcs11.SessionHandle, error)
//findKeyPairFromSKI 通过Ctx、session及ski来获取对应的公私钥
func findKeyPairFromSKI(mod *pkcs11.Ctx, session pkcs11.SessionHandle, ski []byte, keyType bool) (*pkcs11.ObjectHandle, error)
//代码在bccsp/pkcs11/pkcs11.go
```

## 5、BCCSP工厂

通过factory可以获得两类BCCSP实例：sw和pkcs11。
BCCSP实例是通过工厂来提供的，sw包对应的工厂在swFactory.go中实现，pkcs11包对应的工厂在pkcs11Factory.go中实现，它们都共同实现了BCCSPFactory接口。

### 5.1、factory目录结构

* factory.go，定义BCCSPFactory接口，声明全局变量bccspMap来保存实例化的bccsp，声明bootBCCSP来保存缺省的实例，以及定义factory初始化函数和Get函数。
* nopkcs11.go/pkcs11.go，定义了两个版本的工厂选项FactoryOpts、初始化和Get函数。区别在于编译时是否指定nopkcs11或!nopkcs11，默认是nopkcs11。
* opts.go，定义了默认的FactoryOpts，即SW。
* pkcs11factory.go，pkcs11类型的bccsp工厂实现PKCS11Factory。
* swfactory.go， sw类型的bccsp工厂实现SWFactory。

### 5.2、BCCSPFactory接口定义

```go
type BCCSPFactory interface {
	Name() string //获取工厂名称
	Get(opts *FactoryOpts) (bccsp.BCCSP, error) //使用FactoryOpts获取BCCSP实例
}
//代码在bccsp/factory/factory.go
```

nopkcs11的FactoryOpts：

```go
type FactoryOpts struct {
	ProviderName string
	SwOpts       *SwOpts
}
//代码在bccsp/factory/nopkcs11.go
```

pkcs11的FactoryOpts：

```go
type FactoryOpts struct {
	ProviderName string
	SwOpts       *SwOpts
	Pkcs11Opts   *pkcs11.PKCS11Opts
}
//代码在bccsp/factory/pkcs11.go
```

### 5.3、swfactory实现

type SWFactory struct{}

涉及方法：

```go
func (f *SWFactory) Name() string //此处返回SoftwareBasedFactoryName，即"SW"
func (f *SWFactory) Get(config *FactoryOpts) (bccsp.BCCSP, error) //使用FactoryOpts获取BCCSP实例
//代码在bccsp/factory/swfactory.go
```

func (f *SWFactory) Get(config *FactoryOpts) (bccsp.BCCSP, error)代码如下：

```go
swOpts := config.SwOpts
var ks bccsp.KeyStore
if swOpts.Ephemeral == true { //密钥是暂时的
	ks = sw.NewDummyKeyStore()
} else if swOpts.FileKeystore != nil { //密钥是永久的并且定义了FileKeystore
	fks, err := sw.NewFileBasedKeyStore(nil, swOpts.FileKeystore.KeyStorePath, false)
	ks = fks
} else {
	ks = sw.NewDummyKeyStore() //默认是暂时的
}
return sw.New(swOpts.SecLevel, swOpts.HashFamily, ks) //创建sw实例
//代码在bccsp/factory/swfactory.go
```

### 5.4、pkcs11factory实现

type PKCS11Factory struct{}

涉及方法：

```go
func (f *PKCS11Factory) Name() string //此处返回PKCS11BasedFactoryName，即"PKCS11"
func (f *PKCS11Factory) Get(config *FactoryOpts) (bccsp.BCCSP, error) //使用FactoryOpts获取BCCSP实例
//代码在bccsp/factory/pkcs11factory.go
```

func (f *PKCS11Factory) Get(config *FactoryOpts) (bccsp.BCCSP, error)代码如下：
代码结构基本与swfactory相同，但此处有一个TODO提示。即：
PKCS11是不需要密钥库（keystore）的，但目前还没有从PKCS11 BCCSP中拆分出去，所以这里留着待后续进行改进，因此代码实现中依然保留了一部分keystore的实现。 

```go
p11Opts := config.Pkcs11Opts
//TODO: PKCS11 does not need a keystore, but we have not migrated all of PKCS11 BCCSP to PKCS11 yet
var ks bccsp.KeyStore
if p11Opts.Ephemeral == true {
	ks = sw.NewDummyKeyStore()
} else if p11Opts.FileKeystore != nil {
	fks, err := sw.NewFileBasedKeyStore(nil, p11Opts.FileKeystore.KeyStorePath, false)
	ks = fks
} else {
	ks = sw.NewDummyKeyStore()
}
return pkcs11.New(*p11Opts, ks)
//代码在bccsp/factory/pkcs11factory.go
```

### 5.5、Factories初始化

nopkcs11版本Factories初始化：

```go
factoriesInitOnce.Do(func() { //仅执行一次
	bccspMap = make(map[string]bccsp.BCCSP) //初始化全局bccsp.BCCSP map：bccspMap

	// Software-Based BCCSP
	if config.SwOpts != nil {
		f := &SWFactory{} //创建SWFactory
		err := initBCCSP(f, config) //创建BCCSP实例，即调用f.Get，并加入bccspMap中
	}

	var ok bool
	defaultBCCSP, ok = bccspMap[config.ProviderName] //将其作为默认defaultBCCSP
})
//代码在bccsp/factory/nopkcs11.go
```

pkcs11版本Factories初始化：

```go
func InitFactories(config *FactoryOpts) error {
	factoriesInitOnce.Do(func() {
		setFactories(config)
	})
}

func setFactories(config *FactoryOpts) error {
	bccspMap = make(map[string]bccsp.BCCSP)

	// Software-Based BCCSP，如果是SW
	if config.SwOpts != nil {
		f := &SWFactory{}
		err := initBCCSP(f, config)
	}

	// PKCS11-Based BCCSP，如果是PKCS11
	if config.Pkcs11Opts != nil {
		f := &PKCS11Factory{}
		err := initBCCSP(f, config)
	}

	var ok bool
	defaultBCCSP, ok = bccspMap[config.ProviderName]
}
//代码在bccsp/factory/pkcs11.go
```

func initBCCSP(f BCCSPFactory, config *FactoryOpts) error代码如下：

```go
csp, err := f.Get(config) //调取f.Get生成BCCSP实例
bccspMap[f.Name()] = csp //新生成的实例，加入bccspMap中
//代码在bccsp/factory/factory.go
```

## 6、本文使用到的网络内容

* [fabric源码解析13——peer的BCCSP服务](http://blog.csdn.net/idsuf698987/article/details/77200287)
* [[区块链]Hyperledger Fabric源代码（基于v1.0 beta版本）阅读之乐扣老师解读系列 （三）BCCSP包之工厂包](http://blog.csdn.net/lsttoy/article/details/73278445)
* [[区块链]Hyperledger Fabric源代码（基于v1.0 beta版本）阅读之乐扣老师解读系列 （四）BSSCP包之pkcs11加密包](http://blog.csdn.net/lsttoy/article/details/73292182)
* [[区块链]Hyperledger Fabric源代码（基于v1.0 beta版本）阅读之乐扣老师解读系列 （五）BSSCP包之SW加密包](http://blog.csdn.net/lsttoy/article/details/73322148)
* [[区块链]Hyperledger Fabric源代码（基于v1.0 beta版本）阅读之乐扣老师解读系列 （六）BSSCP包之UTILS工具包](http://blog.csdn.net/lsttoy/article/details/73459950)
* [Hyperledger Fabric密码模块系列之BCCSP（一）](http://www.cnblogs.com/informatics/p/7522445.html)
* [Hyperledger Fabric密码模块系列之BCCSP（二）](http://www.cnblogs.com/informatics/p/7572969.html)
* [Hyperledger Fabric密码模块系列之BCCSP（三）](http://www.cnblogs.com/informatics/p/7604461.html)
* [Hyperledger Fabric密码模块系列之BCCSP（四）](http://www.cnblogs.com/informatics/p/7604470.html)
* [Hyperledger Fabric密码模块系列之BCCSP（五） - 国密算法实现](http://www.cnblogs.com/informatics/p/7648039.html)
* [【翻译】BCCSP密码算法套件解析](http://www.blockchainbrother.com/article/31)
