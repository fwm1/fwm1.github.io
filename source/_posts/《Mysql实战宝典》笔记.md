---
title: 《Mysql实战宝典》笔记
date: 2023-05-05 21:01:46
tags: [数据库,mysql]
categories:  数据库
---

### 数字类型

![image-20230505212034763](https://raw.githubusercontent.com/fwm1/PicturesRepo/master/blog-images/20230505212036.png)

- 在使用 unsigned时需要注意，MySQL默认要求unsigned数值相减之后依然为 unsigned，否则就会报错。设置参数**SET** sql_mode='NO_UNSIGNED_SUBTRACTION'可以避免这个报错；
- 自增整型类型做主键，务必使用类型 BIGINT，当达到自增整型类型的上限值时，再次自增插入，MySQL 数据库会报重复错误；
- **注意**：MySQL 8.0 版本前，自增整型会有回溯问题；
- **不要**使用浮点类型 Float、Double，MySQL 后续版本将不再支持上述两种类型；
- 账户金额字段，可以考虑用整型类型（使用‘分’作为单位存储而不是‘元’）而不是 DECIMAL 类型，这样性能更好，存储更紧凑。



### 字符类型

- CHAR 和 VARCHAR 虽然分别用于存储定长和变长字符，但对于变长字符集（如 GBK、UTF8mb4），其本质是一样的，都是变长，**设计时完全可以用 VARCHAR 替代 CHAR；
- 排序规则很重要，用于字符的比较和排序，但大部分场景不需要用区分大小写的排序规则；
- 修改表中已有列的字符集，使用命令 **ALTER **TABLE  table_1 **CONVERT** **TO** CHARSET utf8mb4；
- 用户性别，运行状态等有限值的列，MySQL 8.0.16 版本可以使用 CHECK 约束机制，之前的版本可使用 ENUM 枚举字符串类型，外加 SQL_MODE 的严格模式；
- 加密字段处理。简单的MD5算法可以进行暴力破解，推荐使用动态盐+动态加密算法进行隐私数据的存储。比如动态拼接：$salt$encry_type$encryped_content

### 索引

- 一般不要对区分度小的列单独创建索引，比如性别。但是在一些业务场景中要根据记录状态进行筛选，但 MySQL 并不知道这个状态列数据的分布情况，认为是平均分布的，故一般会使用全表扫描，此时可以利用Mysql8的直方图功能，让Sql优化器提前知道数据分布，会根据数据分布情况选择合适的索引。

### JOIN连接

Nested Loop Join 之间的表关联是使用索引进行匹配的，假设表 R 和 S 进行连接，其算法伪代码大致如下：

```sql
// R为驱动表，S为被驱动表
for each row r in R with matching condition:

    lookup index idx_s on S where index_key = r

    if (found)

      send to client
```

left join左边为驱动表，right join右表为驱动表，inner join查询优化器会选择where过滤后数据量小的作为驱动表。

##### 避免依赖子查询DEPENDENT SUBQUERY



### Mysql高可用

mysql bin-log复制相关参数：

```properties
gtid_mode = on

enforce_gtid_consistency = 1

binlog_gtid_simple_recovery = 1

relay_log_recovery = ON

master_info_repository = TABLE 

relay_log_info_repository = TABLE
```

复制类型：

- 异步复制
- 半同步复制：半同步复制要求 Master 事务提交过程中，至少有 N 个 Slave 接收到二进制日志
  - 有损半同步（版本< 5.7）
  - 无损半同步（版本>=5.7）
- 多源复制：多个master复制到一台slave
- 延迟复制：允许slave 延迟回放接收到的binlog，为了避免master上的误操作，同步到从slave，导致数据完全丢失

![image-20230513234227571](https://raw.githubusercontent.com/fwm1/PicturesRepo/master/blog-images/20230513234229.png)

![image-20230513234248259](https://raw.githubusercontent.com/fwm1/PicturesRepo/master/blog-images/20230513234250.png)

可以看到有损半同步复制时在commit事务之后发送日志，如果在事务commit之后，slave节点收到日志之前master挂了，会导致已提交的数据在slave节点上丢失；

无损同步复制则是在commit事务之前发送日志，如果此时master挂了，因为还没有commit，所以不存在丢数据。

##### binlog的优缺点

- 优点：逻辑日志简单易懂，方便数据之间的同步
- 缺点：逻辑日志会记录更改前后的数据内容，会产生大量日志文件，会影响事务提交耗时

##### 主从复制延迟优化

- 升级到5.7版本，salve支持多线程并行回放binlog

- 设置并行复制模式为WRITESET。基于每个事务并行，如果事务间更新的记录不冲突，就可以并行。即使master是单线程执行，回放是slave也能够并行

  ```properties
  binlog_transaction_dependency_tracking = WRITESET
  
  transaction_write_set_extraction = XXHASH64
  
  slave-parallel-type = LOGICAL_CLOCK
  
  slave-parallel-workers = 16
  ```

主从复制延迟监控不能依赖 Seconds_Behind_Master 的值，最好的方法是额外配置一张心跳表。master定时写入心跳表，更新最新时间，同步到slave节点之后，复制延迟=slave节点时间 - 心跳表最新时间。

读写分离时发生主从复制延迟怎么办？ 可以控制读写分离的权重，当发生读写分离时，将读请求的权重全部分配到主节点，相当于关闭读写分离。

##### 容灾方案

同城三中心

![image-20230514151550650](https://raw.githubusercontent.com/fwm1/PicturesRepo/master/blog-images/20230514151552.png)

