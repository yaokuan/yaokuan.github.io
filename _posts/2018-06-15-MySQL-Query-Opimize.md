---
layout:     post
title:      深入理解MySQL：查询优化篇
date:       2018-06-15 20:09:00
categories: MySQL
tags:       MySQL 索引 数据库 优化
---

* 文章目录
{:toc}

### 查询优化
mysql查询要经过客户端传输、服务器解析、生成执行计划、执行、返回给客户端等步骤



#### 优化数据访问
是否向数据库请求了不必要的数据
1. 查询不需要的记录
2. 多表关联返回全部列
3. 总是取出全部列(select * from)
4. 重复查询相同数据

#### mysql是否在扫描额外的记录
1. 响应时间
2. 扫描的行数
3. 访问类型
4. 全表扫描 > 索引扫描 > 范围扫描 > 唯一索引扫描 > 常数引用

#### 重构查询的方式
1. 一个复杂查询 or 多个简单查询
2. 切分查询
3. 分解关了查询

---

### 查询执行的基础

#### 查询执行路径
![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/mysql/query-execution-plan.png)

#### 查询状态
1. Sleep
2. Query
3. Locked
4. Analyzing and statistics
5. Coping to tmp table [on disk]
6. Sorting result
7. Sending data

#### 关联查询
假设有查询语句：
```sql
SELECT tbl1.col1, tbl2.col2 FROM tbl1 INNER JOIN tbl2 USING(col3) WHERE tbl1.col1 IN(5,6);
```
则查询关联的顺序如图中所示：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/mysql/relation-query.png)

**tips**:
1. 如果对某个查询执行explain extended，再执行show warnings，就可以看到mysql重构后的完整查询。
2. msyql做关联查询时会做关联表优化，使用straight_join关键字可以禁止关联顺序优化，这在某些特殊场合是有效的。

#### 排序优化
应该尽量避免排序，如果一定需要排序，优先使用索引排序，若不能使用索引；尽量使用order by字段的表作为驱动表。这样可以在处理第一张表的时候就进行文件排序，避免生成临时表。

#### 索引合并优化
- 松散索引扫描
- 最大值和最小值优化

---

### 优化limit分页
优化分页查询最简单的办法就是尽可能地使用索引覆盖扫描，然后根据需要再做一次关联返回需要的列；而不是直接查询所有的列。下面我们看一个例子：

#### 表结构
假设有一张表film，结构如下：
![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/mysql/demo-structure.png)

#### 执行计划
##### 优化前
![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/mysql/table-structure.png)
例如我们常用的分页查询，执行时间是0.03s，使用explan命令可以看到执行计划中rows为1000，而且Extra列有**using filesort**，如果数据量更大，sql可能会更慢

##### 优化后
![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/mysql/join-optimize.png)
那么如果我们换一种关联查询的方式，title字段order by使用了覆盖索引扫描，即不用再做文件排序，这种方式对于大表的排序是非常有效的。整个查询时间小于0.01s，通过explain可以看到执行计划中驱动表的Extra列为using index，则即意味着我们在这个查询使用到了覆盖索引，这里有几点需要说明：
1. film表的主键是film_id，而title列是二级索引，所以在第一次查询的film_id和description使得无法使用title字段做覆盖索引扫描
2. 如果我们先扫描一次film表通过title索引覆盖扫描film_id，然后通过临时表关联主表通过primary key查出需要的列

