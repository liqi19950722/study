# 高性能Mysql笔记

## 基础

### Mysql逻辑架构

### 并发控制

shared lock 共享锁(读锁)

- 多个客户同一时刻读取同一资源

exclusive lock 排他锁(写锁)

- 只有一个客户执行

原则：锁定的数据量越少，并发性越高。

表锁

行锁

### 事务

ACID

隔离级别

- READ UNCOMMITTED
- READ COMMITTED
- REPEATABLE READ
- SERIAZABLE

死锁

```mysql
START TRANSACTION;
UPDATE Stockprice SET close = 45.50 WHERE stock_id=4 and date='2002-06-01';
UPDATE Stockprice SET close = 19.80 WHERE stock_id=3 and date='2002-06-01';
COMMIT;

START TRANSACTION;
UPDATE Stockprice SET close = 19.80 WHERE stock_id=3 and date='2002-06-01';
UPDATE Stockprice SET close = 45.50 WHERE stock_id=4 and date='2002-06-01';
COMMIT;
```

#### Mysql事务

自动提交

```mysql
SHOW VARIABLES LIKE 'AUTOCOMMIT';
SET AUTOCOMMIT = 1;//1 启用 0 禁用

```

### MVCC

通过保存数据在某一时间点的快照。

InnoDB的MVCC

每行记录后面保存两个隐藏列实现。（系统版本号）

- 一个保存行的创建时间；

- 一个保存行的过期时间（删除时间）。 

在REPITABLE READ级别下

SELECT

1. 查找版本早于当前事务的行
2. 行的删除版本未定义（NULL）,大于当前事务版本号

INSERT

- 新插入的每一行保存当前系统版本号

DELETE

- 删除的每一行保存当前系统版本号

UPDATE

- 插入一行新纪录，保存当前系统版本号作为行号。
- 保存当前系统版本好的作为原来行的删除标识。

只在 `REPITABLE READ` 和 `READ COMMITTED` 隔离级别下工作

### 存储引擎

```mysql
SHOW TABLE STATUS 
```

InnoDB

- 支持事务
- 数据存储在表空间

MyISAM

- 全文索引
- 压缩
- 空间函数
- 不支持事务和行级锁
- 表存储在数据文件和索引文件中

Archive

- 只支持INSERT和SELECT
- I/O少
- 不支持事务

Memory

- Hash索引
- 不支持BLOB和TEXT

## 索引

索引可以包含一个或多个值。索引包含多个列，顺序十分重要（最左前缀列）

优点

- 减少扫描的数据量
- 帮助服务器避免排序和临时表
- 将随机IO变为顺序IO

### B-Tree索引

结构：

- key
- 指向叶子的指针
- 指向下一个叶子节点的指针

查询

- 全值匹配
- 匹配最左列前缀
- 匹配列前缀
- 匹配范围值
- 精确匹配某一列并范围匹配另外一列
- 只访问了索引查询

限制：

- 不按最左列匹配，无法使用索引
- 不能跳过索引中的列
- 查询中有某个列的范围查询，右边所有列无法使用索引优化查询

### Hash索引

精确匹配索引才有效。

对所有索引列计算一个hash码，hash码存储在索引中，同时在hash表中保存指向每个数据行的指针。

### R-tree

MyISAM 表支持空间索引。

从所有纬度来索引数据。使用任意纬度来组合查询

### 全文索引

类似搜索引擎

### 聚簇索引

### 覆盖索引