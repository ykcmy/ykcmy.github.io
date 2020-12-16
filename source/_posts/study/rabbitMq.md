---
title: rabbitMq学习
date: 2020-12-15 10:40:38
categories: study
tags: [MQ]
---

## rabbitMq：

> 优点：解耦，异步，缓冲，伸缩性，扩展性
>
> 与Rpc的区别：Rpc同步，mq异步		

### rabbitMq概念：

- Broker:标识消息队列服务器实体。
- Virtual Host：虚拟主机， 每个vhost本质上就是一个mini版的RabbitMQ服务器。
- Exchange：交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。
- Queue:消息队列，用来保存消息直到发送给消费者，一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。
- Banding：绑定，用于消息队列和交换机之间的关联，一个绑定就是基于路由键将交换机和消息队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。
- Channel：信道，多路复用连接中的一条独立的双向数据流通道

## Exchange分发策略

- direct:消息中的路由键如果和Binding中的binding key一致，交换器就将消息发到对应的队列中。路由键 		      				   与队列名完全匹配。(完全匹配)

- fanout(广播):每个发到fanout类型交换器的消息都会分到所有绑定的队列上去

- topic:必须由（.）分割的单词列表。比如apple.banana.orange

  > 可以代替一个单词。
  >
  >  \#可以代替零个或多个单词。



<img src="/image/截图.png" width="70%">

# 交换器（原生客户端）

### Direct交换器：

#### product:

```java
//创建连接、连接到RabbitMQ
ConnectionFactory connectionFactory= new ConnectionFactory();
//设置下连接工厂的连接地址(使用默认端口5672)
connectionFactory.setHost("127.0.0.1");

//创建连接
Connection connection =connectionFactory.newConnection();
//创建信道
Channel channel =connection.createChannel();

//在信道中设置交换器
channel.exchangeDeclare("direct_logs",BuiltinExchangeType.DIRECT);

//申明队列（放在消费者中去做）
//申明路由键\消息体
String[] routeKeys ={"aaa","bbb","ccc"};
for (int i=0;i<6;i++){
    String routeKey = routeKeys[i%3];
    String msg = "Hello,RabbitMQ"+(i+1);
    
//发布消息 
 channel.basicPublish("direct_logs",routeKey,null,msg.getBytes());
    System.out.println("Sent:"+routeKey+":"+msg);
}
channel.close();
connection.close();
```

#### consumer:

```java
//创建连接、连接到RabbitMQ
ConnectionFactory connectionFactory= new ConnectionFactory();
//设置下连接工厂的连接地址(使用默认端口5672)
connectionFactory.setHost("127.0.0.1");

//创建连接
Connection connection =connectionFactory.newConnection();
//创建信道
Channel channel =connection.createChannel();

//在信道中设置交换器
channel.exchangeDeclare("direct_logs",BuiltinExchangeType.DIRECT);

//申明队列（放在消费者中去做）
String queueName="queue-a";
channel.queueDeclare(queueName,false,false,false,null);

//绑定：将队列(queuq-a)与交换器通过 路由键 绑定(aaa)
String routeKey ="aaa";
channel.queueBind(queueName,DirectProducer.EXCHANGE_NAME,routeKey);
System.out.println("waiting for message ......");

//申明一个消费者
final Consumer consumer  = new DefaultConsumer(channel){
    @Override
    public void handleDelivery(String s, Envelope envelope, AMQP.BasicProperties basicProperties, byte[] bytes) throws IOException {
        String message = new String(bytes,"UTF-8");
        System.out.println("Received["+envelope.getRoutingKey()+"]"+message);
    }
};
//消息者正是开始在指定队列上消费。(queue-a)
//TODO 这里第二个参数是自动确认参数，如果是true则是自动确认
channel.basicConsume(queueName,true,consumer);
```

### fanout 交换器

routeKey无论设置为任何键，消息都会发到每一个队列

### Topic交换器

>  \*与#的区别：如果我们发送的路由键变成king.kafka.A,那么队列中如果绑定了king.* 不能匹配队列中如果绑定了king.# 能够匹配

```java
//TODO 根据需求绑定不同routeKey
channel.queueBind(queueName,TopicProducer.EXCHANGE_NAME, "king.#");
```

# 消息发布和消费的权衡

## 发布时的权衡：

在RabbitMQ 中，有不同的投递机制（生产者），但是每一种机制都对性能有一定的影响。一般来讲速度快的可靠性低，可靠性好的性能差，具体怎么使用需要根据你的应用程序来定，所以说没有最好的方式，只有最合适的方式。只有把你的项目和技术相结合，才能找到适合你的平衡。

<img src="/image/WechatIMG8525.png" width="100%">

#### 1.无保障：

_在演示各种交换器中使用的就是无保障的方式，通过basicPublish 发布你的消息并使用正确的交换器和路由信息，你的消息会被接收并发送到合适的队列中。但是如果有网络问题，或者消息不可路由，或者RabbitMQ 自身有问题的话，这种方式就有风险。所以无保证的消息发送一般情况下不推荐。_

#### 2.失败确认

_在发送消息时设置mandatory 标志，告诉RabbitMQ，如果消息不可路由，应该将消息返回给发送者，并通知失败。可以这样认为，开启mandatory是开启故障检测模式。_

> 注意：它只会让RabbitMQ 向你通知失败，而不会通知成功。如果消息正确路由到队列，则发布者不会受到任何通知。带来的问题是无法确保发布消息一定是成功的，因为通知失败的消息可能会丢失。

```java

//失败通知 回调
channel.addReturnListener(new ReturnListener() {
    public void handleReturn(int replycode, String replyText, String exchange, String routeKey, AMQP.BasicProperties basicProperties, byte[] bytes) throws IOException {
        String message = new String(bytes);
        System.out.println("返回的replycode:"+replycode);
    }
});
//TODO 第三个参数mandatory设置为true,开启故障检测模式 
channel.basicPublish(EXCHANGE_NAME,routekey,true,null,message.getBytes());
```

#### 3.事务

_事务的实现主要是对信道（Channel）的设置，主要的方法有三个：_

1. channel.txSelect()声明启动事务模式；

2. channel.txComment()提交事务；

3. channel.txRollback()回滚事务；

_在发送消息之前，需要声明channel 为事务模式，提交或者回滚事务即可。开启事务后，客户端和RabbitMQ 之间的通讯交互流程：_

- 客户端发送给服务器Tx.Select(开启事务模式)

- 服务器端返回Tx.Select-Ok（开启事务模式ok）

- 推送消息

- 客户端发送给事务提交Tx.Commit
-  服务器端返回Tx.Commit-Ok

```java
//加入事务
channel.txSelect();
try {
    for(int i=0;i<3;i++){
        String routekey = routekeys[i%3];
        // 发送的消息
        String message = "Hello World_"+(i+1)
                +("_"+System.currentTimeMillis());
        channel.basicPublish(EXCHANGE_NAME, routekey, true,
                null, message.getBytes());
        System.out.println("Message: [" + message + "]");
        Thread.sleep(200);
    }
    //事务提交
    channel.txCommit();
} catch (IOException e) {
    e.printStackTrace();
    //TODO
    //事务回滚
    channel.txRollback();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

> 那么，既然已经有事务了，为何还要使用发送方确认模式呢，原因是因为事务的性能是非常差的。根据相关资料，事务会降低2~10 倍的性能。

#### 4.发送发确认模式：

_基于事务的性能问题，RabbitMQ 团队为我们拿出了更好的方案，即采用发送方确认模式，该模式比事务更轻量，性能影响几乎可以忽略不计。原理：生产者将信道设置成confirm 模式，一旦信道进入confirm 模式，所有在该信道上面发布的消息都将会被指派一个唯一的ID(从1 开始)，由这个id 在生产者和RabbitMQ 之间进行消息的确认。不可路由的消息，当交换器发现，消息不能路由到任何队列，会进行确认操作，表示收到了消息。如果发送方设置了mandatory 模式,则会先调用addReturnListener 监听器。_

_可路由的消息，要等到消息被投递到所有匹配的队列之后，broker 会发送一个确认给生产者(包含消息的唯一ID)，这就使得生产者知道消息已经正确到达目的队列了，如果消息和队列是可持久化的，那么确认消息会在将消息写入磁盘之后发出，broker 回传给生产者的确认消息中delivery-tag 域包含了确认消息的序列号。_

_confirm 模式最大的好处在于他可以是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用便可以通过回调方法来处理该确认消息，如果RabbitMQ 因为自身内部错误导致消息丢失，就会发送一条nack 消息，生产者应用程序同样可以在回调方法中处理该nack 消息决定下一步的处理。_

##### Confirm 的三种实现方式：

- 方式一：channel.waitForConfirms()普通发送方确认模式；消息到达交换器，就会返回true。

```java
// 启用发送者确认模式
channel.confirmSelect();
```

```java
//TODO
//确认是否成功(true成功)
if(channel.waitForConfirms()){
    System.out.println("send success");
}else{
    System.out.println("send failure");
}
```

- 方式二：channel.waitForConfirmsOrDie()批量确认模式；使用同步方式等所有的消息发送之后才会执行后面代码，只要有一个消息未到达交换器就会抛出IOException 异常。

```java
// 启用发送者确认模式
channel.confirmSelect();
```

```java
// 启用发送者确认模式（批量确认）
channel.waitForConfirmsOrDie();
```

- 方式三：channel.addConfirmListener()异步监听发送方确认模式；

  ```java
  //TODO启用发送者确认模式 
  channel.confirmSelect();
  //TODO添加发送者确认监听器
  channel.addConfirmListener(new ConfirmListener() {
      //TODO 成功
      public void handleAck(long deliveryTag, boolean multiple)
              throws IOException {     System.out.println("send_ACK:"+deliveryTag+",multiple:"+multiple);}
      //TODO 失败
      public void handleNack(long deliveryTag, boolean multiple)
              throws IOException {
          System.out.println("Erro----send_NACK:"+deliveryTag+",multiple:"+multiple);
      }
  });
  ```

#### 5.备用交换器

_在第一次声明交换器时被指定，用来提供一种预先存在的交换器，如果主交换器无法路由消息，那么消息将被路由到这个新的备用交换器。使用备用交换器，向往常一样，声明Queue 和备用交换器，把Queue 绑定到备用交换器上。然后在声明主交换器时，通过交换器的参数，alternate-exchange,，将备用交换器设置给主交换器。_

```java
// 声明备用交换器
Map<String,Object> argsMap = new HashMap<String,Object>();
argsMap.put("alternate-exchange","ae");
//主交换器
channel.exchangeDeclare("main-exchange","direct",
        false,false,argsMap);
//备用交换器
channel.exchangeDeclare("ae",BuiltinExchangeType.FANOUT,
        true,false,null);
```

## 消息的消费权衡

<img src="/image/WechatIMG8528.png" width="80%">

### 消息的获取方式

#### 拉取Get

_属于一种轮询模型，发送一次get 请求，获得一个消息。如果此时RabbitMQ 中没有消息，会获得一个表示空的回复。总的来说，这种方式性能比较差，很明显，每获得一条消息，都要和RabbitMQ 进行网络通信发出请求。而且对RabbitMQ 来说，RabbitMQ 无法进行任何优化，因为它永远不知道应用程序何时会发出请求。具体使用，参见代码native 模块包cn.enjoyedu.consumer_balance.GetMessage 中。对我们实现者来说，要在一个循环里，不断去服务器get 消息。_

```java
//TODO 无限循环拉取
while(true){
    //拉一条，自动确认的(rabbit 认为这条消息消费 -- 从队列中删除)
    GetResponse getResponse = channel.basicGet(queueName, false);
    if(null!=getResponse){
        System.out.println("received["
                +getResponse.getEnvelope().getRoutingKey()+"]"
                +new String(getResponse.getBody()));
    }
    //确认(自动、手动)
    channel.basicAck(0,true);
    Thread.sleep(1000);
}
```

#### 推送Consume

_属于一种推送模型。注册一个消费者后，RabbitMQ 会在消息可用时，自动将消息进行推送给消费者。_

#### 消息的应答

_前面说过，消费者收到的每一条消息都必须进行确认。消息确认后，RabbitMQ 才会从队列删除这条消息，RabbitMQ 不会为未确认的消息设置超时时间，它判断此消息是否需要重新投递给消费者的唯一依据是**消费该消息的消费者连接是否已经断开**。这么设计的原因是RabbitMQ 允许消费者消费一条消息的时间可以很久很久。_

#### 自动确认

_消费者在声明队列时，可以指定autoAck 参数，当autoAck=true 时，一旦消费者接收到了消息，就视为自动确认了消息。如果消费者在处理消息的过程中，出了错，就没有什么办法重新处理这条消息，所以我们很多时候，需要在消息处理成功后，再确认消息，这就需要手动确认。_

#### 手动确认

​		_当autoAck=false 时，RabbitMQ 会等待消费者显式发回ack 信号后才从内存(和磁盘，如果是持久化消息的话)中移去消息。否则，RabbitMQ 会在队列中消息被消费后立即删除它。_	

​		_采用消息确认机制后，只要令autoAck=false，消费者就有足够的时间处理消息(任务)，不用担心处理消息过程中消费者进程挂掉后消息丢失的问题，因为RabbitMQ 会一直持有消息直到消费者显式调用basicAck 为止。_

​		_当autoAck=false 时，对于RabbitMQ 服务器端而言，队列中的消息分成了两部分：一部分是等待投递给消费者的消息；一部分是已经投递给消费者，但是还没有收到消费者ack 信号的消息。如果服务器端一直没有收到消费者的ack 信号，并且消费此消息的消费者已经断开连接，则服务器端会安排该消息重新进入队列，等待投递给下一个消费者（也可能还是原来的那个消费者）。_

```java
        /*声明了一个消费者*/
        final Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag,
 Envelope envelope,AMQP.BasicProperties properties, byte[] body) throws IOException {
   try {
     String message = new String(body, "UTF-8");                   System.out.println("Received"+message);
                    //确认
channel.basicAck(envelope.getDeliveryTag(),false);
   } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                    //拒绝
//Reject方式拒绝，第二个参数决定是否重新投递
channel.basicReject(envelope.getDeliveryTag(), true)
                }
            }
        };
        /*消费者正式开始在指定队列上消费消息*/
        //TODO 这里第二个参数是自动确认参数，如果是false则是手动确认
        channel.basicConsume(queueName,false,consumer);
```

#### QoS 预取模式

_在确认消息被接收之前，消费者可以预先要求接收一定数量的消息，在处理完一定数量的消息后，批量进行确认。如果消费者应用程序在确认消息之前崩溃，则所有未确认的消息将被重新发送给其他消费者。所以这里存在着一定程度上的可靠性风险。这种机制一方面可以实现限速（将消息暂存到RabbitMQ 内存中）的作用，一方面可以保证消息确认质量（比如确认了但是处理有异常的情况）。_

> 注意：消费确认模式必须是非自动ACK 机制（这个是使用baseQos 的前提条件，否则会Qos 不生效），然后设置basicQos 的值；另外，还可以基于consume 和channel 的粒度进行设置（global）。

```java
/**
 * 批量确认 -----消费者
 */
public class BatchAckConsumer extends DefaultConsumer {
    //计数，第多少条
    private  int meesageCount =0;
    public BatchAckConsumer(Channel channel) {
        super(channel);
        System.out.println("批量消费者启动了......");
    }

    @Override
    public void handleDelivery(String consumerTag,
      Envelope envelope,AMQP.BasicProperties properties,
     byte[] body) throws IOException {
        //把消息体拉出来
        String message = new String(body,"UTF-8");
 System.out.println(message);
        meesageCount++;
        //批量确认 50一批
        if(meesageCount %50 ==0){        this.getChannel().basicAck(envelope.getDeliveryTag(),true);
 System.out.println("批量消息费进行消息的确认------------");
        }
      //如果是最后一条消息，则把剩余的消息都进行确认
       if(message.equals("stop")){         this.getChannel().basicAck(envelope.getDeliveryTag(),true);
System.out.println("批量消费者进行最后业务消息的确认---------");
        }
    }
}
```

## 消息的拒绝

> Reject 和 N a c k  

​        消息确认可以让RabbitMQ 知道消费者已经接受并处理完消息。但是如果消息本身或者消息的处理过程出现问题怎么办？需要一种机制，通知RabbitMQ，这个消息，我无法处理，请让别的消费者处理。这里就有两种机制，Reject 和Nack。Reject 在拒绝消息时，可以使用requeue 标识，告诉RabbitMQ 是否需要重新发送给别的消费者。如果是false 则不重新发送，一般这个消息就会被RabbitMQ 丢弃。Reject 一次只能拒绝一条消息。如果是true 则消息发生了重新投递。

​        Nack 跟Reject 类似，只是它可以一次性拒绝多个消息。也可以使用requeue 标识，这是RabbitMQ 对AMQP 规范的一个扩展。具体使用，当requeue参数设置为true 时，消息发生了重新投递。当requeue 参数设置为false 时，消息丢失了。

```java
//Reject方式拒绝，第二个参数决定是否重新投递
channel.basicReject(envelope.getDeliveryTag(), true)
  
//Nack方式拒绝，第二个参数决定是否批量,第三参数是否重新投递
channel.basicNack(envelope.getDeliveryTag(), false, false)
```

## 死信交换器（DLX）

> 1.过期消息
>
> 2.消息队列达到最大长度，最开始的投递的消息
>
> 3.拒绝且requeue = false的消息

``` java
//绑定死信交换器
channel.exchageDeclare(DlxProducer.EXCHAGE_NAME, BuiltinExchageType.TOPIC);
//申明队列
String queueName = "dlx_make";
Map<String, Object> args = new HashMap<>();
args.put("x-dead-letter-exchage", "死信交换器名称");
//TODO 死信路由键会替换原来路由键
channel.queueDeclare(queueName, false,true,false,args);
//将队列和交换器通过路由键绑定
channel.queueBind(queueName,DlxProducer.EXCHAGE_NAME, "#");
```

#### **死信交换器和备用交换器的区别**

> 备用交换器是备胎：没有队列要它的时候只能选择备用交换器。死信交换器是老实人：消息已经被消费者玩过一遍，消费者没有消费，只能被死信交换器接收。

### 控制队列

#### 临时队列

_临时队列对应的是没有持久化的队列，也就是如果RabbitMQ 服务器重启，那么这些队列就不会存在，所以我们称之为临时队列。_

#### 自动删除队列

_自动删除队列和普通队列在使用上没有什么区别，唯一的区别是，当消费者断开连接时，队列将会被删除。自动删除队列允许的消费者没有限制，也就是说当这个队列上最后一个消费者断开连接才会执行删除。_

> 自动删除队列只需要在声明队列时，设置属性auto-delete 标识为true 即可。系统声明的随机队列，缺省就是自动删除的。

```java
//第四个参数
channel.queueDeclare(queueName,false,false, false,null);
```

#### 单消费者队列

_普通队列允许的消费者没有限制，多个消费者绑定到多个队列时，RabbitMQ 会采用轮询进行投递。如果需要消费者独占队列，在队列创建的时候，设定属性exclusive 为true。_

```java
//第三个参数
channel.queueDeclare(queueName,false,false, false,null);
```

#### 自动过期队列

_指队列在超过一定时间没使用，会被从RabbitMQ 中被删除。_

_什么是没使用？1.一定时间内没有Get 操作发生。2.没有Consumer 连接在队列上。特别的：就算一直有消息进入队列，也不算队列在被使用。_

```java
//TODO /*自动过期队列--参数需要Map传递*/
String queueName = "setQueue";
Map<String, Object> arguments = new HashMap<String, Object>();
arguments.put("x-expires",10*1000);//10秒被删除
//TODO 队列的各种参数
/*加入队列的各种参数*/
channel.queueDeclare(queueName,true,true, false,arguments);
```

#### 永久队列

_持久化队列和非持久化队列的区别是，持久化队列会被保存在磁盘中，固定并持久的存储，当Rabbit 服务重启后，该队列会保持原来的状态在RabbitMQ中被管理，而非持久化队列不会被保存在磁盘中，Rabbit 服务重启后队列就会消失。_

```java
//将属性durable 设置为“true”，
channel.queueDeclare(queueName,true,false, false,null);
```

#### 队列级别消息过期

_就是为每个队列设置消息的超时时间。只要给队列设置x-message-ttl 参数，就设定了该队列所有消息的存活时间，时间单位是毫秒。如果声明队列时指定了死信交换器，则过期消息会成为死信消息。_

```java
//TODO /*自动过期队列--参数需要Map传递*/
String queueName = "setQueue";
Map<String, Object> arguments = new HashMap<String, Object>();
arguments.put("x-expires",10*1000);//10秒被删除
//设定该队列消息存活时间，时间单位毫秒
arguments.put("x-message-ttl",10*1000);//消息10秒被删除
//TODO 队列的各种参数
/*加入队列的各种参数*/
channel.queueDeclare(queueName,true,true, false,arguments);
```

- 对队列中消息的条数进行限制x-max-length

- 对队列中消息的总量进行限制x-max-length-bytes

### 消息的持久化

_默认情况下，队列和交换器在服务器重启后都会消失，消息当然也是。将队列和交换器的durable 属性设为true，缺省为false，但是消息要持久化还不够，还需要将消息在发布前，将投递模式设置为2。消息要持久化，必须要有持久化的队列、交换器和投递模式都为2。_

```java
//TODO 创建持久化交换器 durable=true
channel.exchangeDeclare(EXCHANGE_NAME,"direct",true);
//TODO 发布持久化的消息(delivery-mode=2)
channel.basicPublish(EXCHANGE_NAME,routekey,
        MessageProperties.PERSISTENT_TEXT_PLAIN,
        msg.getBytes());
```

```java
//TODO 声明一个持久化队列(durable=true)
String queueName = "msgdurable";
channel.queueDeclare(queueName,true,false,
        false,null);
```

### Request-Response 模式

​		_我们前面的学习模式中都是一方负责发送消息而另外一方负责处理。而我们实际中的很多应用相当于一种一应一答的过程，需要双方都能给对方发送消息。于是请求-应答的这种通信方式也很重要。它也应用的很普遍。_

```java
//TODO 响应QueueName ，消费者将会把要返回的信息发送到该Queue
String responseQueue = channel.queueDeclare().getQueue();
//TODO 消息的唯一id
String msgId = UUID.randomUUID().toString();
//TODO 设置消息中的应答属性
AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
  //replyTO构建回复消息的私有响应队列
        .replyTo(responseQueue)
        .messageId(msgId)
        .build();
 /*声明了一个消费者*/
final Consumer consumer = new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag,
                                       Envelope envelope,
                                       AMQP.BasicProperties properties,
                                       byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println("Received["+envelope.getRoutingKey()
                        +"]"+message);
            }
};
//TODO 消费者应答队列上的消息
channel.basicConsume(responseQueue,true,consumer);
 String msg = "Hello,RabbitMq";
 //TODO 发送消息时，把响应相关属性设置进去
 channel.basicPublish(EXCHANGE_NAME,"error",properties,
          msg.getBytes());
System.out.println("Sent error:"+msg);
```

```java
/*绑定，将队列和交换器通过路由键进行绑定*/
String routekey = "error";/*表示只关注error级别的日志消息*/
channel.queueBind(queueName,ReplyToProducer.EXCHANGE_NAME,routekey);

System.out.println("waiting for message........");

/*声明了一个消费者*/
final Consumer consumer = new DefaultConsumer(channel){
    @Override
    public void handleDelivery(String consumerTag,
                               Envelope envelope,
                               AMQP.BasicProperties properties,
                               byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println("Received["+envelope.getRoutingKey()
                +"]"+message);
        //TODO 从消息中拿到相关属性（确定要应答的消息ID,）
        AMQP.BasicProperties respProp
                = new AMQP.BasicProperties.Builder()
                .replyTo(properties.getReplyTo())
                .correlationId(properties.getMessageId())
                .build();
        //TODO 消息消费时，同时需要生作为生产者生产消息（以OK为标识）
        channel.basicPublish("", respProp.getReplyTo() ,
                respProp ,
                ("OK,"+message).getBytes("UTF-8"));
   }
};
/*消费者正式开始在指定队列上消费消息*/
channel.basicConsume(queueName,true,consumer);
```