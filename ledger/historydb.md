# Fabric 1.0源代码笔记 之 Ledger（4）historydb（历史数据库）

## 1、historydb概述

historydb代码分布在core/ledger/kvledger/history/historydb目录下，目录结构如下：

* historydb.go，定义核心接口HistoryDBProvider和HistoryDB。
* histmgr_helper.go，historydb工具函数。
	* historyleveldb目录，historydb基于leveldb的实现。
		* historyleveldb.go，HistoryDBProvider和HistoryDBz接口实现，即HistoryDBProvider和historyDB结构体及方法。
		* historyleveldb_query_executer.go，定义LevelHistoryDBQueryExecutor和historyScanner结构体及方法。

## 2、核心接口定义

