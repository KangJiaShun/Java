<h1 style=" text-align: center; color: cornflowerblue">JAVA</h1>

# **Kafka知识总结**

## **一、讲讲acks参数对消息持久化的影响**

### **目录**

1. 写在前面
2. 如何保证宕机时数据不丢失？
3. 多副本之间数据如何同步？
4. ISR到底指的是什么东西？
5. acks参数的含义？
6. 最后的思考

### **1.写在前面**

面试大厂时，一旦简历上写了Kafka，几乎必然会被问到一个问题：说说acks参数对消息持久化的影响？

这个acks参数在kafka的使用中，是非常核心以及关键的一个参数，决定了很多东西。

所以无论是为了面试还是实际项目使用，大家都值得看一下这篇文章对Kafka的acks参数的分析，以及背后的原理。

### **2.如何保证宕机的时候数据不丢失？（或者kafka如何保证高可用、或者Kafka如何保证高可用）**

- Kafka 一个最基本的架构认识：由多个 broker 组成，每个 broker 是一个节点；创建一个 topic，这个 topic 可以划分为多个 partition，每个 partition 可以存在于不同的 broker 上，每个 partition 就放一部分数据。

  这就是**天然的分布式消息队列**，就是说一个 topic 的数据，是**分散放在多个机器上的，每个机器就放一部分数据**。

- 而且Kafka还提供replica**副本机制**，每个partition的数据都会同步到其他机器上，形成自己的多个replica副本。所有replica会选举出来一个leader出来，那么**生产和消费都跟这个leader打交道**，然后其他replica就是follower。写的时候，leader会负责把数据同步到所有follower上去，读的时候就直接读leader上的数据即可。

如果某个broker宕机了，那个broker上的partition在其他机器上都有副本。如果这个宕机的broker上面有某个partition的leader，那么从follower中重新选举一个新的leader出来，然后继续读写新的leader即可，这就是所谓的高可用。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/1.jpg>)

### 3.**多副本之间数据如何保证同步**

其实任何一个Partition，只有Leader是对外提供读写服务的，也就是说，如果有一个客户端往一个Partition写入数据，此时一般就是写入这个Partition的Leader副本。

然后Leader副本接收到数据之后，Follower副本会不停的给他发送请求尝试去拉取最新的数据，拉取到自己本地后，写入磁盘中。如下图所示：

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/2.jpg>)

### **4.ISR到底指的是什么东西？**

ISR全称是“In-Sync Replicas”，也就是**保持同步的副本**，他的含义就是，跟Leader始终保持同步的Follower有哪些。

大家可以想一下 ，如果说某个Follower所在的Broker因为JVM FullGC之类的问题，导致自己卡顿了，无法及时从Leader拉取同步数据，那么是不是会导致Follower的数据比Leader要落后很多？

所以这个时候，就意味着Follower已经跟Leader不再处于同步的关系了。但是只要Follower一直及时从Leader同步数据，就可以保证他们是处于同步的关系的。

所以每个Partition都有一个ISR，这个ISR里一定会有Leader自己，因为Leader肯定数据是最新的，然后就是那些跟Leader保持同步的Follower，也会在ISR里。

### **5.acks参数的含义**

首先这个acks参数，是在KafkaProducer，也就是生产者客户端里设置的

也就是说，你往kafka写数据的时候，就可以来设置这个acks参数。然后这个参数实际上有三种常见的值可以设置，分别是：**0、1 和 all**。

**第一种选择是把acks参数设置为0**，意思就是我的KafkaProducer在客户端，只要把消息发送出去，不管那条数据有没有在哪怕Partition Leader上落到磁盘，我就不管他了，直接就认为这个消息发送成功了。

如果你采用这种设置的话，那么你必须注意的一点是，可能你发送出去的消息还在半路。结果呢，Partition Leader所在Broker就直接挂了，然后结果你的客户端还认为消息发送成功了，此时就会**导致这条消息就丢失了**。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/3.jpg>)

**第二种选择是设置 acks = 1**，意思就是说只要Partition Leader接收到消息而且写入本地磁盘了，就认为成功了，不管他其他的Follower有没有同步过去这条消息了。

这种设置其实是**kafka默认的设置**

也就是说，默认情况下，你要是不管acks这个参数，只要Partition Leader写成功就算成功。

但是这里有一个问题，万一Partition Leader刚刚接收到消息，Follower还没来得及同步过去，结果Leader所在的broker宕机了，此时也会导致这条消息丢失，因为人家客户端已经认为发送成功了。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/4.jpg>)

**最后一种情况，就是设置acks=all**，这个意思就是说，**Partition Leader接收到消息之后，还必须要求ISR列表里跟Leader保持同步的那些Follower都要把消息同步过去**，才能认为这条消息是写入成功了。

如果说Partition Leader刚接收到了消息，但是结果Follower没有收到消息，此时Leader宕机了，那么客户端会感知到这个消息没发送成功，他会重试再次发送消息过去。

此时可能Partition 2的Follower变成Leader了，此时ISR列表里只有最新的这个Follower转变成的Leader了，那么只要这个新的Leader接收消息就算成功了。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/5.jpg>)

### **6.最后的思考**

acks=all 就可以代表数据一定不会丢失了吗？

当然不是，如果你的Partition只有一个副本，也就是一个Leader，任何Follower都没有，你认为acks=all有用吗？

当然没用了，因为ISR里就一个Leader，他接收完消息后宕机，也会导致数据丢失。

所以说，**这个acks=all，必须跟ISR列表里至少有2个以上的副本配合使用**，起码是有一个Leader和一个Follower才可以。

这样才能保证说写一条数据过去，一定是2个以上的副本都收到了才算是成功，此时任何一个副本宕机，不会导致数据丢失。

**参考**：https://mp.weixin.qq.com/s/IxS46JAr7D9sBtCDr8pd7A

## 二、Kafka参数调优实战

### 目录

1. 背景引入：很多同学看不懂的Kafka参数
2. 一段Kafka生产端的示例代码
3. 内存缓冲的大小
4. 多少数据打包为一个Batch合适？
5. 要是一个Batch迟迟无法凑满怎么办？
6. 最大请求大小
7. 重试机制
8. 持久化机制

#### 1、背景引入：很多同学看不懂的kafka参数

在使用Kafka的客户端编写代码与服务器交互的时候，是需要对客户端设置很多的参数的。

#### 2、一段Kafka生产端的示例代码

```scala
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092"); 
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("buffer.memory", 67108864); 
props.put("batch.size", 131072); 
props.put("linger.ms", 100); 
props.put("max.request.size", 10485760); 
props.put("acks", "1"); 
props.put("retries", 10); 
props.put("retry.backoff.ms", 500);

KafkaProducer<String, String> producer = new KafkaProducer<String, String>(props);
```

#### 3、内存缓冲的大小

首先看看“**buffer.memory**”这个参数是什么意思？

Kafka的客户端发送数据到服务器，一般都是要经过**缓冲**的，也就是说，**通过KafkaProducer发送出去的消息都是先进入到客户端本地的内存缓冲里，然后把很多消息收集成一个一个的Batch，再发送到Broker上去的**。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/6.jpg>)

所以这个“**buffer.memory”的本质就是用来约束KafkaProducer能够使用的内存缓冲的大小的，他的默认值是32MB**。

你可以先想一下，如果这个内存缓冲设置的过小的话，可能会导致一个什么问题？

首先要明确一点，那就是在内存缓冲里大量的消息会缓冲在里面，形成一个一个的Batch，每个Batch里包含多条消息。

然后KafkaProducer有一个Sender线程会把多个Batch打包成一个Request发送到Kafka服务器上去。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/7.jpg>)

那么如果要是**内存设置的太小**，可能**导致一个问题**：消息快速的写入内存缓冲里面，但是Sender线程来不及把Request发送到Kafka服务器。

这样是不是会造成内存缓冲很快就被写满？一旦被写满，就会阻塞用户线程，不让继续往Kafka写消息了。

所以对于“buffer.memory”这个参数应该结合自己的实际情况来进行压测，你需要测算一下在生产环境，你的用户线程会以每秒多少消息的频率来写入内存缓冲。

比如说每秒300条消息，那么你就需要压测一下，假设内存缓冲就32MB，每秒写300条消息到内存缓冲，是否会经常把内存缓冲写满？经过这样的压测，你可以调试出来一个合理的内存大小。

#### 4、多少数据打包为一个Batch合适？

接着你需要思考第二个问题，就是你的“**batch.size**”应该如何设置？**这决定了你的每个Batch要存放多少数据就可以发送出去了**。

比如说你要是给一个Batch设置成是16KB的大小，那么里面凑够16KB的数据就可以发送了。

这个**参数的默认值是16KB**，一般可以尝试把这个参数调节大一些，然后利用自己的生产环境发消息的负载来测试一下。

比如说发送消息的频率就是每秒300条，那么如果比如“batch.size”调节到了32KB，或者64KB，是否可以提升发送消息的整体吞吐量。

因为理论上来说，提升batch的大小，可以允许更多的数据缓冲在里面，那么一次Request发送出去的数据量就更多了，这样吞吐量可能会有所提升。

但是**不能无限的大**，过于大了之后，要是数据老是缓冲在Batch里迟迟不发送出去，那么岂不是你发送消息的延迟就会很高，**导致高延迟问题**。

比如说，一条消息进入了Batch，但是要等待5秒钟Batch才凑满了64KB，才能发送出去。那这条消息的延迟就是5秒钟。

所以需要在这里按照生产环境的发消息的速率，调节不同的Batch大小自己测试一下最终出去的吞吐量以及消息的 延迟，设置一个最合理的参数。

#### 5、要是一个Batch迟迟无法凑满怎么办？

要是一个Batch迟迟无法凑满，此时就需要引入另外一个参数了，“**linger.ms**”

**含义是一个Batch被创建之后，最多过多久，不管这个Batch有没有写满，都必须发送出去了**。

给大家举个例子，比如说batch.size是16kb，但是现在某个低峰时间段，发送消息很慢。

这就导致可能Batch被创建之后，陆陆续续有消息进来，但是迟迟无法凑够16KB，难道此时就一直等着吗？

当然不是，假设你现在设置“linger.ms”是50ms，那么只要这个Batch从创建开始到现在已经过了50ms了，哪怕他还没满16KB，也要发送他出去了。

所以“linger.ms”决定了你的消息一旦写入一个Batch，最多等待这么多时间，他一定会跟着Batch一起发送出去。

避免一个Batch迟迟凑不满，导致消息一直积压在内存里发送不出去的情况。**这是一个很关键的参数。**

这个参数一般要非常慎重的来设置，要配合batch.size一起来设置。

举个例子，首先假设你的Batch是32KB，那么你得估算一下，正常情况下，一般多久会凑够一个Batch，比如正常来说可能20ms就会凑够一个Batch。

那么你的linger.ms就可以设置为25ms，也就是说，正常来说，大部分的Batch在20ms内都会凑满，但是你的linger.ms可以保证，哪怕遇到低峰时期，20ms凑不满一个Batch，还是会在25ms之后强制Batch发送出去。

如果要是你把linger.ms设置的太小了，比如说默认就是0ms，或者你设置个5ms，那可能导致你的Batch虽然设置了32KB，但是经常是还没凑够32KB的数据，5ms之后就直接强制Batch发送出去，这样也不太好其实，会导致你的Batch形同虚设，一直凑不满数据。

#### 6、最大请求大小

**“max.request.size”这个参数决定了每次发送给Kafka服务器请求的最大大小**，同时也会限制你一条消息的最大大小也不能超过这个参数设置的值，这个其实可以根据你自己的消息的大小来灵活的调整。

给大家举个例子，你们公司发送的消息都是那种大的报文消息，每条消息都是很多的数据，一条消息可能都要20KB。

此时你的batch.size是不是就需要调节大一些？比如设置个512KB？然后你的buffer.memory是不是要给的大一些？比如设置个128MB？

只有这样，才能让你在大消息的场景下，还能使用Batch打包多条消息的机制。但是此时“max.request.size”是不是也得同步增加？

因为可能你的一个请求是很大的，默认他是1MB，你是不是可以适当调大一些，比如调节到5MB？

#### 7、重试机制

**“retries”和“retries.backoff.ms”决定了重试机制，也就是如果一个请求失败了可以重试几次，每次重试的间隔是多少毫秒**。

这个大家适当设置几次重试的机会，给一定的重试间隔即可，比如给100ms的重试间隔。

#### 8、持久化机制

“acks”参数决定了发送出去的消息要采用什么样的持久化策略，这个涉及到了很多其他的概念，大家可以参考之前专门为“acks”写过的一篇文章。

**参考**：[](https://mp.weixin.qq.com/s/YLrGg-jx5ddmHECmdccppw)

## 三、消息中间件消费到的消息处理失败怎么办？

消息中间件最核心的作用是：解耦、异步、削峰。

假如有如下的系统：

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/8.jpg>)

生产中存在这种情况：如果独立仓库系统或者第三方物流系统故障了，导致仓储系统消费到一条订单消息之后，尝试进行发货失败，也就是对这条消费到的消息处理失败。这种情况，怎么处理？

#### 死信队列的使用：处理失败的消息

一般生产环境中，如果你有丰富的架构设计经验，都会在使用MQ的时候设计两个队列：一个是**核心业务队列**，一个是**死信队列**。

核心业务队列，就是比如上面专门用来让订单系统发送订单消息的，然后另外一个死信队列就是用来处理异常情况的。

面试被问到这个问题时，必须要结合你自己的业务实践经验来说。

比如说要是第三方物流系统故障了，此时无法请求，那么仓储系统每次消费到一条订单消息，尝试通知发货和配送，都会遇到对方的接口报错。

此时仓储系统就可以把这条消息拒绝访问，或者标志位处理失败！**注意，这个步骤很重要。**

一旦标志这条消息处理失败了之后，MQ就会把这条消息转入提前设置好的一个死信队列中。

然后你会看到的就是，在第三方物流系统故障期间，所有订单消息全部处理失败，全部会转入死信队列。

然后你的仓储系统得专门有一个后台线程，监控第三方物流系统是否正常，能否请求的，不停的监视。

一旦发现对方恢复正常，这个后台线程就从死信队列消费出来处理失败的订单，重新执行发货和配送的通知逻辑。

**死信队列的使用，其实就是MQ在生产实践中非常重要的一环，也就是架构设计必须要考虑的**。

整个过程，如下图所示：

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/9.jpg>)

## 四、Kafka选举

Kafka中的选举大致可以分为三大类：

- 控制器的选举
- 分区leader的选举
- 消费者相关的选举

#### 1、控制器选举

在Kafka集群中会有一个或多个broker，其中有一个broker会被选举为控制器（Kafka Controller），它负责管理整个集群中所有分区和副本的状态等工作。

比如**当某个分区的leader副本出现故障时，由控制器负责为该分区选举新的leader副本**。再比如当检测到某个分区的ISR集合发生变化时，由控制器负责通知所有broker更新其元数据信息。

Kafka Controller的选举是依赖Zookeeper来实现的，在Kafka集群中那个broker能够成功创建/controller这个临时（Ephemeral）节点他就可以成为Kafka Controller。

这里需要说明一下的是Kafka Controller的实现还是相当复杂的，涉及到各个方面的内容，如果你掌握了Kafka Controller，你就掌握了Kafka的“半壁江山”。

#### 2、分区leader的选举

分区leader副本的选举**由Kafka Controller 负责具体实施**。

当创建分区（创建主题或增加分区都有创建分区的动作）或分区上线（比如分区中原先的leader副本下线，此时分区需要选举一个新的leader上线来对外提供服务）的时候都需要执行leader的选举动作。

基本思路是按照AR集合中副本的顺序查找第一个存活的副本，并且这个副本在ISR集合中。

一个分区的AR集合在分配的时候就被指定，并且只要不发生重分配的情况，集合内部副本的顺序是保持不变的，而分区的ISR集合中副本的顺序可能会改变。

注意：这里是根据AR的顺序而不是ISR的顺序进行选举的。这个说起来比较抽象，有兴趣的读者可以手动关闭/开启某个集群中的broker来观察一下具体的变化。

还有一些情况也会发生分区leader的选举，比如当分区进行重分配（reassign）的时候也需要执行leader的选举动作。

这个思路比较简单：从重分配的AR列表中找到第一个存活的副本，且这个副本在目前的ISR列表中。

再比如当发生优先副本（preferred replica partition leader election）的选举时，直接将优先副本设置为leader即可，AR集合中的第一个副本即为优先副本。

还有一种情况就是当某节点被优雅地关闭（也就是执行ControlledShutdown）时，位于这个节点上的leader副本都会下线，所以与此对应的分区需要执行leader的选举。

这里的具体思路为：从AR列表中找到第一个存活的副本，且这个副本在目前的ISR列表中，与此同时还要确保这个副本不处于正在被关闭的节点上。

#### 3、消费者相关的选择

组协调器GroupCoordinator需要为消费组内的消费者选举出一个消费组的leader，这个选举的算法也很简单，分两种情况分析。

- **如果消费组内还没有leader，那么第一个加入消费组的消费者即为消费组的leader**。

- **如果某一时刻leader消费者由于某些原因退出了消费组，那么会重新选举一个新的leader，这个重新选举leader的过程又更“随意”了，相关代码如下**：

```scala
//scala code.
private val members = new mutable.HashMap[String, MemberMetadata]
var leaderId = members.keys.head
```

解释一下这2行代码：在GroupCoordinator中消费者的信息是以HashMap的形式存储的，其中key为消费者的member_id，而value是消费者相关的元数据信息。

leaderId表示leader消费者的member_id，它的取值为HashMap中的第一个键值对的key，这种选举的方式基本上和随机无异。

总体上来说，消费组的leader选举过程是很随意的。

到这里就结束了吗？还有分区分配策略的选举呢。

或许你对此有点陌生，但是用过Kafka的同学或许对partition.assignment.strategy（取值为RangeAssignor、RoundRobinAssignor、StickyAssignor等）这个参数并不陌生。

每个消费者都可以设置自己的分区分配策略，对消费组而言需要从各个消费者呈报上来的各个分配策略中选举一个彼此都“信服”的策略来进行整体上的分区分配。

这个分区分配的选举并非由leader消费者决定，而是根据消费组内的各个消费者投票来决定的。

**参考**：[](https://mp.weixin.qq.com/s/XvDpq1xxXPzRoRKMO-MxeQ)

## 五、如何保证消息不被重复消费？（如何保证消息消费的幂等性）

### 面试官心理分析

其实这是很常见的一个问题，这俩问题基本可以连起来问。既然是消费消息，那肯定要考虑会不会重复消费？能不能避免重复消费？或者重复消费了也别造成系统异常可以吗？这个是 MQ 领域的基本问题，其实本质上还是问你**使用消息队列如何保证幂等性**，这个是你架构里要考虑的一个问题。

### 面试题剖析

回答这个问题，首先大概说一说可能会有哪些重复消费的问题。

首先，比如 RabbitMQ、RocketMQ、Kafka，都有可能会出现消息重复消费的问题，挑 Kafka 来举个例子，说说怎么重复消费吧。

Kafka 实际上有个 offset 的概念，就是每个消息写进去，都有一个 offset，代表消息的序号，然后 consumer 消费了数据之后，**每隔一段时间**（**定时定期**），会把自己消费过的消息的 offset 提交一下，表示“我已经消费过了，下次我要是重启啥的，你就让我继续从上次消费到的 offset 来继续消费吧”。

但是，你有时候重启系统，看你怎么重启了，如果碰到点着急的，直接 kill 进程了，再重启。这会导致 consumer 有些消息处理了，但是**没来得及提交 offset，重启之后，少数消息会再次消费一次**。

例如，数据 1/2/3 依次进入 kafka，kafka 会给这三条数据每条分配一个 offset，代表这条数据的序号，我们就假设分配的 offset 依次是 152/153/154。消费者从 kafka 去消费的时候，也是按照这个顺序去消费。假如当消费者消费了 `offset=153` 的这条数据，刚准备去提交 offset 到 zookeeper，此时消费者进程被重启了。那么此时消费过的数据 1/2 的 offset 并没有提交，kafka 也就不知道你已经消费了 `offset=153` 这条数据。那么重启之后，消费者会找 kafka 说，嘿，哥儿们，你给我接着把上次我消费到的那个地方后面的数据继续给我传递过来。由于之前的 offset 没有提交成功，那么数据 1/2 会再次传过来，如果此时消费者没有去重的话，那么就会导致重复消费。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/10.png>)

**如何保证消息队列消费的幂等性**？

回答这个问题需要结合业务思考，有如下几个思路：

- 比如数据要写库，先根据主键查一下，如果这数据都有了，就别插入了，update 一下。
- 比如是写 Redis，那没问题了，因为每次都是 set，天然幂等性。
- 比如不是上面两个场景，那做的稍微复杂一点，你需要让生产者发送每条数据的时候，里面**加一个全局唯一的 id**，类似订单 id 之类的东西，然后你这里消费到了之后，先根据这个 id 去比如 Redis 里查一下，之前消费过吗？如果没有消费过，你就处理，然后这个 id 写 Redis。如果消费过了，就别处理，保证别重复处理相同的消息即可。
- 比如基于数据库的唯一键来保证重复数据不会重复插入多条。因为有唯一键约束了，重复数据插入只会报错，不会导致数据库中出现脏数据。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/11.png>)

## 六、如何保证消息的可靠性传输？（如何处理消息丢失的问题？）

## 面试官心理分析

这个是肯定的，用 MQ 有个基本原则，就是**数据不能多一条，也不能少一条**，不能多，就是前面说的[重复消费和幂等性问题。不能少，就是说这数据别搞丢了。那这个问题你必须得考虑一下。

如果说你这个是用 MQ 来传递非常核心的消息，比如说计费、扣费的一些消息，那必须确保这个 MQ 传递过程中**绝对不会把计费消息给弄丢**。

## 面试题剖析

数据的丢失问题，可能出现在**生产者、MQ、消费者**中，从 Kafka 来分析一下。

### Kafka

### 1、消费者丢失数据

唯一可能导致消费者弄丢数据的情况，是消费到了这个消息，然后消费者那边**自动提交了 offset**，让 Kafka 以为你已经消费好了这个消息，但其实你才刚准备处理这个消息，你还没处理，你自己就挂了，此时这条消息就丢咯。

由于 Kafka 会自动提交 offset，那么只要**关闭自动提交** offset，在处理完之后自己手动提交 offset，就可以保证数据不会丢。但是此时确实还是**可能会有重复消费**，比如你刚处理完，还没提交 offset，结果自己挂了，此时肯定会重复消费一次，自己保证幂等性就好了。

生产环境碰到的一个问题是Kafka 消费者消费到了数据之后是写到一个内存的 queue 里先缓冲一下，结果有的时候，你刚把消息写入内存 queue，然后消费者会自动提交 offset。然后此时我们重启了系统，就会导致内存 queue 里还没来得及处理的数据就丢失了。

### 2、Kafka弄丢数据

这块比较常见的一个场景，就是 Kafka 某个 broker 宕机，然后重新选举 partition 的 leader。如果此时其他的 follower 刚好还有些数据没有同步，结果此时 leader 挂了，然后选举某个 follower 成 leader 之后，不就少了一些数据？这就丢了一些数据啊。

所以此时一般是要求起码设置如下 4 个参数：

- 给 topic 设置 `replication.factor` 参数：这个值必须大于 1，要求每个 partition 必须有至少 2 个副本。
- 在 Kafka 服务端设置 `min.insync.replicas` 参数：这个值必须大于 1，这个是要求一个 leader 至少感知到有至少一个 follower 还跟自己保持联系，没掉队，这样才能确保 leader 挂了还有一个 follower 吧。
- 在 producer 端设置 `acks=all`：这个是要求每条数据，必须是**写入所有 replica 之后，才能认为是写成功了**。
- 在 producer 端设置 `retries=MAX`（很大很大很大的一个值，无限次重试的意思）：这个是**要求一旦写入失败，就无限重试**，卡在这里了。

我们生产环境就是按照上述要求配置的，这样配置之后，至少在 Kafka broker 端就可以保证在 leader 所在 broker 发生故障，进行 leader 切换时，数据不会丢失。

### 3、生产者会不会弄丢数据？

如果按照上述的思路设置了 `acks=all`，一定不会丢，要求是，你的 leader 接收到消息，所有的 follower 都同步到了消息之后，才认为本次写成功了。如果没满足这个条件，生产者会自动不断的重试，重试无限次。

## 七、如何保证消息的顺序性？

Kafka：比如说我们建了一个 topic，有三个 partition。生产者在写的时候，其实可以指定一个 key，比如说我们指定了某个订单 id 作为 key，那么这个订单相关的数据，一定会被分发到同一个 partition 中去，而且这个 partition 中的数据一定是有顺序的。
消费者从 partition 中取出来数据的时候，也一定是有顺序的。到这里，顺序还是 ok 的，没有错乱。接着，我们在消费者里可能会搞**多个线程来并发处理消息**。因为如果消费者是单线程消费处理，而处理比较耗时的话，比如处理一条消息耗时几十 ms，那么 1 秒钟只能处理几十条消息，这吞吐量太低了。而多个线程并发跑的话，顺序可能就乱掉了。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/12.png>)

#### 解决方案：

- 一个 topic，一个 partition，一个 consumer，内部单线程消费，单线程吞吐量太低，一般不会用这个。
- 写 N 个**内存 queue**，具有相同 key 的数据都到同一个内存 queue；然后对于 N 个线程，每个线程分别消费一个内存 queue 即可，这样就能保证顺序性。

![](<https://github.com/XU-ZHOU/Java/blob/master/pictures/13.png>)


