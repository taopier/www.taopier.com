---
layout: post
title: "spark源码分析(一):worker"
date: 2014-02-13 02:42:25 +0800
comments: true
categories: [code-digger, spark]
---
####it lies in spark-core, 主要就是在启动之前完成的准备工作，接收不同类型的消息进行相应的处理

<!-- more -->
  - preStart
    - 创建自己的workDir 
    - 在master上注册自己
        - 尝试在所有的master上注册：发送消息类型为RegisterWorker
        - 如果没有成功，即registered这个标志位仍然为初始的false，则一共试3次，如果某一次注册成功，完成，否则放弃
    - 更新web ui页面（建立自己的metrics和相关链接)
  - receive 根据消息类型进行相应处理
    - master相关
        - RegisteredWorker：从master接收的消息类型，注册成功，则更新自己的masterUrl，schedule发送固定间隔的SendHeartbeat类型的消息给自己
        - RegisterWorkerFailed：从master接收的消息类型，注册失败，直接退出
        - SendHeartbeat：若自己与master是连接的，则发送自己的id的心跳给master，消息类型为Heartbeat
    - driver相关
        - Heartbeat： 接收到driver的心跳信息
    - executor相关
        - LaunchExecutor
