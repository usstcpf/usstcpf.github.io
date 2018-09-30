---
title: RocketMQ学习
date: 2018-09-27 21:11:57
tags: 
- 中间件
- MQ
categories: 
- RocketMQ学习笔记
---

消息队列，即MQ-Message Queue，是“先进先出”的一种数据结构。分布式消息队列可以提供应用解耦、 流量消峰、消息分发等功能，已经成为大型互联网服务架构里标配的中间件。

- 应用解耦：比如一条订单产生后需要通知支付、库存、物流系统，即时一个系统挂掉，也不影响其他系统。
- 流量消峰：一次能接收海量请求，但可以一部分一部分处理
- 消息分发：一条消息发给多个系统

常见的消息队列有ActiveMQ，Kafka，RocketMQ等。今天来学习RocketMQ，RocketMQ使用Java 语言开发，目前是Apache顶级项目，在阿里被大面积使用。

# 架构

RocketMQ分为四个角色： Producer、 Consumer、 Broker 和 NameServer

- NameServer 负责维护状态信息。集群中各个组件通过Nameserver了解全局信息并定期向NameServer上报自己的状态。
- Broker 负责存储、传输消息
- Producer 负责发送消息
- Consumer 负责接受消息，多个Consumer组成一个消费者组
- 所有角色均可以集群部署

<!-- more -->

# 在IDE中启动RocketMQ

官方给出的例子是使用命令来启动，但是在开发时一般会使用IDE直接main函数，如果要在IDE中启动main函数，需要做一些额外工作。

## 启动broker

启动类是`org.apache.rocketmq.broker.BrokerStartup`，但如果直接启动该类，会报错`Please set the ROCKETMQ_HOME variable in your environment to match the location of the RocketMQ installation`，需要做一些初始化操作才可以正常启动

1. 在createBrokerController方法中设置rocketmqHome ```brokerConfig.setRocketmqHome("D:\\workspace\\ideaworkspace\\RocketMQ-iqiyi\\distribution");```
1. 在distribution\conf\broker.conf中添加一行配置 ```brokerIP1={yourIP}```
1. 通过 ```-c D:\workspace\ideaworkspace\RocketMQ\distribution\conf\broker.conf``` 指定配置文件路径
1. broker启动需要指定nameServer的地址，可以在BrokerStartup中设置```brokerConfig.setNamesrvAddr```，或者通过命令行参数 ```-n yournameserverAddress:port```，或者在broker.conf配置文件中配置```namesrvAddr={yournamesrvAddr}```
1. 接下来直接启动BrokerStartup即可

**注意** 如果不做第2、3步，极有可能在生产和消费的时候报错```connect to XXXX:10909 failed```

# 消费

## 消费组

RocketMQ中的消费是以消费者组的形式来进行的，一个或多个消费者的集合是一个消费者组，消费者必须指定其所在的组

## 消费方式

两种消费方式：

1. BROADCASTING 广播模式，即所有的消费者可以消费同样的消息，每条消息会被多次分发，被多个Consumer消费
2. CLUSTERING 默认为集群模式，即所有的消费者平均来消费一组消息。同一个ConsumerGroup里的每个Consumer只消费所订阅消息的一部分内容，同一个 ConsumerGroup里所有的Consumer消费的内容合起来才是所订阅Topic 内容的整体，从而达到负载均衡的目的。

## 消费位置

请多注意最后一条说明

- 消费者可以通过setConsumeFromWhere设置消费起始位置
- ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET 第一次启动从队列初始位置消费，后续再启动接着上次消费的进度开始消费 
- ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET 第一次启动从队列最后位置消费，后续再启动接着上次消费的进度开始消费
- ConsumeFromWhere.CONSUME_FROM_TIMESTAMP 第一次启动从指定时间点位置消费，后续再启动接着上次消费的进度开始消费 
- **第一次启动是指从来没有消费过的消费者，如果该消费者消费过，那么会在broker端记录该消费者的消费位置，如果该消费者挂了再启动，那么自动从上次消费的进度开始**

## 消费者类型

一种是DefaultMQPushConsumer，一种是DefaultMQPullConsumer。两者区别在于push类型的由系统控制读取操作，收到消息后自动调用传入的处理方法来处理，而pull类型的大部分功能由使用者自己控制。

### DefaultMQPushConsumer

首先看一下DefaultMQPushConsumer的使用，查看```org.apache.rocketmq.example.quickstart```下面的Consumer类

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

- 必须设定consumerGroupName
- 必须指定topic，可以通过```consumer.subscribe("TopicTest", "tag1 || tag2 || tag3")```过滤tag。null或*表示不过滤，消费所有信息。
- 必须指定NameServer地址
- 可以指定从特定位置消费，默认```ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET```，具体规则见[消费位置](#消费位置)
- 可以指定消费模式，默认为```MessageModel.CLUSTERING```

查看DefaultMQPushConsumer代码可以看到它的所有具体实现都是由DefaultMQPullConsumerImpl实现的

### DefaultMQPullConsumer

再来看DefaultMQPullConsumer的使用，查看```org.apache.rocketmq.example.simple```下面的PullConsumer类

```
public class PullConsumer {
    private static final Map<MessageQueue, Long> OFFSE_TABLE = new HashMap<MessageQueue, Long>();
    public static void main(String[] args) throws MQClientException {
        DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("please_rename_unique_group_name_5");
        consumer.start();
        Set<MessageQueue> mqs = consumer.fetchSubscribeMessageQueues("TopicTest1");
        for (MessageQueue mq : mqs) {
            System.out.printf("Consume from the queue: %s%n", mq);
            SINGLE_MQ:
            while (true) {
                try {
                    PullResult pullResult =
                        consumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32);
                    System.out.printf("%s%n", pullResult);
                    putMessageQueueOffset(mq, pullResult.getNextBeginOffset());
                    switch (pullResult.getPullStatus()) {
                        case FOUND:
                            break;
                        case NO_MATCHED_MSG:
                            break;
                        case NO_NEW_MSG:
                            break SINGLE_MQ;
                        case OFFSET_ILLEGAL:
                            break;
                        default:
                            break;
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        consumer.shutdown();
    }
    private static long getMessageQueueOffset(MessageQueue mq) {
        Long offset = OFFSE_TABLE.get(mq);
        if (offset != null)
            return offset;
        return 0;
    }
    private static void putMessageQueueOffset(MessageQueue mq, long offset) {
        OFFSE_TABLE.put(mq, offset);
    }
}
```

## 没分类的

MQClientInstance负责与Broker联系，处于生产者和消费者的底层。生产者和消费者都需要创建MQClientInstance，一般连接同一个rocketMQ只会创建一个MQClientInstance，也就是在同一个JVM，默认多个Producer和Consumer公用一个MQClientInstance。在连接多个RocketMQ时，一定要手动指定不同的InstanceName，底层会创建多个MQClientInstance对象，设定方法为```producer.setInstanceName("");```或 ```consumer.setInstanceName("");```


# 参考资料

- 《RocketMQ实战与原理解析》 杨开元 著