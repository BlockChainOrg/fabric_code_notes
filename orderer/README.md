# Fabric 1.0源代码笔记 之 Orderer

## 1、Orderer概述

Orderer，为排序节点，对所有发往网络中的交易进行排序，将排序后的交易安排配置中的约定整理为块，之后提交给Committer进行处理。

Orderer代码分布在orderer目录，目录结构如下：

* orderer目录
	* main.go，main入口。
	* util.go，orderer工具函数。
	* metadata目录，metadata.go实现获取版本信息。
	* localconfig目录，config.go，本地配置相关实现。
	* ledger目录，账本区块存储。
		* file目录，账本区块文件存储。
		* json目录，账本区块json文件存储。
		* ram目录，账本区块内存存储。
	
如下为分节说明Orderer代码：
	
* [Fabric 1.0源代码笔记 之 Orderer（1）orderer start命令实现](orderer_start.md)
* [Fabric 1.0源代码笔记 之 Orderer（2）localconfig（Orderer配置文件定义）](localconfig.md)
* [Fabric 1.0源代码笔记 之 Orderer（3）ledger（Orderer Ledger）](orderer_ledger.md)