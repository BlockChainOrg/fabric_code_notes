# Fabric 1.0源代码笔记 之 consenter（共识插件） #filter（过滤器）

## 1、filter概述

filter代码分布在common/filter、common/sizefilter、common/sigfilter、orderer/multichain目录下。

common/filter/filter.go，Rule接口定义及emptyRejectRule和acceptRule实现，Committer接口定义及noopCommitter实现，RuleSet结构体及方法。
orderer/multichain目录，filter工具函数。

## 2、Rule接口定义及实现

### 2.1、Rule接口定义

```go
type Action int
const (
	Accept = iota
	Reject
	Forward
)

type Rule interface { //定义一个过滤器函数, 它接受、拒绝或转发 (到下一条规则) 一个信封
	Apply(message *ab.Envelope) (Action, Committer)
}
//代码在orderer/common/filter/filter.go
```

### 2.2、emptyRejectRule（空拒绝规则）和acceptRule（接受规则）结构体

emptyRejectRule结构体（空拒绝规则）：

```go
type emptyRejectRule struct{}
var EmptyRejectRule = Rule(emptyRejectRule{})

func (a emptyRejectRule) Apply(message *ab.Envelope) (Action, Committer) {
	if message.Payload == nil {
		return Reject, nil
	}
	return Forward, nil
}
//代码在orderer/common/filter/filter.go
```

acceptRule结构体:

```go
type acceptRule struct{}
var AcceptRule = Rule(acceptRule{})

func (a acceptRule) Apply(message *ab.Envelope) (Action, Committer) {
	return Accept, NoopCommitter
}
//代码在orderer/common/filter/filter.go
```

## 3、Committer接口定义及实现

Committer接口定义:

```go
type Committer interface {
	Commit() //提交
	Isolated() bool //判断交易是孤立的块，或与其他交易混合的块
}
//代码在orderer/common/filter/filter.go
```

noopCommitter结构体：

```go
type noopCommitter struct{}
var NoopCommitter = Committer(noopCommitter{})

func (nc noopCommitter) Commit()        {}
func (nc noopCommitter) Isolated() bool { return false }
//代码在orderer/common/filter/filter.go
```

### 4、RuleSet结构体及方法

```go
type RuleSet struct {
	rules []Rule
}

func NewRuleSet(rules []Rule) *RuleSet //构造RuleSet
func (rs *RuleSet) Apply(message *ab.Envelope) (Committer, error) {
	for _, rule := range rs.rules {
		action, committer := rule.Apply(message)
		switch action {
		case Accept: //接受
			return committer, nil
		case Reject: //拒绝
			return nil, fmt.Errorf("Rejected by rule: %T", rule)
		default:
		}
	}
	return nil, fmt.Errorf("No matching filter found")
}
//代码在orderer/common/filter/filter.go
```

### 5、filter工具函数