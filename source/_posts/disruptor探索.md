---
title: disruptor探索
date: 2021-10-30 16:33:03
tags: disruptor
categories: 多线程
---

## 从Logger4j2引出Disruptor

最近把日志框架从logback切换到了log4j2，号称在多线程环境下性能优于log4j和logback数十倍，下面是log4j2官网提供的经典峰值吞吐量对比图。Log4j 2的 Async Loggers 支持使用无锁数据结构（Disruptor），而 Logback、 Log4j 则只支持 ArrayBlockingQueue，多线程下无法避免锁争用，从而导致吞吐量大大降低。
![image-20211031110132988](https://i.loli.net/2021/10/31/HzdbZD95w4CFip1.png)



## Disruptor是什么

> Disruptor是一个高性能的无锁并发框架，提供了一种线程消息传递的方式。

