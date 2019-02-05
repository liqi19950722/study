# Mysql

## 索引（关键字 数据区 子节点引用）

为了加速表中数据行的检索而创建的一种分散存储的数据结构。

为什么用索引

1. 减少存储引擎需要扫描的数据量

2. 把随机IO ->顺序IO

3. 在进行分组、排序操作时，避免使用临时表

B+树

https://www.cs.usfca.edu/~galles/visualization/Algorithms.html

二叉树、平衡二叉树

问题

- 树的高度太高。

- 每个节点保存的数据量太小。

多路平衡二叉树

B+数（左闭合）

B树 和B+树区别

1. B+树节点关键字采用闭合区间
2. B+树非叶节点不保存数据相关信息，只保存关键字和引用
3. B+数关键字对应的数据保存在叶子节点中
4. B+数叶子节点是顺序排列，相邻节点具有顺序引用关系

B+树优势

1. B+树扫表能力
2. B+树IO读写更强
3. 排序能力更强
4. 查询效率更稳定（永远在叶子节点取数据）

B+树在存储引擎中的提现形式

Myisam

数据和索引分开存储

索引（.MYI）

数据（.MYD）

Innodb

以主键为索引来组织数据的存储（.IBD）

其他

列的离散性

- 离散性越高越好

最左匹配原则

- 关键字对比时 依次从左往右进行，不可跳过。

联合索引

- 空间少
- 最左匹配
- 离散度高

覆盖索引

查询列可通过索引节点中的关键字直接返回

减少IO

## Mysql体系结构

Client Connector -> SQL Interface/Parser/Optimizer/Caches ->

Pluggable Storage Engines->

FileSystem

## 存储引擎

### CSV存储引擎

没有索引，NOT NULL 不能设置列自增

存储数据用","隔开 可以直接编辑。

### Archive存储引擎

压缩协议进行数据存储

.arz

特点

- 支持insert和select操作
- 只允许自增ID列建立索引
- 行级锁
- 不支持事务
- 数据占用磁盘少

场景

- 日志系统
- 大量设备数据采集

### Memory存储引擎

数据存储在内存中，IO效率比其他引擎高

特点

- 支持Hash索引，B tree索引 不支持大数据存储类型字段

- 表级锁

场景

- 等值查询热度较高的数据
- 查询结果内存中的计算

### Myisam引擎

特点

- 表级锁
- 不支持事务

### Innodb引擎

- 支持事务
- 行级锁



https://dev.mysql.com/doc/refman/5.7/en/storage-engines.html



## 查询流程

1. 客户端/服务端通信

   查看连接线程状态

   https://dev.mysql.com/doc/refman/5.7/en/general-thread-states.html

   ```
   show full processlist/ show processlist
   ```

2. 查询缓存

   默认关闭

3. 查询优化处理

   解析SQL

   预处理

   查询优化器

4. 查询执行引擎

5. 返回客户端

## 执行计划

**ID**

1. ID相同，执行顺序由上至下
2. ID不同，如果是子查询ID序号递增，ID值越大优先级越高，越被先执行
3. ID相同又不同，ID相同同组 从上往下，所有组ID越大越先执行（非绝对）

**select_type**

1. SIMPLE 不包含子查询

2. PRIMARY 包含子查询部分，最外层为PRIMARY

3. SUBQUERY/MATERIALIZED 

   SUBQUERY 在SELECT 或 WHERE 列中包含子查询,

   MATERUALIZED 表示WHERE后面IN条件子查询

4. UNION  若第二个SELECT 出现在UNION之后，则被标记为UNION；

5. UNION RESULT 从UNION表获取结果的SELECT 

**table**

1. 直接显示表名
2. <unionM,N> 由ID为M,N 查询union产生的结果
3. <subqueryN> 由ID为N查询生产的结果

**type**

system>const>eq_ref>ref>range>index>all

`system`：表只有一行记录（等于系统表），const类型的特例，基本不会出现，可以忽略不计

`const`：表示通过索引一次就找到了，const用于比较primary key 或者 unique索引
`eq_ref`：唯一索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键 或 唯一索引扫描
`ref`：非唯一性索引扫描，返回匹配某个单独值的所有行，本质是也是一种索引访问
`range`：只检索给定范围的行，使用一个索引来选择行
`index`：索引全表扫描，把索引从头到尾扫一遍
`ALL`：遍历全表以找到匹配的行

**possible_keys**

查询过程中可能用到的索引

**key**

实际用到的索引

**rows**

影响行数

**filterred**

它指返回结果的行占需要读到的行 (rows 列的值) ) 的百分比
表示返回结果的行数占需读取行数的百分比， filtered的值越大越好

**Extra**

Using filesort

Using temporary

Using index

Using where

select tables optimized away

https://tech.meituan.com/2014/06/30/mysql-index.html

## 定位慢sql

业务驱动

测试驱动

慢日志查询

```
show variable like 'slow_query_log'
set global slow_query_log = on
set golbal slow_query_log_file='路径'
set global log_queries_not_using_indexes = on
set global long_query_time = 0.001（秒）
```

- 慢查询分析

  - Time:日志记录时间

  - User@host: 执行的用户及主机

  - Query_time: 查询耗费时间 Lock_time: 锁表时间 Row_sent发送给请求方的记录 Rows_examined: 语句扫描的记录条数

  - Set timestamp 语句执行的时间点

  - 分析工具

    ```
    mysqldumpslow --help
    ```

## 事务ACID

原子性

- 一起成功，一起失败

一致性

- 状态一致

隔离性

- 一个事务所操作的数据在提交之前，对其他事务的可见性设定（一般设定为不可见）

持久性

- 永久保存

### 事务并发

脏读

不可重复读

幻读

sql92标准

http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt

Read Uncommitted

Read Committed

Repeatable Read

Serializable

### 锁

表锁VS行锁

锁定粒度：表锁>行锁

加锁效率：表锁>行锁

冲突概率：表锁>行锁

并发性能：表锁<行锁

InnoDB存储引擎支持（表锁行锁）

### InnoDB 锁类型

https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html

- 共享锁（行锁）(读锁) `IN SHARE MODE`

- 排他锁（行锁）（写锁）`FOR UPDATE`

  行锁是通过对索引项加锁实现

  只有通过索引条件进行数据检索，InnoDB才使用行级锁，否则，InnoDB将使用表锁（锁住索引的所有记录）

- 意向共享锁（表锁）

- 意向排他锁（表锁）

  当事务想去进行锁表时，可以先判断意向锁是否存在，存在时则可快速返回该表不能
  启用表锁

- 自增锁

- 记录锁（Record Lock）

  锁住具体的索引项

- 间隙锁（Gap Lock）

  锁住数据不存在的区间（左开右开）

- 临键锁（Next-key Lock）

  锁住记录+ 区间（左开右闭）

  防止幻读

## MVCC（多版本并发控制）

`DB_TRX_ID` 提交版本号

`DB_ROLL_PT` 删除版本号

插入 插入数据 `DB_TRX_ID`为当前事务ID

删除 把数据的`DB_ROLL_PT` 修改为当前事务ID

修改 复制一条数据，`DB_TRX_ID`为当前事务ID;旧数据的`DB_ROLL_PT` 修改为当前事务ID

查询 找出`DB_TRX_ID`小于当前事务ID， `DB_TRX_ID`为NULL 或者大于当前事务ID的数据