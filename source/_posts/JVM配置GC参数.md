---
title: JVM配置GC参数
date: 2021-06-07 23:15:58
tags: GC
categories: Java
---

##### GC日志配置参考

```java
# 必备
-XX:+PrintGCDetails 
-XX:+PrintGCDateStamps
# 打印对象年龄分布情况
-XX:+PrintTenuringDistribution 
# 每次GC时，对比一下GC前后堆内存情况
-XX:+PrintHeapAtGC 
# 打印STW时间,是GC最重要的指标
-XX:+PrintGCApplicationStoppedTime
    
# GC日志输出的文件路径，%t表示以时间戳命名文件，避免gc日志每次重启被覆盖
-Xloggc:/path/to/gc-%t.log
# 开启日志文件分割
-XX:+UseGCLogFileRotation 
# 最多分割几个文件，超过之后从头文件开始写
-XX:NumberOfGCLogFiles=14
# 每个文件上限大小，超过就触发分割
-XX:GCLogFileSize=100M
```

