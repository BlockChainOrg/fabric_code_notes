# Fabric 1.0源代码笔记 之 Tx（1）RWSet（读写集）

## 1、RWSet概述

在背书节点模拟Transaction期间，为交易准备了一个读写集合。
Read Set包含模拟Transaction读取的Key和版本的列表，Write Set包含Key、写入的新值、以及删除标记（是否删除Key）。

![](rwset.png)

## 2、
