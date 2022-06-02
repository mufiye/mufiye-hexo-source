---
title: CMU15-445笔记1——Introduction
date: 2022-06-02 13:15:15
tags: [CMU 15-445，存储]
categories: 数据库
---

# 1. 为什么需要数据库

如果使用csv file而不使用数据库

## 1.1 数据一致性

1. 多个csv文件中相关联的多条数据之间不冲突
2. 对于csv文件中字段进行非法的字符修改
3. 我们如何能够存储多个内容到一个字段（比如书的作者可能有多个，我们要存储所有这些作者名字到书的作者这一栏，但这会难以解析）

## 1.2 实现

1. 如何在cvs file中找到一条特定的记录
2. 使用csv file对于不同应用需要不同的程序来读取数据
3. 多线程读写问题

## 1.3 持久性

1. 存储csv file的机器在读写文件时宕机
2. 复制该csv file在多台机器上保证高可用性
# 2. DBMS

分析了使用csv file来存储数据，可以看出我们迫切需要数据库和DBMS。DBMS(Database Management System)是允许应用程序在数据库中存储和分析信息的软件，而广义的DBMS被设计来定义、创建、查询、更新和管理数据库，这也是该课程的项目所要做的东西。

## 2.1 Data model
data model是指如何组织数据的高层次概念。

* SQL - Relational
* NoSQL
  * Key/Value
  * Graph
  * Document
  * Column-family
* Other
  * Array/Matrix（在机器学习中使用）
  * Hierarchical（过时的）
  * Network（过时的）

## 2.2 Scheme

Scheme是关于存储数据时所使用的定义，其是指使用给定数据模型对特定数据集合的描述。

## 2.3 Data Manipulation Languages

* Procedural：这种请求指定了高层次的策略告诉DBMS应当如何去找到想要的结果。（关系代数）（关系型数据库需要关注的）

* Non-Procedural：这个请求只告诉DBMS想要什么数据而不是如何去找到它们。（关系演算）

## 2.4 查询以及查询优化

数据模型应当独立于查询语句。

# 3. 关系数据模型

**论文**：《A Relational Model of Data for Large Shared Data Banks》

## 3.1 关系数据模型三要素：

1. 将关系转化为简单的数据结构然后存入数据库
2. 使用高级语言来获取数据、应用数据
3. 大型数据库的物理存储策略取决于数据库管理系统的实现

## 3.2 关系数据模型的三个组成部分：

* Structure：我们在我们的关系中该定义什么，它们的内容是什么，类型是什么。
* Integrity：关于数据的约束。
* Manipulation：操作数据库中内容的方式。

## 3.3 关系的组成：

* tuple：一个tuple表示关系数据模型中的一条记录，其对应了一组属性值。
* primary key：主键某一个能够唯一标识一条记录的属性。
* foreign key：外键用于指定一张表中的属性必须存在于另一张表中，它用来维护不同表之间的数据一致性。

## 3.4 最初的关系代数运算符：

* Select：选择满足条件的子集
* Projection：生成一个新的输出关系，它里面只包含一个来自我们给定输入关系中的指定属性
* Union：将两个关系组合生成一个新的关系，这其中包含了这两个关系中的全部tuple
* Intersection：生成一个包含了在两个关系中都出现过的tuple的输出关系
* Difference：只取在一个关系中出现的元素，而不取另一个关系中的元素
* Product：笛卡尔积，生成两个输入关系中所有tuple的可能组合
* Join：自然连接，对于一个关系中的每一个tuple，观察它与另一个关系是否具有相同名称，相同类型的所有属性匹配，如果有，那么这就是这两个关系共同拥有的元素，那么就可以进行连接操作（将对应的tuple连接起来，去除掉一份相同的部分）。要注意的是，该操作符和difference很像但是有区别，difference取的两个tuple必须关系属性以及关系属性的内容完全相同但是Join可以有不相同的关系属性（相同的关系属性其内容必须相同）。

## 3.5 额外的关系代数运算符：

* Rename
* Assignment
* Duplicate Elimination
* Aggregation
* Sorting
* Division



