# Fabric 1.0源码旅程 之 附录-ECDSA（椭圆曲线数字签名算法）

## 1、椭圆曲线算法概论

### 1.1、无穷远点、无穷远直线、射影平面

* 平行线相交于无穷远点；
* 直线上有且只有一个无穷远点；
* 一组相互平行的直线有公共的无穷远点；
* 平面上任何相交的两直线，有不同的无穷远点；
* 全部无穷远点沟通一条无穷远直线；
* 平面上全部无穷远点和全部普通点构成射影平面。

### 1.2、射影平面点定义

对于普通平面上点(x, y)，令x=X/Z，y=Y/Z，Z≠0，则投影为射影平面上的点为(X : Y : Z)。
如点(1，2)在射影平面的坐标为：(Z : 2Z : Z) Z≠0，即(1 : 2 : 1)或(2 : 4 : 2)均为(1, 2)在射影平面上的点。
Z=0时，(X : Y : 0)即为无穷远点，Z=0即为无穷远直线。

### 1.3、椭圆曲线方程

椭圆曲线的定义：
一条椭圆曲线是在射影平面上满足方程Y²Z+a1XYZ+a3YZ²=X³+a2X²Z+a4XZ²+a6Z³的所有点的集合，且曲线上的每个点都是非奇异（或光滑）的。
该方程为维尔斯特拉斯方程，是一个齐次方程。
所谓“非奇异”或“光滑”的，即满足方程的任意一点都存在切线。

椭圆曲线存在无穷远点(0, Y, 0)，可以在平面坐标系中用椭圆曲线、加一个无穷远点来表示。
令x=X/Z，y=Y/Z，代入椭圆曲线方程，即椭圆曲线普通方程：y²+a1xy+a3y = x³+a2x²+a4x+a6。

### 1.4、椭圆曲线上的加法

任意取椭圆曲线上两点P、Q （若P、Q两点重合，则做P点的切线）做直线交于椭圆曲线的另一点R’，过R’做y轴的平行线交于R。我们规定P+Q=R。

根据这个法则，可以知道椭圆曲线无穷远点O∞与椭圆曲线上一点P的连线交于P’，过P’作y轴的平行线交于P，所以有 无穷远点 O∞+ P = P 。
这样，无穷远点 O∞的作用与普通加法中零的作用相当（0+2=2），我们把无穷远点 O∞ 称为 零元。同时我们把P’称为P的负元（简称，负P；记作，-P）。

根据这个法则，可以得到如下结论 ：如果椭圆曲线上的三个点A、B、C，处于同一条直线上，那么他们的和等于零元，即A+B+C= O∞ 。
k个相同的点P相加，我们记作kP。如：P+P+P = 2P+P = 3P。

### 1.5、有限域椭圆曲线

椭圆曲线是连续的，并不适合用于加密；所以，我们必须把椭圆曲线变成离散的点，我们要把椭圆曲线定义在有限域上。
* 我们给出一个有限域Fp
* Fp中有p（p为质数）个元素0,1,2,…, p-2,p-1
* Fp的加法是a+b≡c(mod p)
* Fp的乘法是a×b≡c(mod p)
* Fp的除法是a÷b≡c(mod p)，即 a×b^(-1)≡c (mod p)，b^(-1)也是一个0到p-1之间的整数，但满足b×b^(-1)≡1 (mod p)
* Fp的单位元是1，零元是0

同时，并不是所有的椭圆曲线都适合加密。y²=x³+ax+b是一类可以用来加密的椭圆曲线，也是最为简单的一类。
下面我们就把y²=x³+ax+b这条曲线定义在Fp上：

选择两个满足下列条件的小于p(p为素数)的非负整数a、b，4a³+27b²≠0　(mod p) 。
则满足下列方程的所有点(x,y)，再加上 无穷远点O∞ ，构成一条椭圆曲线。 
y²=x³+ax+b  (mod p) 其中 x,y属于0到p-1间的整数，并将这条椭圆曲线记为Ep(a,b)。

Fp上的椭圆曲线同样有加法，但已经不能给以几何意义的解释。

```
无穷远点 O∞是零元，有O∞+ O∞= O∞，O∞+P=P 
P(x,y)的负元是 (x,-y)，有P+(-P)= O∞ 
P(x1,y1),Q(x2,y2)的和R(x3,y3) 有如下关系： 
　　x3≡k2-x1-x2(mod p) 
　　y3≡k(x1-x3)-y1(mod p) 
　　其中若P=Q 则 k=(3x1²+a)/2y1  若P≠Q，则k=(y2-y1)/(x2-x1)
```

例 已知E23(1,1)上两点P(3,10)，Q(9,7)，求1)-P，2)P+Q，3) 2P。

```
1)  –P的值为(3,-10) 
2)  k=(7-10)/(9-3)=-1/2，2的乘法逆元为12 因为2*12≡1 (mod 23) 
	k≡-1*12 (mod 23) 故 k=11。 
	x=112-3-9=109≡17 (mod 23); 
	y=11[3-(-6)]-10=89≡20 (mod 23) 
	故P+Q的坐标为(17,20) 
3)  k=[3(3²)+1]/(2*10)=1/4≡6 (mod 23) 
	x=62-3-3=30≡20 (mod 23) 
	y=6(3-7)-10=-34≡12 (mod 23) 
	故2P的坐标为(7,12) 
```

如果椭圆曲线上一点P，存在最小的正整数n，使得数乘nP=O∞，则将n称为P的阶，若n不存在，我们说P是无限阶的。 
事实上，在有限域上定义的椭圆曲线上所有的点的阶n都是存在的。

### 1.6、椭圆曲线上简单的加密/解密

考虑如下等式：
K=kG  [其中 K,G为Ep(a,b)上的点，k为小于n（n是点G的阶）的整数]。
不难发现，给定k和G，根据加法法则，计算K很容易；但给定K和G，求k就相对困难了。
这就是椭圆曲线加密算法采用的难题。
我们把点G称为基点（base point），k（k<n，n为基点G的阶）称为私有密钥（privte key），K称为公开密钥（public key)。

补充：如上内容参考自
* [【信息安全】ECC加密算法入门介绍](https://yq.aliyun.com/articles/23897)
* [ECC椭圆曲线详解(有具体实例)](https://yq.aliyun.com/articles/23897)

## 2、椭圆曲线接口及实现

### 2.1、椭圆曲线接口定义

```go
type Curve interface {
	Params() *CurveParams //返回曲线参数
	IsOnCurve(x, y *big.Int) bool //(x,y)是否在曲线上
	Add(x1, y1, x2, y2 *big.Int) (x, y *big.Int) //(x1,y1)和(x2,y2)求和
	Double(x1, y1 *big.Int) (x, y *big.Int) //2*(x,y)
	ScalarMult(x1, y1 *big.Int, k []byte) (x, y *big.Int) //返回k*(Bx,By)
	ScalarBaseMult(k []byte) (x, y *big.Int) //返回k*G，其中G为基点
}
//代码在crypto/elliptic/elliptic.go
```

### 2.2、CurveParams结构体定义及通用实现

CurveParams包括椭圆曲线的参数，并提供了一个通用椭圆曲线实现。代码如下：

```go
type CurveParams struct {
	P       *big.Int //% p中的p
	N       *big.Int //基点的阶，如果椭圆曲线上一点P，存在最小的正整数n，使得数乘nP=O∞，则将n称为P的阶
	B       *big.Int //曲线方程中常数b，如y² = x³ - 3x + b
	Gx, Gy  *big.Int //基点G(x,y)
	BitSize int      //基础字段的大小
	Name    string   //椭圆曲线的名称
}
//代码在crypto/elliptic/elliptic.go
```

CurveParams涉及如下方法：

```go
func (curve *CurveParams) Params() *CurveParams //返回曲线参数，即curve
func (curve *CurveParams) IsOnCurve(x, y *big.Int) bool //(x,y)是否在曲线上
func (curve *CurveParams) Add(x1, y1, x2, y2 *big.Int) (*big.Int, *big.Int) //(x1,y1)和(x2,y2)求和
func (curve *CurveParams) Double(x1, y1 *big.Int) (*big.Int, *big.Int) //2*(x,y)
func (curve *CurveParams) ScalarMult(Bx, By *big.Int, k []byte) (*big.Int, *big.Int) //返回k*(Bx,By)
func (curve *CurveParams) ScalarBaseMult(k []byte) (*big.Int, *big.Int) //返回k*G，其中G为基点
//代码在crypto/elliptic/elliptic.go
```

### 2.3、几种曲线

```go
func P224() Curve //实现了P-224的曲线
func P256() Curve //实现了P-256的曲线
func P384() Curve //实现了P-384的曲线
func P521() Curve //实现了P-512的曲线
//代码在crypto/elliptic/elliptic.go
```

## 3、椭圆曲线数字签名算法

结构体定义：
```go
type PublicKey struct { //公钥
	elliptic.Curve
	X, Y *big.Int
}
type PrivateKey struct { //私钥
	PublicKey
	D *big.Int
}
type ecdsaSignature struct { //椭圆曲线签名
	R, S *big.Int
}
//代码在crypto/ecdsa/ecdsa.go
```

涉及如下方法：

```go
func (priv *PrivateKey) Public() crypto.PublicKey //获取公钥
func (priv *PrivateKey) Sign(rand io.Reader, msg []byte, opts crypto.SignerOpts) ([]byte, error) //使用私钥对任意长度的hash值进行签名
func GenerateKey(c elliptic.Curve, rand io.Reader) (*PrivateKey, error) //生成一对公钥/私钥
func Sign(rand io.Reader, priv *PrivateKey, hash []byte) (r, s *big.Int, err error) //使用私钥对任意长度的hash值进行签名
func Verify(pub *PublicKey, hash []byte, r, s *big.Int) bool //使用公钥验证hash值和两个大整数r、s构成的签名
//代码在crypto/ecdsa/ecdsa.go
```

## 4、本文使用到的网络内容

* [初学者如何理解射影平面](https://wenku.baidu.com/view/3d245b608e9951e79b892768.html)
* [ECC椭圆曲线详解(有具体实例)](http://www.cnblogs.com/Kalafinaian/p/7392505.html)
* [说说椭圆曲线](http://blog.sina.com.cn/s/blog_564e1db00102vq25.html)
* [椭圆曲线密码学简介](http://www.8btc.com/introduction)
* [【信息安全】ECC加密算法入门介绍](https://yq.aliyun.com/articles/23897)
* [椭圆曲线算法：入门（1）](http://www.jianshu.com/p/2e6031ac3d50)

