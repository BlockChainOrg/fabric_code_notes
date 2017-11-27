# Fabric 1.0源代码笔记 之 Orderer

## 1、Orderer概述

Orderer，为排序节点，对所有发往网络中的交易进行排序，将排序后的交易安排配置中的约定整理为块，之后提交给Committer进行处理。

Orderer代码分布在orderer目录，目录结构如下：

* orderer目录
	* main.go，main入口。
	
如下为分节说明Orderer代码：
	
* [Fabric 1.0源代码笔记 之 Orderer（1）orderer start命令实现](orderer_start.md)