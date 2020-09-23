# kafka安装部署

kafka 集群依赖于zookeper集群，所以需要先安装zookeper



## 0、安装并启动zookeper

1. 安装zk

2. 配置zoo.cfg文件

```properties
dataDir=/opt/modules/zookeeper-3.4.5/zkdata
server.1=bigdata3.ibeifeng.com:2888:3888
server.2=bigdata4.ibeifeng.com:2888:3888
server.3=bigdata5.ibeifeng.com:2888:3888
```

3. 创建zkdata目录，在zkdata目录下创建myid文件，编辑myid文件内容，就是此台server的zk的id号

4. 启动三台zkServer

```shell
$ bin/zkServer.sh start			##启动服务
$ bin/zkServer.sh status		##查看状态
```



## 1、下载kafka安装包

```shell
curl -O http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.3.0/kafka_2.12-2.3.0.tgz

# 解压kafka压缩包到 /opt/modules 目录下
tar -zxf kafka_2.12-2.3.0.tgz -C /opt/modules/
```



## 2、修改配置文件

```shell
cd /opt/modules/kafka_2.12-2.3.0/config/

#进入配置文件编辑模式
vim server.properties
```

```properties
listeners=PLAINTEXT://:9092

#bd-server1.mml.com 是主机的地址
advertised.listeners=PLAINTEXT://bd-server1.mml.com:9092

#设置kafka的日志输出目录
log.dirs=/opt/kafka/kafka_2.12-2.3.0/logs	

# zookeeper 的地址
zookeeper.connect=localhost:2181
```



## 3、启动kafka

```shell
#命令启动。非后台模式，启动后可以查看日志，退出既关闭kafka
bin/kafka-server-start.sh config/server.properties

#后台启动模式，启动后日志输出至配置文件中设置的位置中
bin/kafka-server-start.sh -daemon config/server.properties
```



## 4、生产和消费消息

```shell
# 创建test topic
kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test

# 获取所有topic
kafka-topics.sh --list --bootstrap-server localhost:9092

# 生产消息（进入交互模式输入消息内容，Ctrl + C 退出）
kafka-console-producer.sh --broker-list localhost:9092 --topic test

# 消费消息（进入交互模式获取消息内容，Ctrl + C 退出）
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```





# kafak 学习



## 入门

#### 简介

Kafka  分布式、发布/订阅、消息系统。

#### 作用

1. 解耦：将服务功能分解

2. 异步：减少请求耗时

3. 削峰

#### 特性

1. 容错性：topic和多个副本分区

2. 可靠性

消息：基本数据单元，由key、value组成

key负责消息路由，根据一定策略决策消息路由的分区，保证同一个key路由到同一个分区

topic：用于存储消息的逻辑概念，消息的集合



## kafka 架构与设计



### kafka是写磁盘还是写内存

磁盘。不同于Redis的内存 + 持久化磁盘，也不同于Mongodb仅把索引放于内存。kafka是基于磁盘的持久化。



### kafka究竟是由 consumer 从 broker 那里 pull 数据，还是由 broker 将数据 push 到 consumer？

Kafka 在这方面采取了一种较为传统的设计方式，也是大多数的消息系统所共享的方式：即 producer 把数据 push 到 broker，然后 consumer 从 broker 中 pull 数据。



pull-based 系统有一个很好的特性， 那就是当 consumer 速率落后于 producer 时，可以在适当的时间赶上来。还可以通过使用某种 backoff 协议来减少这种现象：即 consumer 可以通过 backoff 表示它已经不堪重负了，然而通过获得负载情况来充分使用 consumer（但永远不超载）这一方式实现起来比它看起来更棘手。前面以这种方式构建系统的尝试，引导着 Kafka 走向了更传统的 pull 模型。



另一个 pull-based 系统的优点在于：它可以大批量生产要发送给 consumer 的数据。而 push-based 系统必须选择立即发送请求或者积累更多的数据，然后在不知道下游的 consumer 能否立即处理它的情况下发送这些数据。如果系统调整为低延迟状态，这就会导致一次只发送一条消息，以至于传输的数据不再被缓冲，这种方式是极度浪费的。 而 pull-based 的设计修复了该问题，因为 consumer 总是将所有可用的（或者达到配置的最大长度）消息 pull 到 log 当前位置的后面，从而使得数据能够得到最佳的处理而不会引入不必要的延迟。



简单的 pull-based 系统的不足之处在于：如果 broker 中没有数据，consumer 可能会在一个紧密的循环中结束轮询，实际上 busy-waiting 直到数据到来。为了避免 busy-waiting，我们在 pull 请求中加入参数，使得 consumer 在一个“long pull”中阻塞等待，直到数据到来（还可以选择等待给定字节长度的数据来确保传输长度）。



### 消费者如何跟踪已消费内容的位置？或如何区分已消费（consumed）的记录？

为了解决消息丢失的问题，许多消息系统增加了确认机制：即当消息被发送出去的时候，消息仅被标记为sent 而不是 consumed；然后 broker 会等待一个来自 consumer 的特定确认，再将消息标记为consumed。这个策略修复了消息丢失的问题，但也产生了新问题。 首先，如果 consumer 处理了消息但在发送确认之前出错了，那么该消息就会被消费两次。第二个是关于性能的，现在 broker 必须为每条消息保存多个状态（首先对其加锁，确保该消息只被发送一次，然后将其永久的标记为 consumed，以便将其移除）。 还有更棘手的问题要处理，比如如何处理已经发送但一直得不到确认的消息。

Kafka的做法非常高效。Kafka的 topic 被分割成了一组完全有序的 partition，其中每一个 partition 在任意给定的时间内只能被每个订阅了这个 topic 的 consumer 组中的一个 consumer 消费。**这意味着 partition 中 每一个 consumer 的位置仅仅是一个数字，即下一条要消费的消息的offset。**这使得被消费的消息的状态信息相当少，每个 partition 只需要一个数字。这个状态信息还可以作为周期性的 checkpoint。这以非常低的代价实现了和消息确认机制等同的效果。

这种方式还有一个附加的好处。consumer 可以回退到之前的 offset 来再次消费之前的数据，这个操作违反了队列的基本原则，但事实证明对大多数 consumer 来说这是一个必不可少的特性。 例如，如果 consumer 的代码有 bug，并且在 bug 被发现前已经有一部分数据被消费了， 那么 consumer 可以在 bug 修复后通过回退到之前的 offset 来再次消费这些数据



### kafka用什么方法保障持久化的低延迟和高效率？

* 线性写磁盘文件

* 预读取

* 预写、批量写

* zero-copy（零拷贝）优化

* 端到端的批量压缩



#### 线性写磁盘

通常认为磁盘很慢，那是随机写慢，随机写的速度只有100K/秒，而几年前的6个7200rpm SATA RAID-5 的磁盘阵列上线性写的速度大概是600M/秒。相差6000倍。一个好的磁盘结构设计（线性写）可以使之跟网络速度一样快。



#### 预读取

预读取将有效地将每个磁盘中有用的数据预存到缓存。现代的操作系统提供了预读和写技术，将多个大块预取数据。



#### 预写

较小的写入组合成一个大的物理写。

对于主要用于日志处理的消息系统，数据的持久化可以简单的通过将数据追加到文件中实现，读的时候从文件中读就好了。

这样做的好处是读和写都是 ***O(1)***  的，并且读操作不会阻塞写操作和其他操作。这样带来的性能优势是很明显的，因为性能和数据的大小没有关系了。这意味着我们可以提供一般消息传递系统无法提供的特性。 例如，在Kafka中，消息被消费后不是立马被删除，我们可以保留消息相对较长的时间（默认设置为一个星期）。 这将为消费者带来很大的灵活性。

一旦消除了磁盘访问模式不佳的情况，该类系统性能低下的主要原因就剩下了两个：**大量的小型 I/O 操作，以及过多的字节拷贝**。

为了避免这种情况，我们的协议是建立在一个 “消息块” 的抽象基础上，合理将消息分组。 这使得网络请求将多个消息打包成一组，而不是每次发送一条消息，从而使整组消息分担网络中往返的开销。Consumer 每次获取多个大型有序的消息块，并由服务端 依次将消息块一次加载到它的日志中。这个简单的优化对速度有着数量级的提升。批处理允许更大的网络数据包，更大的顺序读写磁盘操作，连续的内存块等等，所有这些都使 KafKa 将随机流消息顺序写入到磁盘， 再由 consumers 进行消费。批处理是提升性能的一个主要驱动，为了允许批量处理，kafka 生产者会尝试在内存中汇总数据，并用一次请求批次提交信息。 批处理，不仅仅可以配置指定的消息数量，也可以指定等待特定的延迟时间(如64k 或10ms)，这允许汇总更多的数据后再发送，在服务器端也会减少更多的IO操作。 该缓冲是可配置的，并给出了一个机制，通过权衡少量额外的延迟时间获取更好的吞吐量。

```properties
# 两者有一个满足，就会将消息发送给kafka
kafka.producer.batchSize=64		#生产者消息块大小
kafka.producer.lingerMS=10		#生产者发送间隔时间
```



#### zero-copy（零拷贝）优化

broker 维护的消息日志本身就是一个文件目录，每个文件都由一系列以相同格式写入到磁盘的消息集合组成，这种写入格式被 producer 和 consumer 共用。保持这种通用格式可以对一些很重要的操作进行优化: 持久化日志块的网络传输。 现代的unix 操作系统提供了一个高度优化的编码方式，用于将数据从 pagecache 转移到 socket 网络连接中；在 Linux 中系统调用 sendfile 做到这一点。



为了理解 sendfile 的意义，了解数据从文件到套接字的常见数据传输路径就非常重要（四次 copy）：



1. 操作系统从磁盘读取数据到内核空间的 pagecache

2. 应用程序读取内核空间的数据到用户空间的缓冲区

3. 应用程序将数据(用户空间的缓冲区)写回内核空间到套接字缓冲区(内核空间)

4. 操作系统将数据从套接字缓冲区(内核空间)复制到通过网络发送的 NIC 缓冲区



![zerocopy1](https://img2018.cnblogs.com/blog/41852/201912/41852-20191205181223572-1081370570.gif "zerocopy1")



这显然是低效的，有四次 copy 操作和两次系统调用。使用 sendfile 方法，可以允许操作系统**将数据从 pagecache 直接发送到网络**，这样避免重新复制数据。所以这种优化方式，只需要最后一步的copy操作，将数据复制到 NIC 缓冲区。	

我们期望一个普遍的应用场景，一个 topic 被多消费者消费。使用上面提交的 zero-copy（零拷贝）优化，数据在使用时只会被复制到 pagecache 中一次，节省了每次拷贝到用户空间内存中，再从用户空间进行读取的消耗。这使得消息能够以接近网络连接速度的 上限进行消费。



![zerocopy2](https://img2018.cnblogs.com/blog/41852/201912/41852-20191205181237201-912732940.gif "zerocopy2")



pagecache 和 sendfile 的组合使用意味着，在一个kafka集群中，大多数 consumer 消费时，您将看不到磁盘上的读取活动，因为数据将完全由缓存提供。



#### 端到端的批量压缩



在某些情况下，数据传输的瓶颈不是 CPU ，也不是磁盘，而是网络带宽。对于需要通过广域网在数据中心之间发送消息的数据管道尤其如此。当然，用户可以在不需要 Kakfa 支持下一次一个的压缩消息。但是这样会造成非常差的压缩比和消息重复类型的冗余，比如 JSON 中的字段名称或者是或 Web 日志中的用户代理或公共字符串值。高性能的压缩是一次压缩多个消息，而不是压缩单个消息。



Kafka 以高效的批处理格式支持一批消息可以压缩在一起发送到服务器。这批消息将以压缩格式写入，并且在日志中保持压缩，只会在 consumer 消费时解压缩。



Kafka 支持 GZIP，Snappy 和 LZ4 压缩协议。



### kafka的消息保证有几种方式？请描述细节。

Kafka可以提供的消息交付语义保证有多种：

* At most once最多一次—— 消息可能会丢失但绝不重传。

* At least once最少一次—— 消息可以重传但绝不丢失。

* Exactly once只一次—— 每一条消息只被传递一次.



#### 生产端的at-least-once

在 0.11.0.0 之前的版本中, 如果 producer 没有收到表明消息已经被提交的响应, 那么 producer 除了将消息重传之外别无选择。 这里提供的是 at-least-once 的消息交付语义，因为如果最初的请求事实上执行成功了，那么重传过程中该消息就会被再次写入到 log 当中。



#### 生产端的exactly-once

从 0.11.0.0 版本开始，Kafka producer新增了幂等性的传递选项，该选项保证重传不会在 log 中产生重复条目。 为实现这个目的, broker 给每个 producer 都分配了一个 ID ，并且 producer 给每条被发送的消息分配了一个序列号来避免产生重复的消息。 同样也是从 0.11.0.0 版本开始, producer 新增了使用类似事务性的语义将消息发送到多个 topic



partition 的功能： 也就是说，要么所有的消息都被成功的写入到了 log，要么一个都没写进去。这种语义的主要应用场景就是 Kafka topic 之间的 exactly-once 的数据传递(如下所述)。



#### 消费端的at-most-once

Consumer 可以先读取消息，然后将它的位置保存到 log 中，最后再对消息进行处理。在这种情况下，消费者进程可能会在保存其位置之后，带还没有保存消息处理的输出之前发生崩溃。而在这种情况下，即使在此位置之前的一些消息没有被处理，接管处理的进程将从保存的位置开始。在 consumer 发生故障的情况下，这对应于“at-most-once”的语义，可能会有消息得不到处理。



#### 消费端的at-least-once

Consumer 可以先读取消息，然后处理消息，最后再保存它的位置。在这种情况下，消费者进程可能会在处理了消息之后，但还没有保存位置之前发生崩溃。而在这种情况下，当新的进程接管后，它最初收到的一部分消息都已经被处理过了。在 consumer 发生故障的情况下，这对应于“at-least-once”的语义。 在许多应用场景中，消息都设有一个主键，所以更新操作是幂等的（相同的消息接收两次时，第二次写入会覆盖掉第一次写入的记录）。



#### 消费端的exactly-once

使用我们上文提到的 0.11.0.0 版本中的新事务型 producer，达成消息存储的exactly-once。将 consumer 的offset存储为一个 topic 中的消息，所以我们可以在输出 topic 接收信号（数据已经被处理的)时候，在同一个事务中向 Kafka 写入 offset。如果事务被中断，则消费者的位置将恢复到原来的值，而输出 topic 上产生的数据对其他消费者是否可见，取决于事务的“隔离级别”。 在默认的“read_uncommitted”隔离级别中，所有消息对 consumer 都是可见的，即使它们是中止的事务的一部分，但是在“read_committed”的隔离级别中，消费者只能访问已提交的事务中的消息（以及任何不属于事务的消息）。举一个非常实际的例子，HR系统将组织架构的变更生产到Kafka，同步程序来完成其他业务系统（比如OA、CRM等）来进行exactly-once的同步变更，需要在 consumer 的 offset 与实际存储为输出的内容间进行协调，比方消费的ID号批次为10095-10099，则offset截止到10099（上一次的offset为10094），实际同步程序也在处理数据同步，保持exactly-once的一般做法为two-phase commit，但有些系统不支持two-phase commit，一般的做法是使consumer和输出系统 将其 offset 存储在与其输出相同的位置，在这个同步组织架构的例子就是同步程序处理完了，可以将offsize保存在db、cache、文件系统、HDFS之类的介质中，如果同步失败（比如10098和10099失败了），则把offsize保存为10097，这样可以下次再次（实际上为exactly-once）处理10098和10099两条记录。



最后说一下，Kafka 默认保证 at-least-once 的消息交付，这样可以避免exactly-once带来的业务复杂性，至于多次传的重复数据处理，完全可以通过更新操作幂等来去除重复（即交给业务系统来去重）。



资料来源：

> https://www.cnblogs.com/starcrm/p/11990972.html