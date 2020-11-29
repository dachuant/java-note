# 1. RabbitMQ简介

​	RabbitMQ是基于AMQP（Advanced Message Queuing Protocol）的一款消息队列，用Erlang语言编写，支持多种客户端。

# 2. 基本概念

![img](https://gitee.com/dachuant/image/raw/master/picgo/20201129204128.jpg) 

参考 [https://zhuanlan.zhihu.com/p/63700605](https://zhuanlan.zhihu.com/p/63700605)

- Producer：即数据的发送方

	> Create messages and publish (send) them to a Broker Server
	
	一般一个Message有两个部分：payload（有效载荷）和label（标签），payload顾名思义就是传输的数据，label是exchange的名字或者说是一个tag，它描述了payload，而且RabbitMQ也是通过这个label来决定把这个Message发给哪个Consumer
	
- Exchange：即**rabbitmq**内的消息交换器

	> Exchanges are message routing agents， An exchange accepts messages from the producer application and routes them to message queues with help of header attributes, bindings, and routing keys.
	
	exchange从生产者那收到消息后，一般会指定一个Routing Key，来指定这个消息的路由规则，当然Routing Key需要与Exchange Type及Binding key联合使用才能最终生效，根据路由规则，匹配查询表中的routing key，分发消息到queue中。
	
- binding：即绑定

  > A **binding** is a "link" that you set up to bind a queue to an exchange

  在绑定（Binding）Exchange与Queue的同时，一般会指定一个Binding key;但Binding key并不是在所有情况下都生效，它依赖于Exchange Type

- Queue：即队列

  是rabbitmq内部对象，用于存储消息；

  消息最终被送到这里等待consumer取走。一个message可以被同时拷贝到多个queue中。

  那么谁应该负责创建这个queue呢？是Consumer，还是Producer？

	> 如果queue不存在，当然Consumer不会得到任何的Message。那么Producer Publish的Message会被丢弃。所以，还是为了数据不丢失，Consumer和Producer都try to create the queue！反正不管怎么样，这个接口都不会出问题。
	> queue对load balance的处理是完美的。对于多个Consumer来说，RabbitMQ 使用循环的方式（round-robin）的方式均衡的发送给不同的Consumer。

- Connection与Channel：两者都是RabbitMQ对外提供的API中最基本的对象

	> A Connection represents a real TCP connection to the message broker, whereas a Channel is a virtual connection (AMPQ connection) inside it
	
	**Connection**就是一个TCP的连接，Producer和Consumer都是通过TCP连接到RabbitMQ Server的
	
	**Channel**是建立在上述的TCP连接中，因为建立TCP Connection的开销将是巨大的，所以是节省开销；
	
	Channel是我们与RabbitMQ打交道的最重要的一个接口，我们大部分的业务操作是在Channel这个接口中完成的，包括定义Queue、定义Exchange、绑定Queue与Exchange、发布消息等
	
- Consumer：即数据的接收方

  > Consumers attach to a Broker Server (RabbitMQ) and subscribe to a queue。

  如果有多个消费者同时订阅同一个Queue中的消息，Queue中的消息会被平摊给多个消费者

- Broker: 即RabbitMQ Server

  > RabbitMQ isn't a food truck, it's a delivery service

  其作用是维护一条从Producer到Consumer的路线，保证数据能够按照指定的方式进行传输

- Virtual host：即虚拟主机

  当多个不同的用户使用同一个RabbitMQ server提供的服务时，可以划分出多个vhost，每个用户在自己的vhost创建exchange queue

# 3. 消息模型

参考

[https://blog.csdn.net/qq_38762237/article/details/89416444](https://blog.csdn.net/qq_38762237/article/details/89416444)

[https://www.cnblogs.com/ithushuai/p/12443460.html](https://www.cnblogs.com/ithushuai/p/12443460.html)

## 3.1. 基本消息模型

![img](https://gitee.com/dachuant/image/raw/master/picgo/20201129234328.png)

- P：生产者，也就是要发送消息的程序
- C：消费者：消息的接受者，会一直等待消息到来。
- queue：消息队列，图中红色部分。类似一个邮箱，可以缓存消息；生产者向其中投递消息，消费者从其中取出消息。

基本模型中，一个生产者，一个消费者，生产的消息直接被消费者消费。

## 3.2. Work Queues工作队列模型



![img](https://gitee.com/dachuant/image/raw/master/picgo/20201129234838.png)

Work Queues工作队列模型中，一个生产发送消息到队列，允许有多个消费者接收消息，但是一条消息只会被一个消费者获取。

## 3.3. 订阅模型 - Fanout

![img](https://gitee.com/dachuant/image/raw/master/picgo/20201129235100.png)

-  可以有多个消费者
- 每个**消费者有自己的queue**（队列）
- 每个**队列都要绑定到Exchange**（交换机）
- **生产者发送的消息，只能发送到交换机**，交换机来决定要发给哪个队列，生产者无法决定。
- 交换机把消息发送给绑定过的所有队列
- 队列的消费者都能拿到消息。实现一条消息被多个消费者消费

## 3.4. 订阅模型 -  Direct

![img](https://gitee.com/dachuant/image/raw/master/picgo/20201129235331.png)

在Direct模式下，交换机不再将消息发送给所有绑定的队列，而是根据Routing Key将消息发送到指定的队列，队列在与交换机绑定时会设定一个Routing Key，而生产者发送的消息时也需要携带一个Routing Key。

## 3.5. 订s阅模型 -  Topic

![在这里插入图片描述](https://gitee.com/dachuant/image/raw/master/picgo/20201129235434.png)

`Topic`类型的`Exchange`与`Direct`相比，都是可以根据`RoutingKey`把消息路由到不同的队列。只不过`Topic`类型`Exchange`可以让队列在绑定`Routing key` 的时候使用通配符！

Routingkey一般都是有一个或多个单词组成，多个单词之间以”.”分割，例如：item.insert

通配符规则：

 `#`：匹配一个或多个词
​ `*`：匹配不多不少恰好1个词

# 4. RabbitMq + Spring boot 配置参数


| 属性名 | 含义 |
| ------ | ---- |
|spring.rabbitmq.host| 服务Host|
|spring.rabbitmq.port| 服务端口|
|spring.rabbitmq.username| 登陆用户名|
|spring.rabbitmq.password| 登陆密码|
|spring.rabbitmq.virtual-host| 连接到rabbitMQ的vhost|
|spring.rabbitmq.addresses| 指定client连接到的server的地址，多个以逗号分隔(优先取addresses，然后再取host)|
|spring.rabbitmq.requested-heartbeat| 指定心跳超时，单位秒，0为不指定；默认60s|
|spring.rabbitmq.publisher-confirms| 是否启用【发布确认】|
|spring.rabbitmq.publisher-returns| 是否启用【发布返回】|
|spring.rabbitmq.connection-timeout| 连接超时，单位毫秒，0表示无穷大，不超时|
|spring.rabbitmq.parsed-addresses||
|spring.rabbitmq.ssl.enabled|是否支持ssl|
|spring.rabbitmq.ssl.key-store| 指定持有SSL certificate的key store的路径|
|spring.rabbitmq.ssl.key-store-password| 指定访问key store的密码|
|spring.rabbitmq.ssl.trust-store| 指定持有SSL certificates的Trust store|
|spring.rabbitmq.ssl.trust-store-password| 指定访问trust store的密码|
|spring.rabbitmq.ssl.algorithm| ssl使用的算法，例如，TLSv1.1|
|spring.rabbitmq.cache.channel.size| 缓存中保持的channel数量|
|spring.rabbitmq.cache.channel.checkout-timeout| 当缓存数量被设置时，从缓存中获取一个channel的超时时间，单位毫秒；如果为0，则总是创建一个新channel|
|spring.rabbitmq.cache.connection.size| 缓存的连接数，只有是CONNECTION模式时生效|
|spring.rabbitmq.cache.connection.mode| 连接工厂缓存模式：CHANNEL 和 CONNECTION|
|spring.rabbitmq.listener.simple.auto-startup| 是否启动时自动启动容器|
|spring.rabbitmq.listener.simple.acknowledge-mode| 表示消息确认方式，其有三种配置方式，分别是none、manual和auto；默认auto|
|spring.rabbitmq.listener.simple.concurrency| 最小的消费者数量|
|spring.rabbitmq.listener.simple.max-concurrency| 最大的消费者数量|
|spring.rabbitmq.listener.simple.prefetch| 指定一个请求能处理多少个消息，如果有事务的话，必须大于等于transaction数量.|
|spring.rabbitmq.listener.simple.transaction-size| 指定一个事务处理的消息数量，最好是小于等于prefetch的数量.|
|spring.rabbitmq.listener.simple.default-requeue-rejected| 决定被拒绝的消息是否重新入队；默认是true（与参数acknowledge-mode有关系）|
|spring.rabbitmq.listener.simple.idle-event-interval| 多少长时间发布空闲容器时间，单位毫秒|
|spring.rabbitmq.listener.simple.retry.enabled| 监听重试是否可用|
|spring.rabbitmq.listener.simple.retry.max-attempts| 最大重试次数|
|spring.rabbitmq.listener.simple.retry.initial-interval| 第一次和第二次尝试发布或传递消息之间的间隔|
|spring.rabbitmq.listener.simple.retry.multiplier| 应用于上一重试间隔的乘数|
|spring.rabbitmq.listener.simple.retry.max-interval| 最大重试时间间隔|
|spring.rabbitmq.listener.simple.retry.stateless| 重试是有状态or无状态|
|spring.rabbitmq.template.mandatory| 启用强制信息；默认false|
|spring.rabbitmq.template.receive-timeout| receive() 操作的超时时间|
|spring.rabbitmq.template.reply-timeout| sendAndReceive() 操作的超时时间|
|spring.rabbitmq.template.retry.enabled| 发送重试是否可用|
|spring.rabbitmq.template.retry.max-attempts| 最大重试次数|
|spring.rabbitmq.template.retry.initial-interval| 第一次和第二次尝试发布或传递消息之间的间隔|
|spring.rabbitmq.template.retry.multiplier| 应用于上一重试间隔的乘数|
|spring.rabbitmq.template.retry.max-interval| 最大重试时间间隔|