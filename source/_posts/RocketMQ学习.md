---
title: RocketMQ学习
date: 2018-09-27 21:11:57
tags: 
- 中间件
- MQ
categories: 
- 技术学习
---

今天来学一下RocketMQ

# 架构
RocketMQ分为四个角色： Producer、 Consumer、 Broker 和 NameServer
- NameServer 负责维护状态信息。集群中各个组件通过Nameserver了解全局信息并定期向NameServer上报自己的状态。
- Broker 负责存储、传输消息
- Producer 负责发送消息
- Consumer 负责接受消息，多个Consumer组成一个消费者组
- 所有角色均可以集群部署

<!-- more -->

# 在本机启动RocketMQ


# 消费

消费方式分为两种：
1. BROADCASTING 广播模式，即所有的消费者可以消费同样的消息 
2. CLUSTERING 集群模式，即所有的消费者平均来消费一组消息 

消费位置：
- 消费位置通过setConsumeFromWhere设置
- ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET 第一次启动从队列初始位置消费，后续再启动接着上次消费的进度开始消费 
- ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET 第一次启动从队列最后位置消费，后续再启动接着上次消费的进度开始消费
- ConsumeFromWhere.CONSUME_FROM_TIMESTAMP 第一次启动从指定时间点位置消费，后续再启动接着上次消费的进度开始消费 
- 第一次启动是指从来没有消费过的消费者，如果该消费者消费过，那么会在broker端记录该消费者的消费位置，如果该消费者挂了再启动，那么自动从上次消费的进度开始




