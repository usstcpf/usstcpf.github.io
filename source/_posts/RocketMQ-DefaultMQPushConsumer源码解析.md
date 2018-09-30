---
title: RocketMQ-DefaultMQPushConsumer源码解析
date: 2018-09-30 11:03:23
tags: 
- 中间件
- MQ
categories: 
- RocketMQ学习笔记
---

之前讲过，RocketMQ消费者有两种，push和pull，push方法由broker向client推送消息，是最常用的消费者，它的实现类是DefaultMQPushConsumer。本篇介绍一下DefaultMQPushConusmer的启动流程和处理逻辑。

<!-- more -->

# 类结构

## DefaultMQPushConsumer

DefaultMQPushConsumer在org.apache.rocketmq.client.consumer包中，承担上层接口角色，用户只与DefaultMQPushConsumer打交道即可。

## DefaultMQPushConsumerImpl

DefaultMQPushConsumerImpl在org.apache.rocketmq.client.impl.consumer包中，DefaultMQPushConsumer的各个逻辑实现都需要依赖DefaultMQPushConsumerImpl实现。

## ClientConfig

ClientConfig在org.apache.rocketmq.client中，是一个基础配置类，所有的Producer和所有的Consumer类都继承自ClientConfig，它配置了consumer和produce的公用配置。其中buildMQClientId()和changeInstanceNameToPID()可以留意一下，后面会用到。

```java
public class ClientConfig {
    public static final String SEND_MESSAGE_WITH_VIP_CHANNEL_PROPERTY = "com.rocketmq.sendMessageWithVIPChannel";
    private String namesrvAddr = System.getProperty(MixAll.NAMESRV_ADDR_PROPERTY, System.getenv(MixAll.NAMESRV_ADDR_ENV));
    private String clientIP = RemotingUtil.getLocalAddress();
    private String instanceName = System.getProperty("rocketmq.client.name", "DEFAULT");
    private int clientCallbackExecutorThreads = Runtime.getRuntime().availableProcessors();
    private int pollNameServerInterval = 1000 * 30;
    private int heartbeatBrokerInterval = 1000 * 30;
    private int persistConsumerOffsetInterval = 1000 * 5;
    private boolean unitMode = false;
    private String unitName;
    private boolean vipChannelEnabled = Boolean.parseBoolean(System.getProperty(SEND_MESSAGE_WITH_VIP_CHANNEL_PROPERTY, "true"));
    private boolean useTLS = TlsSystemConfig.tlsEnable;
    private LanguageCode language = LanguageCode.JAVA;
    public String buildMQClientId() {
        StringBuilder sb = new StringBuilder();
        sb.append(this.getClientIP());
        sb.append("@");
        sb.append(this.getInstanceName());
        if (!UtilAll.isBlank(this.unitName)) {
            sb.append("@");
            sb.append(this.unitName);
        }
        return sb.toString();
    }
    public void changeInstanceNameToPID() {
        if (this.instanceName.equals("DEFAULT")) {
            this.instanceName = String.valueOf(UtilAll.getPid());
        }
    }
    public void resetClientConfig(final ClientConfig cc) {
        this.namesrvAddr = cc.namesrvAddr;
        //set all fileds
    }
    public ClientConfig cloneClientConfig() {
        ClientConfig cc = new ClientConfig();
        //copy all fields
        return cc;
    }
    //other getter and setter
}
```

## MQClientManager

MQClientManager在org.apache.rocketmq.client.impl包中，负责创建和维护MQClientInstance

## MQClientInstance

MQClientInstance是客户端各种类型的Consumer和Producer的底层类，它会从NameServer获取并保存各种配置信息，比如Tipic的Route信息。同时会通过MQClientAPIImpl类实现消息的收发。它的创建是由MQClientManager负责的。

# 启动流程

首先看一下Consumer的入口

```
public class Consumer {
    public static void main(String[] args) throws InterruptedException, MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");
        consumer.setNamesrvAddr("name-server1-ip:9876;name-server2-ip:9876");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("TopicTest", "*");
        consumer.setMessageModel(MessageModel.BROADCASTING);
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```

看一下DefaultMQPushConsumer构造方法

```java
public DefaultMQPushConsumer(final String consumerGroup) {
    this(consumerGroup, null, new AllocateMessageQueueAveragely());
}

public DefaultMQPushConsumer(final String consumerGroup, RPCHook rpcHook,
    AllocateMessageQueueStrategy allocateMessageQueueStrategy) {
    this.consumerGroup = consumerGroup;
    this.allocateMessageQueueStrategy = allocateMessageQueueStrategy;
    defaultMQPushConsumerImpl = new DefaultMQPushConsumerImpl(this, rpcHook);
}
```

实际调用的是有三个参数的构造方法。consumerGroup是消费者组名；AllocateMessageQueueStrategy是消息分配策略，默认采用queue平均分配策略；RPCHook是一个切面类，里面有两个方法，分别在发送请求前和收到response时执行。

```java
public interface RPCHook {
    void doBeforeRequest(final String remoteAddr, final RemotingCommand request);
    void doAfterResponse(final String remoteAddr, final RemotingCommand request,
        final RemotingCommand response);
}
```

