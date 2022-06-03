---
title: MIT6.824——Introduction
date: 2022-06-03 12:47:00
tags: [MIT6.824, 分布式]
categories: 分布式系统
---



# Introduction

**分布式系统的特点：**

* parallelism

* fault tolerance

* physical

* security / isolated

**分布式系统面临的挑战：**

* concurrency
* partial failure
* performance

**分布式系统在基础架构的使用：**

* Storage
* Communication
* Computation

**分布式系统的实现工具:**

* RPC
* threads
* concurrency lock

**分布式系统需要考虑的：**

* Scalability：两倍的计算机数量能让我获得两倍的性能
* Availability：发生故障时的可用能力
* Recoverability：从故障中恢复的能力，依靠非易失性存储和副本
* consistency：一致性，考虑它的原因是，在分布式系统中，经常有不只一份数据
  * 强一致性：很耗时
  * 弱一致性

# MapReduce



