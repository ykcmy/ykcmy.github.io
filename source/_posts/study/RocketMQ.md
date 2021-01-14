---
title: rocketMq学习
date: 2020-12-15 10:40:38
categories: study
tags: [MQ]
---

# RokectMQ学习

### 1.概念

nameServer: Name Server是一个几乎无状态节点，可集群部署，节点之间无任何信息同步

消息：字节数组

**设计理念**：

**基于主题**，发布和订阅

**核心功能**：消息发送，消息存储（持久化方式：磁盘），消息消费

**设计目标**：

- 顺序消息
- 消息过滤（客户端实现，服务端实现）
- 消息存储 从两个维度：消息堆积（顺序读写在一个文件，存储空间预警机制，过期文件删除机制）；存储性能

#### 参数

#### **nameserver**

相对来说，nameserver的稳定性非常高。原因有二:

1. nameserver互相独立，彼此没有通信关系，单台nameserver挂掉，不影响其他nameserver，即使全部挂掉，也不影响业务系统使用，这点类似于dubbo的zookeeper。
2. nameserver不会有频繁的读写，所以性能开销非常小，稳定性很高。

#### **broker**

##### **1 与nameserver关系**

单个broker和所有nameserver保持长连接

**心跳间隔**：每隔30秒（**此时间无法更改**）向所有nameserver发送心跳，心跳包含了自身的topic配置信息。

**心跳超时**：nameserver每隔10秒钟（**此时间无法更改**），扫描所有还存活的broker连接，若某个连接2分钟内（**当前时间与最后更新时间差值超过2分钟，此时间无法更改**）没有发送心跳数据，则断开连接。

**时机**：broker挂掉；心跳超时导致nameserver主动关闭连接

**动作**：一旦连接断开，nameserver会立即感知，更新topc与队列的对应关系，但不会通知生产者和消费者

**2 负载均衡**
一个topic分布在多个broker上，一个broker可以配置多个topic，它们是多对多的关系。
如果某个topic消息量很大，应该给它多配置几个队列，并且尽量多分布在不同broker上，减轻某个broker的压力。
topic消息量都比较均匀的情况下，如果某个broker上的队列越多，则该broker压力越大。

**3 可用性**
由于消息分布在各个broker上，一旦某个broker宕机，则该broker上的消息读写都会受到影响。所以rocketmq提供了master/slave的结构，salve定时从master同步数据，如果master宕机，则slave提供消费服务，但是不能写入消息，此过程对应用透明，由rocketmq内部解决。
这里有两个关键点：
一旦某个broker master宕机，生产者和消费者多久才能发现？受限于rocketmq的网络连接机制，默认情况下，最多需要30秒，但这个时间可由应用设定参数来缩短时间。这个时间段内，发往该broker的消息都是失败的，而且该broker的消息无法消费，因为此时消费者不知道该broker已经挂掉。
消费者得到master宕机通知后，转向slave消费，但是slave不能保证master的消息100%都同步过来了，因此会有少量的消息丢失。但是消息最终不会丢的，一旦master恢复，未同步过去的消息会被消费掉。

**4 可靠性**
所有发往broker的消息，有同步刷盘和异步刷盘机制，总的来说，可靠性非常高
同步刷盘时，消息写入物理文件才会返回成功，因此非常可靠
异步刷盘时，只有机器宕机，才会产生消息丢失，broker挂掉可能会发生，但是机器宕机崩溃是很少发生的，除非突然断电

#### 消费者

**1 与nameserver关系**
**连接**
单个消费者和一台nameserver保持长连接，定时查询topic配置信息，如果该nameserver挂掉，消费者会自动连接下一个nameserver，直到有可用连接为止，并能自动重连。
**心跳**
与nameserver没有心跳
**轮询时间**
默认情况下，消费者每隔30秒从nameserver获取所有topic的最新队列情况，这意味着某个broker如果宕机，客户端最多要30秒才能感知。该时间由DefaultMQPushConsumer的pollNameServerInteval参数决定，可手动配置。

**2 与broker关系**
**连接**：
单个消费者和该消费者关联的所有broker保持长连接。
**心跳**：
默认情况下，消费者每隔30秒向所有broker发送心跳，该时间由DefaultMQPushConsumer的heartbeatBrokerInterval参数决定，可手动配置。broker每隔10秒钟（此时间无法更改），扫描所有还存活的连接，若某个连接2分钟内（当前时间与最后更新时间差值超过2分钟，此时间无法更改）没有发送心跳数据，则关闭连接，并向该消费者分组的所有消费者发出通知，分组内消费者重新分配队列继续消费
**断开**：
时机：消费者挂掉；心跳超时导致broker主动关闭连接
动作：一旦连接断开，broker会立即感知到，并向该消费者分组的所有消费者发出通知，分组内消费者重新分配队列继续消费

**3 消费机制**
**本地队列**
消费者不间断的从broker拉取消息，消息拉取到本地队列，然后本地消费线程消费本地消息队列，只是一个异步过程，拉取线程不会等待本地消费线程，这种模式实时性非常高。对消费者对本地队列有一个保护，因此本地消息队列不能无限大，否则可能会占用大量内存，本地队列大小由DefaultMQPushConsumer的pullThresholdForQueue属性控制，默认1000，可手动设置。

**4 如果一个topic在某broker上有3个队列，一个消费者消费这3个队列，那么该消费者和这个broker有几个连接？**
一个连接，消费单位与队列相关，消费连接只跟broker相关，事实上，消费者将所有队列的消息拉取任务放到本地的队列，挨个拉取，拉取完毕后，又将拉取任务放到队尾，然后执行下一个拉取任务

#### 生产者
**1 与nameserver关系**
**连接**
单个生产者者和一台nameserver保持长连接，定时查询topic配置信息，如果该nameserver挂掉，生产者会自动连接下一个nameserver，直到有可用连接为止，并能自动重连。
**轮询时间**
默认情况下，生产者每隔30秒从nameserver获取所有topic的最新队列情况，这意味着某个broker如果宕机，生产者最多要30秒才能感知，在此期间，发往该broker的消息发送失败。该时间由DefaultMQProducer的pollNameServerInteval参数决定，可手动配置。
**心跳**
与nameserver没有心跳

**2 与broker关系**
**连接**
单个生产者和该生产者关联的所有broker保持长连接。
**心跳**
默认情况下，生产者每隔30秒向所有broker发送心跳，该时间由DefaultMQProducer的heartbeatBrokerInterval参数决定，可手动配置。broker每隔10秒钟（此时间无法更改），扫描所有还存活的连接，若某个连接2分钟内（当前时间与最后更新时间差值超过2分钟，此时间无法更改）没有发送心跳数据，则关闭连接。
**连接断开**
移除broker上的生产者信息

**3 负载均衡**
生产者时间没有关系，每个生产者向队列轮流发送消息

**其他参数概念**：

**Topic**:一级消息主题（类型）（例如服装订单消息，支付消息）

**Tags**:二级消息类型 （具体分类，例如男装，女装）

什么时候用topic? 什么时候用tag?

- 消息类型是否一致？不同，用topic区分
- 业务是否相关联，没有关系用topic区分，有关系用tag区分
- 消息的优先级是否一致，不同优先级用不同topic区分
- 消息量是否相当 不同用topic区分

**Message key**:业务层面唯一标识，rocket 会建立key与消息的映射（hash索引）适用解决幂等问题

**MessageID**:rocketMq全局唯一标识（内部机制id，机器ip，消息偏移量）

三种发送方式：

- 单向发送 ：最快，数据有可能丢失  适用日志收集
- 同步可靠发送 ：快 数据不丢失 消息量较小  通知，短信
- 异步可靠发送  ： 视频上传等

### 消费者

**群组消费**：一个队列只能被一个消费者消费

**广播消费**：每个消费者都消费所有消息

#### 消费方式：

**拉模式**：需要自己处理queue，自己保存偏移量，灵活，代码多

```java
public class PullConsumer {
    private static final Map<MessageQueue, Long> OFFSE_TABLE = new HashMap<MessageQueue, Long>();

    public static void main(String[] args) throws MQClientException {
        //拉模式
        DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("pullconsumer");
        consumer.setNamesrvAddr("192.168.0.128:9876");
        //consumer.setBrokerSuspendMaxTimeMillis(1000);

        System.out.println("ms:"+consumer.getBrokerSuspendMaxTimeMillis());
        consumer.start();

        //1.获取MessageQueues并遍历（一个Topic包括多个MessageQueue  默认4个）
        Set<MessageQueue> mqs = consumer.fetchSubscribeMessageQueues("TopicTest");
        for (MessageQueue mq : mqs) {
            System.out.println("queueID:"+ mq.getQueueId());
            //获取偏移量
            long Offset = consumer.fetchConsumeOffset(mq,true);

            System.out.printf("Consume from the queue: %s%n", mq);
            SINGLE_MQ:
            while (true) { //拉模式，必须无限循环
                try {
                    PullResult pullResult =
                        consumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32);
                    System.out.printf("%s%n",pullResult);
                    //2.维护Offsetstore（这里存入一个Map）
                    putMessageQueueOffset(mq, pullResult.getNextBeginOffset());

                    //3.根据不同的消息状态做不同的处理
                    switch (pullResult.getPullStatus()) {
                        case FOUND: //获取到消息
                            for (int i=0;i<pullResult.getMsgFoundList().size();i++) {
                                System.out.printf("%s%n", new String(pullResult.getMsgFoundList().get(i).getBody()));
                            }
                            break;
                        case NO_MATCHED_MSG: //没有匹配的消息
                            break;
                        case NO_NEW_MSG:  //没有新消息
                            break SINGLE_MQ;
                        case OFFSET_ILLEGAL: //非法偏移量
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

**推模式**: 本质上是用拉模式封装的

```java
public static void main(String[] args) throws InterruptedException, MQClientException {
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
    consumer.subscribe("TopicTest", "*");
    consumer.setNamesrvAddr("192.168.0.128:9876");
    consumer.setAllocateMessageQueueStrategy(new AllocateMessageQueueAveragely());
    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);//每次从最后一次消费的地址
    consumer.registerMessageListener(new MessageListenerConcurrently() {
        @Override
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
            System.out.printf("queueID:%d:%s:Messages:%s %n",  msgs.get(0).getQueueId(),Thread.currentThread().getName(), new String(msgs.get(0).getBody()));
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
    });
    consumer.start();
    System.out.printf("ConsumerPartOrder Started.%n");
}
```

#### 消息发送流程

- 验证消息
- 查找路由
- 消息发送

#### 消息队列负载

- 平均分配 AllocateMessageQueueAveragely  （默认情况使用）
- 平均轮询分配 AllocateMessageQueueAveragelyByCircle
- 一致性hash
- 根据配置
- 根据broker部署的机房名

#### 重新分布机制

RebalanceService 每隔20s进行一次队列负载

#### 消息确认（ACK）

确保消费者成功 ConsumeConcurrentlyStatus

每次去消费的时候，记录消费进度，只会把一批（最小的offset进行确认存储），不去一条条提交偏移量

问题：会出现重复消费，在业务中进行幂等处理。

#### 失败重试

首先进入%RETRY%consumerGroup

重新发送到原来的队列（延迟10s）

```java
consumer.setMaxReconsumeTimes(3);//重试次数,默认15次
```

超过重试次数，进入DLQ队列

### 顺序消息

```java
//排序的监听事件
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
        System.out.printf("%s:Messages:%s %n", Thread.currentThread().getName(), new String(msgs.get(0).getBody()));
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

#### RocketMq事务消息

```java
public static void main(String[] args) throws MQClientException, InterruptedException {
    TransactionListener transactionListener = new TransactionListenerImpl();
    //支持事务的生产者 TransactionMQProducer
    TransactionMQProducer producer = new TransactionMQProducer("transaction_producer");
    producer.setNamesrvAddr("192.168.0.128:9876");
    //设置用于事务消息的处理线程池
    ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r);
            thread.setName("client-transaction-msg-check-thread");
            return thread;
        }
    });

    producer.setExecutorService(executorService);
    //设置事务监听器，监听器实现接口org.apache.rocketmq.client.producer.TransactionListener
    //监听器中实现需要处理的交易业务逻辑的处理，以及MQ Broker中未确认的事务与业务的确认逻辑
    producer.setTransactionListener(transactionListener);
    producer.start();

    //生成不同的Tag，用于模拟不同的处理场景
    String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
    for (int i = 0; i < 10; i++) {//10条消息
        try {
            //组装产生消息
            Message msg =
                    new Message("TopicTransaction", tags[i % tags.length], "KEY" + i,
                            ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            //以事务发送消息，并在事务消息被成功预写入到RocketMQ中后，执行用户定义的交易逻辑，
            //交易逻辑执行成功后，再实现实现业务消息的提交逻辑
            SendResult sendResult = producer.sendMessageInTransaction(msg, null);
            System.out.printf("%s%n", sendResult);
            Thread.sleep(10);
        } catch (MQClientException | UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
    for (int i = 0; i < 100000; i++) {
        Thread.sleep(1000);
    }
    producer.shutdown();
}
```

```java
public static void main(String[] args) throws InterruptedException, MQClientException {
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("transaction_producer");
    consumer.setNamesrvAddr("192.168.0.128:9876");
    /**
     * 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费<br>
     * 如果非第一次启动，那么按照上次消费的位置继续消费
     */
    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
    consumer.subscribe("TopicTransaction", "*");
    consumer.registerMessageListener(new MessageListenerOrderly() {
        private Random random = new Random();

        @Override
        public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
            // 设置自动提交
            context.setAutoCommit(true);
            for (MessageExt msg : msgs) {
                System.out.println("获取到消息开始消费："+msg + " , content : " + new String(msg.getBody()));
            }
            try {
                // 模拟业务处理
                TimeUnit.SECONDS.sleep(random.nextInt(5));
            } catch (Exception e) {
                e.printStackTrace();
                //返回处理失败，该消息后续可以继续被消费
                return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
            }
            //返回处理成功，该消息就不会再次投递过来了
            return ConsumeOrderlyStatus.SUCCESS;
        }
    });
    consumer.start();
    System.out.println("consumer start ! ");
}
```

```java
public class TransactionListenerImpl implements TransactionListener {
    private AtomicInteger transactionIndex = new AtomicInteger(0);
    //使用transactionId
    private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();

    //TODO 执行事务
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        //TODO 执行本地事务开始
        int value = transactionIndex.getAndIncrement();
        //业务处理
        System.out.println("执行本地事务..."+value);
        int status = value % 3;
        localTrans.put(msg.getTransactionId(), status);
        switch (status) {
            case 0:
                //LocalTransactionState.UNKNOW表示未知的事件，需要RocketMQ进一步服务业务进行确认该交易的处理
                return LocalTransactionState.UNKNOW;
            case 1:
                return LocalTransactionState.COMMIT_MESSAGE;
            case 2:
                return LocalTransactionState.ROLLBACK_MESSAGE; //这条消息抛弃了（账户余额不足 1W）
            default:
                return LocalTransactionState.COMMIT_MESSAGE;
        }
        //TODO 执行本地事务结束
    }
    //该方法用于RocketMQ与业务确认未提交事务的消息的状态（一分钟执行一次）
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        System.out.println("事务回查-----UNKNOW-----------");
        Integer status = localTrans.get(msg.getTransactionId());
        //业务处理（1分钟）
        int mod = msg.getTransactionId().hashCode() % 2;
        if (null != status) {
            switch (mod) {
                case 0:
                    return LocalTransactionState.ROLLBACK_MESSAGE;
                case 1:
                    return LocalTransactionState.COMMIT_MESSAGE;
                default:
                    return LocalTransactionState.COMMIT_MESSAGE;
            }
        }
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```