# Fabric 1.0源码旅程 之 flogging（Fabric日志系统）

## 1、flogging概要

flogging，即fabric logging，对第三方日志包go-logging做了封装，供全局使用。go-logging地址：https://github.com/op/go-logging。
flogging代码集中在common/flogging目录下，包括logging.go和grpclogger.go。

* logging.go，定义了默认的日志格式、日志级别和日志输出，以及modules和peerStartModules做模块和日志级别的映射。并定义了若干对go-logging封装的函数。
* grpclogger.go，基于封装go-logging定义了结构体grpclogger及其方法，并用于设置grpclog。grpclog默认使用go标准库日志包，此举可使得grpclog也使用go-logging和flogging功能。

## 2、flogging的常量和全局变量

涉及常量如下：

```go
const (
	pkgLogID      = "flogging" //仅在flogging包内代码使用的go-logging名称
	//defaultFormat为默认的日志格式
	defaultFormat = "`%`{color}`%`{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}"
	defaultLevel  = logging.INFO //默认的日志级别
)
//代码在common/flogging/logging.go
```

涉及全局变量如下：

```go
var (
	logger *logging.Logger //仅在flogging包内代码使用的logging.Logger对象
	defaultOutput *os.File //默认的日志输出
	modules          map[string]string //保存所有模块及其各自的日志级别的映射
	peerStartModules map[string]string //存储内容与modules相同
	lock sync.RWMutex //RWMutex读写锁
	once sync.Once    //对于从全局的角度只需要运行一次的代码，比如全局初化操始作，go语言提供了一个Once类型来保证全局的唯一性操作
)
//代码在common/flogging/logging.go
```

## 3、flogging对go-logging的封装

### 3.1、flogging包初始化

flogging包初始化，即init()函数，代码如下：

```go
func init() {
	logger = logging.MustGetLogger(pkgLogID) //创建仅在flogging包内代码使用的logging.Logger对象
	Reset()                                  //全局变量初始化为默认值
	initgrpclogger() //初始化gRPC Logger，即创建logging.Logger对象，并用这个对象设置grpclog
}
//代码在common/flogging/logging.go
```

其中func Reset()代码如下。
其作用为：初始化modules和lock，创建一个日志输出对象并设置为默认的日志格式和默认的日志级别。
设置各模块的日志级别，并更新modules。

```go
func Reset() {
	modules = make(map[string]string) //初始化modules
	lock = sync.RWMutex{} //初始化lock
	defaultOutput = os.Stderr //默认的日志输出置为os.Stderr
	//SetFormat()设置并获取go-logging日志格式，InitBackend()创建一个日志输出对象并设置输出格式和日志级别
	InitBackend(SetFormat(defaultFormat), defaultOutput) 
	InitFromSpec("") //设置各模块日志级别，并更新modules
}
//代码在common/flogging/logging.go
```

func InitBackend(formatter logging.Formatter, output io.Writer)代码如下。
创建一个日志输出对象并设置输出格式和日志级别。

```go
func InitBackend(formatter logging.Formatter, output io.Writer) {
	backend := logging.NewLogBackend(output, "", 0) //创建一个日志输出对象
	backendFormatter := logging.NewBackendFormatter(backend, formatter) //设置日志输出对象的输出格式
	logging.SetBackend(backendFormatter).SetLevel(defaultLevel, "") //设置日志输出对象的日志级别
}
//代码在common/flogging/logging.go

func InitFromSpec(spec string) string代码如下。
其中spec格式为：[<module>[,<module>...]=]<level>[:[<module>[,<module>...]=]<level>...]。
此处传入spec为""，将""模块日志级别设置为defaultLevel，并会将modules初始化为defaultLevel。

```go
levelAll := defaultLevel //defaultLevel为logging.INFO
var err error

if spec != "" { //如果spec不为空，则按既定格式读取
	fields := strings.Split(spec, ":") //按:分割
	for _, field := range fields {
		split := strings.Split(field, "=") //按=分割
		switch len(split) {
		case 1: //只有level
			if levelAll, err = logging.LogLevel(field); err != nil { //levelAll赋值为logging.LogLevel枚举中定义的Level级别
				levelAll = defaultLevel // 如果没有定义，则使用默认日志级别
			}
		case 2: //针对module,module...=level，split[0]为模块集，split[1]为要设置的日志级别
			levelSingle, err := logging.LogLevel(split[1]) //levelSingle赋值为logging.LogLevel枚举中定义的Level级别
			modules := strings.Split(split[0], ",") //按,分割获取模块名
			for _, module := range modules {
				logging.SetLevel(levelSingle, module) //本条规则中所有模块日志级别均设置为levelSingle
			}
		default:
			//...
		}
	}
}

logging.SetLevel(levelAll, "") // 将""模块日志级别设置为levelAll，如果logging.GetLevel(module)没找到时将使用""模块日志级别
for k := range modules {
	MustGetLogger(k) //获取模块日志级别，并更新modules
}
MustGetLogger(pkgLogID) //pkgLogID及其日志级别，更新至modules
return levelAll.String() //返回levelAll
//代码在common/flogging/logging.go
```

MustGetLogger会调取go-logging包中GetLevel()，附GetLevel()代码如下。
优先按module获取日志级别，如未找到则按""模块获取日志级别，如仍未找到则默认按DEBUG级别。

```go
func (l *moduleLeveled) GetLevel(module string) Level {
	level, exists := l.levels[module]
	if exists == false {
		level, exists = l.levels[""]
		if exists == false {
			level = DEBUG
		}
	}
	return level
}
//代码在github.com/op/go-logging/level.go
```

### 3.2、flogging包封装的方法

flogging包封装的方法，如下：








```go


//代码在common/flogging/logging.go
```





	* func SetFormat(formatSpec string) logging.Formatter {
	* func InitBackend(formatter logging.Formatter, output io.Writer) {
	* func DefaultLevel() string {
	* func GetModuleLevel(module string) string {
	* func SetModuleLevel(moduleRegExp string, level string) (string, error) {
	* func setModuleLevel(moduleRegExp string, level string, isRegExp bool, revert bool) (string, error) {
	* func MustGetLogger(module string) *logging.Logger {
	* func InitFromSpec(spec string) string {
	* func SetPeerStartupModulesMap() {
	* func GetPeerStartupLevel(module string) string {
	* func RevertToPeerStartupLevels() error {


## 5、本文使用到的网络内容

* [fabric源码解析3——日志系统](http://blog.csdn.net/idsuf698987/article/details/75223986)

