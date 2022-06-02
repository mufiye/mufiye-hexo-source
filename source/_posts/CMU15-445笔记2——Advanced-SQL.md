---
title: CMU15-445笔记2——Advanced SQL
date: 2022-06-02 15:39:24
tags: [CMU 15-445, 存储, SQL]
categories: 数据库
---
# Relational Languages
* Data Manipulation Language：数据操作语言，例如insert、update、delete和select这些命令来操作存在于数据库中的数据。
* Data Definition Language：通过定义scheme来创建表存储数据。
* Data Control Language：关于安全性授权的语言，它用来控制哪些人可以读取哪些数据。

SQL是基于bags的，bags允许重复，且其中的元素没有次序和固定的位置。

SQL是一种标准，但是各种数据库会在标准上加入新的特性，如果一个数据库说它符合SQL标准，那么认为符合的是SQL-92标准。

## 1. Aggregations + Group By

### 1.1 Aggregations

聚合函数是处理一组元组并返回单个值的函数：

* AVG(col)：返回平均值
* MIN(col)：返回最小值
* MAX(col)：返回最大值
* SUM(col)：返回总和数值
* COUNT(col)：返回数量

### 1.2 Group By

Group By语句使在做数据库操作时可以根据特定属性进行分组。下面的代码块表示根据课程号(cid)和学生姓名(name)分别算出该课程的所有学生平均成绩和该学生的所有课程平均成绩。

```sql
SELECT AVG(S.gpa), e.cid, s.name 
FROM enrolled AS e, student AS s 
WHERE e.sid = s.sid 
GROUP BY e.cid, s.name;
```

HAVING语句可以用来过滤聚合操作的结果（GROUP BY特有的WHERE语句）。

```sql
SELECT AVG(S.gpa) AS avg_gpa, e.cid 
FROM enrolled AS e, student AS s 
WHERE e.sid = s.sid 
GROUP BY e.cid
HAVING avg_gpa > 3.5;
```

## 2. Stirng / Date、Time Operations

### 2.1 String Operation

|          | String Case | String Quotes |
| -------- | :---------- | ------------- |
| **SQL-92** | **Sensitive** | **Single Only** |
| Postgres | Sensitive   | Single Only   |
| MySQL    | Insensitive | Single/Double   |
| SQLite   | Sensitive   | Single/Double   |
| DB2      | Sensitive   | Single Only |
| Oracle   | Sensitive   | Single Only    |

* Like被用来匹配字符串
  * %被用来匹配子字符串（包括空字符串）
  * _被用来匹配任意一个字符
* SUBSTRING(name,0,5)：根据下标取子字符串
* LOWER(name)：使其中的字符小写
* UPPER(name)：使其中的字符大写
* 拼接字符操作（有三种，标准是||）
  * ||
  * +
  * CONCAT(s1,s2)

### 2.2 Date、Time Operation

* 获取当前时间

  * NOW()：函数（postgresql，mysql）
  * CURRENT_TIMESTAMP()：函数（mysql）
  * CURRENT_TIMESTAMP：关键字（postgresql，mysql，sqlite）

* 获取日期中的天数

  ```sql
  # postgresql
  SELECT EXTRACT(DAY FROM DATE('2018-08-29'));
  ```

* 获取距离某一天过了多久

  ```sql
  # mysql
  SELECT DATEDIFF(DATE('2022-06-02'),DATE('2000-05-14')) AS days;
  # sqlite
  SELECT julianday(CURRENT_TIMESTAMP) - julianday('2000-05-14');
  ```

## 3. Output Redirection + Control 

### 3.1 Output Redirection

可以将查询结果的内容存储到另一张表中

```sql
# sql-92
SELECT DISTINCT cid INTO CourseIds 
FROM enrolled;
# Mysql
CREATE TABLE CourseIds(
SELECT DISTINCT cid FROM enrolled);
# CourseIds之前已经创建好了，必须保证属性一致
INSERT INTO CourseIds(
SELECT DISTINCT cid FROM enrolled);
```

### 3.2 Output Control

* 排序

  ```sql
  ORDER BY <column> [ASC | DESC]
  ```

* 限制输出的数量

  ```sql
  # offset表示的是开始输出的tuple的偏移
  # 比如一共有10条记录，count为4，offset为1，则从第2条记录开始输出，一直输出到第5条记录
  LIMIT <count> [offset]
  ```

## 4. Nested Queries

嵌套查询，将一个查询的输出作为另一个查询的输入条件。

* ALL：子查询的所有行必须都满足条件。
* ANY：只要子查询的一行满足条件就可以。
* IN：与ANY()语义相同。
* EXISTS：至少返回一行数据。

这里有很多examples，我没有做记录，感兴趣可以看[这个视频的后半段](https://www.bilibili.com/video/BV1f7411z7dw?p=6)学习。

## 5. Window Functions

==之后再补==

## 6. Common Table Expressions

==之后再补==
