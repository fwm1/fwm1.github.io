---
title: 分布式ID生成方法总结
date: 2021-11-14 10:02:49
tags: 分布式ID
categories: 分布式
---

### 分布式ID特性

- 全局唯一性
- 高可用
- 高性能
- 最好是有序递增

### 分布式ID实现方式

- UUID
- 数据库自增
- 号段模式
- 雪花算法
- 借助Redis

### JDK的UUID

- 优点：简单，性能，全球唯一性
- 缺点：字符串存储，可读性差，无序，数据库占用空间大且查询效率低

### 数据库自增ID

使用数据库的主键不唯一特性保证唯一性。（使用MyISAM引擎是因为不会有update场景）

```sql
CREATE DATABASE `SEQ_ID`;
CREATE TABLE SEQID.SEQUENCE_ID (
    id bigint(20) unsigned NOT NULL auto_increment, 
    PRIMARY KEY (id),
) ENGINE=MyISAM;
```

单台数据库在高并发下很容易达到性能瓶颈且不能保证高可用，因此需要数据库**双主模式集群**，同时两个数据库设置主键的**起始值**和**步长**

mysql1

```sql
set @@auto_increment_offset = 1;     -- 起始值
set @@auto_increment_increment = 2;  -- 步长
```

mysql2

```sql
set @@auto_increment_offset = 2;     -- 起始值
set @@auto_increment_increment = 2;  -- 步长
```

优点：唯一性保证，自增

缺点：mysql单点风险，集群模式扩容方案复杂，高并发能力有限

### 基于数据库的号段模式（常用）

1. 数据库自增ID模式下，每次获取ID都要访问一次数据库，造成数据库压力大
2. 改为每次获取一个segment(step决定大小)号段的值，用完之后再去数据库获取新的号段，可以大大的减轻数据库的压力
3.  各个业务不同的发号需求用app_tag字段来区分，每个app-tag的ID获取相互隔离，互不影响。如果以后有性能需求需要对数据库扩容，不需要上述描述的复杂的扩容操作，只需要对app-tag分库分表就行

```sql
CREATE TABLE `segments`
(
    `app_tag`     VARCHAR(32) NOT NULL,
    `max_id`      BIGINT      NOT NULL,
    `step`        BIGINT      NOT NULL,
    `update_time` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`app_tag`)
) ENGINE = InnoDB  DEFAULT CHARSET = utf8

INSERT INTO segments(`app_tag`, `max_id`, `step`) VALUES ('test_business', 0, 100000);
```

架构图如下

![image-20211114111047924](https://i.loli.net/2021/11/14/uMlwINXBT6kJEHh.png)

test_tag在第一台Leaf机器上是1-1000的号段，当这个号段用完时，会去加载另一个长度为step=1000的号段，假设另外两台号段都没有更新，这个时候第一台机器新加载的号段就应该是3001-4000。同时数据库对应的app_tag这条数据的max_id会从3000被更新成4000，更新号段的SQL语句如下：

```sql
Begin
UPDATE table SET max_id=max_id+step WHERE app_tag=xxx
SELECT tag, max_id, step FROM table WHERE app_tag=xxx
Commit
```

优点：

- Leaf服务可以很方便的线性扩展，性能完全能够支撑大多数业务场景。
- ID号码是**趋势递增**的8byte的64位数字，满足上述数据库存储的主键要求。
- 容灾性高：Leaf服务内部有号段缓存，即使DB宕机，短时间内Leaf仍能正常对外提供服务。
- 可以自定义max_id的大小，非常方便业务从原有的ID方式上迁移过来。

缺点：

- ID号码**不够随机**，通过id号相减能获取大概的id生成数量，泄露id数量的信息，**不太安全**。
- TP999数据波动大，当号段使用完之后还是会hang在更新数据库的I/O上，tg999数据会出现偶尔的尖刺。
- **DB宕机会造成整个系统不可用**。

#### 双buffer优化

如果等到号段用完再去mysql拿新的号段，可能会阻塞业务线程，且业务响应容易受到数据库性能或网络影响。

为此，我们希望DB取号段的过程能够做到无阻塞，不需要在DB取号段的时候阻塞请求线程，即当号段消费到某个点时就异步的把下一个号段加载到内存中。而不需要等到号段用尽的时候才去更新号段。

采用**双buffer**的方式，Leaf服务内部有两个号段缓存区segment。当前号段已下发10%时，如果下一个号段未更新，则另启一个更新线程去更新下一个号段。当前号段全部下发完后，如果下个号段准备好了则切换到下个号段为当前segment接着下发，循环往复。

### 雪花算法

![image-20211114114923848](https://i.loli.net/2021/11/14/GNXkFMyxvTOj4SW.png)

`Snowflake`生成的是Long类型的ID，一个Long类型占8个字节，每个字节占8比特，也就是说一个Long类型占64个比特。Snowflake ID组成结构：`符号位`（占1比特）+ `时间戳`（占41比特）+ `机器ID`（占5比特）+ `数据中心`（占5比特）+ `自增值`（占12比特），总共64比特组成的一个Long类型。

- 符号位（1bit）Java中long的最高位是符号位代表正负，正数是0，负数是1，一般生成ID都为正数，所以默认为0
- 时间戳（41bit）毫秒级的时间，不建议存当前时间戳，而是用`当前时间戳 - 固定开始时间戳`的差值，可以使产生的ID从更小的值开始；41位的时间戳可以使用69年，(1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69年
- 工作机器id（10bit）：也被叫做`workId`，这个可以灵活配置，机房或者机器号组合都可以
- 序列号部分（12bit），自增值支持同一毫秒内同一个节点可以生成**4096**个ID

#### 雪花算法Java实现

```java
public class SnowFlakeShortUrl {

    /**
     * 起始的时间戳
     */
    private final static long START_TIMESTAMP = 1480166465631L;

    /**
     * 每一部分占用的位数
     */
    private final static long SEQUENCE_BIT = 12;   //序列号占用的位数
    private final static long MACHINE_BIT = 5;     //机器标识占用的位数
    private final static long DATA_CENTER_BIT = 5; //数据中心占用的位数

    /**
     * 每一部分的最大值
     * -1 == 1111 1111
     * -1 << 12 == 1111 1111 0000 0000 0000
     * MAX_MACHINE_NUM == -1L ^ (-1L << SEQUENCE_BIT)
     * 1111 1111 1111 1111 1111
     * ^
     * 1111 1111 0000 0000 0000
     * =
     * 0000 0000 1111 1111 1111 == MAX_SEQUENCE
     * ------------------------------------------------
     * MAX_SEQUENCE        = 0000 0000 1111 1111 1111
     * MAX_MACHINE_NUM     = 0000 0000 0000 0001 1111
     * MAX_DATA_CENTER_NUM = 0000 0000 0000 0001 1111
     * ------------------------------------------------
     */
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);
    private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
    private final static long MAX_DATA_CENTER_NUM = -1L ^ (-1L << DATA_CENTER_BIT);

    /**
     * 每一部分向左的位移
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATA_CENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTAMP_LEFT = DATA_CENTER_LEFT + DATA_CENTER_BIT;

    private long dataCenterId;  //数据中心
    private long machineId;     //机器标识
    private long sequence = 0L; //序列号
    private long lastTimeStamp = -1L;  //上一次时间戳

    private long getNextMill() {
        long mill = getNewTimeStamp();
        while (mill <= lastTimeStamp) {
            mill = getNewTimeStamp();
        }
        return mill;
    }

    private long getNewTimeStamp() {
        return System.currentTimeMillis();
    }

    /**
     * 根据指定的数据中心ID和机器标志ID生成指定的序列号
     *
     * @param dataCenterId 数据中心ID
     * @param machineId    机器标志ID
     */
    public SnowFlakeShortUrl(long dataCenterId, long machineId) {
        if (dataCenterId > MAX_DATA_CENTER_NUM || dataCenterId < 0) {
            throw new IllegalArgumentException("DtaCenterId can't be greater than MAX_DATA_CENTER_NUM or less than 0！");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("MachineId can't be greater than MAX_MACHINE_NUM or less than 0！");
        }
        this.dataCenterId = dataCenterId;
        this.machineId = machineId;
    }

    /**
     * 产生下一个ID
     *
     * @return
     */
    public synchronized long nextId() {
        long currTimeStamp = getNewTimeStamp();
        if (currTimeStamp < lastTimeStamp) {
            throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
        }

        if (currTimeStamp == lastTimeStamp) {
            //相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currTimeStamp = getNextMill();
            }
        } else {
            //不同毫秒内，序列号置为0
            sequence = 0L;
        }

        lastTimeStamp = currTimeStamp;

        return (currTimeStamp - START_TIMESTAMP) << TIMESTAMP_LEFT //时间戳部分
                | dataCenterId << DATA_CENTER_LEFT       //数据中心部分
                | machineId << MACHINE_LEFT             //机器标识部分
                | sequence;                             //序列号部分
    }
    
    public static void main(String[] args) {
        SnowFlakeShortUrl snowFlake = new SnowFlakeShortUrl(2, 3);

        for (int i = 0; i < (1 << 4); i++) {
            //10进制
            System.out.println(snowFlake.nextId());
        }
    }
}
```

#### 雪花算法时钟回拨问题

如果机器的时钟发生了回拨，那么就会有可能生成重复的ID号。

- 节点启动时，若zk节点存在，说明之前已经启动过。则用自身系统时间与`forever/${self}`节点记录时间做比较，若小于`leaf_forever/${self}`时间则认为机器时间发生了大步长回拨，服务启动失败

- 节点启动时，若zk节点不存在，说明是新增服务机器，直接创建`forever/${self}`持久子节点，并写入当前机器系统时间；并且获取`temporary根节点`下所有子节点（其他服务机器的ipPort信息），通过rpc或http接口获取到其他所有机器的时间，计算出`时间戳平均值`。
- 如果`abs(时间戳平均值-当前机器系统时间) < 阈值`，说明当前机器时间正常，启动服务并且写入到`temporary/${self}`维持租约； 否则说明当前机器发生偏移，启动失败。
- 每隔一段时间(3s)上报自身系统时间写入`leaf_forever/${self}`

由于强依赖时钟，对时间的要求比较敏感，在机器工作时NTP同步也会造成秒级别的回退，建议可以直接关闭NTP同步。要么在时钟回拨的时候直接不提供服务直接返回ERROR_CODE，等时钟追上即可。**或者做一层重试，然后上报报警系统，更或者是发现有时钟回拨之后自动摘除本身节点并报警**

```java
//发生了回拨，此刻时间小于上次发号时间
 if (timestamp < lastTimestamp) {
            long offset = lastTimestamp - timestamp;
            if (offset <= 5) {
                try {
                	//时间偏差大小小于5ms，则等待两倍时间
                    thread_wait(offset << 1);//wait
                    timestamp = getTimeStamp();
                    if (timestamp < lastTimestamp) {
                       //还是小于，抛异常并上报
                        throwClockBackwardsEx(timestamp);
                      }    
                } catch (InterruptedException e) {  
                   throw  e;
                }
            } else {
                //throw
                throwClockBackwardsEx(timestamp);
            }
        }
 //分配ID     
```

优点：全局唯一，高可用， 利用zookeeper解决时钟回拨问题，弱依赖zookeeper

缺点：依赖第三方组件，架构较为复杂

### 借助Redis

利用`redis`的 `incr`命令实现ID的原子性自增。需要考虑redis持久化的问题。

- `RDB`会定时打一个快照进行持久化，假如连续自增但`redis`没及时持久化，而这会Redis挂掉了，重启Redis后会出现ID重复的情况。
- `AOF`会对每条写命令进行持久化，即使`Redis`挂掉了也不会出现ID重复的情况，但由于incr命令的特殊性，会导致`Redis`重启恢复的数据时间过长。

优点：不依赖数据库，性能好

缺点：依赖第三方组件



参考：

[美团Leaf]: https://tech.meituan.com/2017/04/21/mt-leaf.html

