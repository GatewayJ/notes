---
author : "jihongwei"
tags : ["分布式"]
date : 2021-01-24T11:24:00Z
title : "分布式概念概览"
---


#### 一.分布式系统原理

1. 数据分布方式

   * hash方式
   * 按照数据范围分布
   * 按照数据量分布
   * 一致性hash
   
2. 副本与数据分布

3. 本地化方式 移动数据不如移动计算

4. 分布式协议

   基本副本协议
   
   * 中心化副本控制协议 primary-secondary协议

   数据更新流程

   数据读取方式：实现强制性的集中思路1.primary实现读写2.primary控制secodary副本的可用性3.quorom方案

   primary副本的确定和切换：quorom

   数据同步：1.网络分化 2.脏数据 3.新副本

   * 去中心化的副本协议

5. 一致性的种类 强一致性 弱一致性 最终一致性

#### 二.分布式机制

1. 租约机制 最重要的应用：判定节点状态

2. quorom

3. 日志技术

relog和checkpoint    no undo和no redolog

4. 两阶段提交

5. mvcc

6. paxos 最基本的paxos,会产生活锁

7. raft 主要应用：集群变更，集群分区，日志压缩，与客户端交互，重新选举，日志复制规范，联合共识配置变更

8. zab