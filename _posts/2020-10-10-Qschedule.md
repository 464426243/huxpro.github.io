---
layout:     post
title:      分布式调用组件Qschedule介绍
author:     "冬至未"
header-style: text
tags:
    - Qunar
    - Qschdule
---



# Qschedule

公司自研的分布式调度组件 

## 功能：

1. 客户端可以在服务端注册任务，并声明自身为该任务的候选执行者
2. 有一个管理界面可以人工控制任务相关参数，包括执行时间，任务执行参数，执行者列表等
3. 服务端负责在指定时间从任务候选执行者中选择一名执行者执行任务，并监控任务状态。

## 实现

包括三大模块 客户端api 服务端 管理界面

### 客户端

客户端负责注册和执行任务，在执行中定期向服务器报告任务执行状态。注册通过dubbo调用向server端注册自己，客户端执行任务通过接受服务端的QMQ触发，通过一个CachedThreadPool(核心线程数为0，最大数量无限制，存活时间60s，队列使用同步队列)执行，任务结束后向server端报告任务结束或失败。任务执行状态信息采用“推拉结合”的方式，在执行过程中会通过另一个线程执行心跳响应，并定期报告所有任务执行状态。如果客户端长时间未响应，server端也会主动向客户端发送

### 管理界面

管理界面不多说，可以人工控制任务相关参数，包括执行时间，任务执行参数，执行者列表等

### 服务端

服务端负责任务注册，定时调度，心跳检测，同时支持高可用。

集群启动后选举一台作为leader，server实际是单点的，且没说有没有自动晋升功能。

当某台机器成为Leader后，启动任务调度线程池**Schedule**，作业跟踪线程**Jobtracker**，任务跟踪线程**TaskTracker**。
启动作业注册服务**JobRegister**、进度跟踪服务**ProcessTracker**、日志存储服务**Logger**、任务完成后Ack确认服务**AckService**等。
作业跟踪线程**Jobtracker** 开始查找需要调度（已启动并设置了cron表达式）的Job，纳入Job容器中管理。且每分钟探测一次Db和Job容器同步

任务跟踪线程**TaskTracker**开始查找未完成（正在调度和调度过程中出现错误）的Job，Ping客户端确认调度结果，对出错的任务使用失败恢复策略**FailedRecoverStrategy**处理并向业务线报警。且每间隔1分钟执行一次。

**JobRegistry**通过dubbo接口接收注册并写入到server db

进度跟踪服务**ProcessTracker**接收客户端的任务信息

日志存储服务**Logger**接收客户端的日志并存入Hbase

任务完成后Ack确认服务**AckService**接收客户端完成消息并修改任务状态。