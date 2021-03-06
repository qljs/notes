## 一 基本概念和特性

### 1. 基本概念

- **消息模型（Message Model）：**RocketMQ主要由 Producer、Broker、Consumer 三部分组成，其中Producer 负责生产消息，Consumer 负责消费消息，Broker 负责存储消息。Broker 在实际部署过程中对应一台服务器，每个 Broker 可以存储多个Topic的消息，每个Topic的消息也可以分片存储于不同的 Broker。**Message Queue 用于存储消息的物理地址**，每个Topic中的消息地址存储于多个 Message Queue 中。ConsumerGroup 由多个 Consumer 实例构成。
- **消息生产者（Producer）：**负责生产消息，一般由业务系统负责生产消息。RocketMQ 提供多种发送方式，同步发送、异步发送、顺序发送、单向发送。同步和异步方式均需要Broker返回确认信息，单向发送不需要。
- **消息消费者（consumer）：**负责消费消息，一般是后台系统负责异步消费。从用户应用的角度而言提供了两种消费形式：拉取式消费、推动式消费。
- **主题（Topic）：**表示一类消息的集合，每个主题包含若干条消息，每条消息只能属于一个主题，是RocketMQ进行消息订阅的基本单位。
- **代理服务器（Broker Server）：**消息中转角色，负责存储消息、转发消息。代理服务器在 RocketMQ 系统中负责接收从生产者发送来的消息并存储、同时为消费者的拉取请求作准备。代理服务器也存储消息相关的元数据，包括消费者组、消费进度偏移和主题和队列消息等。
- **名字服务器（Name Server）：**名称服务充当路由消息的提供者。生产者或消费者能够通过名字服务查找各主题相应的Broker IP列表。多个Namesrv实例组成集群，但相互独立，没有信息交换。
- **拉取式消费（Pull Consumer）：**Consumer消费的一种类型，应用通常主动调用Consumer的拉消息方法从Broker服务器拉消息、主动权由应用控制。一旦获取了批量消息，应用就会启动消费过程。
- **推动式消费（Push Consumer）：**Consumer消费的一种类型，该模式下Broker收到数据后会主动推送给消费端，该消费模式一般实时性较高。
- **生产者组（Producer Group）：**同一类Producer的集合，这类Producer发送同一类消息且发送逻辑一致。如果发送的是事务消息且原始生产者在发送之后崩溃，则Broker服务器会联系同一生产者组的其他生产者实例以提交或回溯消费。
- **消费者组（Consumer Group）：**同一类Consumer的集合，这类Consumer通常消费同一类消息且消费逻辑一致。消费者组使得在消息消费方面，实现负载均衡和容错的目标变得非常容易。要注意的是，消费者组的消费者实例必须订阅完全相同的Topic。RocketMQ 支持两种消息模式：集群消费（Clustering）和广播消费（Broadcasting）。
- **集群消费（Clustering）：**集群消费模式下,相同Consumer Group的每个Consumer实例平均分摊消息。
- **广播消费（Broadcasting）：**广播消费模式下，相同Consumer Group的每个Consumer实例都接收全量的消息。
- **普通顺序消息（Normal Ordered Message）：**普通顺序消费模式下，消费者通过同一个消费队列收到的消息是有顺序的，不同消息队列收到的消息则可能是无顺序的。
- **严格顺序消息（Strictly Ordered Message）：**严格顺序消息模式下，消费者收到的所有消息均是有顺序的。
- **标签（Tag）：**为消息设置的标志，用于同一主题下区分不同类型的消息。来自同一业务单元的消息，可以根据不同业务目的在同一主题下设置不同标签。标签能够有效地保持代码的清晰度和连贯性，并优化RocketMQ提供的查询系统。消费者可以根据Tag实现对不同子主题的不同消费逻辑，实现更好的扩展性。
- **消息（Message）：**消息系统所传输信息的物理载体，生产和消费数据的最小单位，每条消息必须属于一个主题。RocketMQ中每个消息拥有唯一的Message ID，且可以携带具有业务标识的Key。系统提供了通过Message ID和Key查询消息的功能。



### 2. 特性

#### 2.1 订阅和发布

发布指某个生产者想某个 topic 发送消息；订阅指某消费者关注消费某个 topic 中带有有些 tag 的消息；



#### 2.2 消息顺序

指消息消费时按照发送的顺序进行消费。

- 全局顺序：对于指定的 topic，所有消息按照 FIFO 的顺序进行发布和消费。使用场景：性能要求不高，所有消息按照 FIFO 原则进行发布和消费；
- 分区顺序：对于指定的 topic，所有消息根据 sharding key 进行分区。同一区块中中的消息按照 FIFO 的顺序进行发布和消费， sharding key 与普通的 key 不同，它用来区分分区；用于性能要求较高的场景。



#### 2.3 消息过滤

RocketMQ 支持根据 Tag 或者自定义属性进行过滤，消息过滤在 Broker 端实现，优点是减少了消费者无用的网络传输，缺点是增加了 Broker 的负担。



#### 2.4 消息可靠性

RocketMQ支持消息的高可靠，影响消息可靠性的几种情况：

1. Broker非正常关闭
2. Broker异常Crash
3. OS Crash
4. 机器掉电，但是能立即恢复供电情况
5. 机器无法开机（可能是cpu、主板、内存等关键设备损坏）
6. 磁盘设备损坏

1)、2)、3)、4) 四种情况都属于硬件资源可立即恢复情况，RocketMQ在这四种情况下能保证消息不丢，或者丢失少量数据（依赖刷盘方式是同步还是异步）。

5)、6)属于单点故障，且无法恢复，一旦发生，在此单点上的消息全部丢失。RocketMQ在这两种情况下，通过异步复制，可保证99%的消息不丢，但是仍然会有极少量的消息可能丢失。通过同步双写技术可以完全避免单点，同步双写势必会影响性能，适合对消息可靠性要求极高的场合，例如与Money相关的应用。注：RocketMQ从3.0版本开始支持同步双写。



#### 2.5 回溯消费

回溯消费是指消费者已经成功消费，但是业务上需要重新消费，即在 Broker 想消费者投递成功后，消息仍要保留。

重新消费一般是按照时间维度，RocketMQ支持按照时间维度回溯消费，精确到毫秒。



#### 2.6 事务消息

在 RocketMQ 中，支持将本地事务和发送消息放到全局事务中，通过事务消息达到分布式事务最终一致性。



#### 2.7  定时消息

定时消息（延迟队列）指消息被发送到 broker 后，在等待指定时间后投递到 topic 上。broker 有配置项`messageDelayLevel`，该属性属于 broker，

定时消息会暂存在名为SCHEDULE_TOPIC_XXXX的topic中，并根据delayTimeLevel存入特定的queue，queueId = delayTimeLevel – 1，即一个queue只存相同延迟的消息，保证具有相同发送延迟的消息能够顺序消费。broker会调度地消费SCHEDULE_TOPIC_XXXX，将消息写入真实的topic。

需要注意的是，定时消息会在第一次写入和调度写入真实topic时都会计数，因此发送数量、tps都会变高。



#### 2.8 消息重试

消费者消息失败后，消息重试机制可以让消费者重新消费一遍。Consumer消费消息失败通常可以认为有以下几种情况：

- 由于消息本身的原因，例如反序列化失败，消息数据本身无法处理（例如话费充值，当前消息的手机号被注销，无法充值）等。这种错误通常需要跳过这条消息，再消费其它消息，而这条失败的消息即使立刻重试消费，99%也不成功，所以最好提供一种定时重试机制，即过10秒后再重试。
- 由于依赖的下游应用服务不可用，例如db连接不可用，外系统网络不可达等。遇到这种错误，即使跳过当前失败的消息，消费其他消息同样也会报错。这种情况建议应用sleep 30s，再消费下一条消息，这样可以减轻Broker重试消息的压力。

RocketMQ会为每个消费组都设置一个Topic名称为“%RETRY%+consumerGroup”的重试队列（这里需要注意的是，这个Topic的重试队列是针对消费组，而不是针对每个Topic设置的），用于暂时保存因为各种异常而导致Consumer端无法消费的消息。考虑到异常恢复起来需要一些时间，会为重试队列设置多个重试级别，每个重试级别都有与之对应的重新投递延时，重试次数越多投递延时就越大。RocketMQ对于重试消息的处理是先保存至Topic名称为“SCHEDULE_TOPIC_XXXX”的延迟队列中，后台定时任务按照对应的时间进行Delay后重新保存至“%RETRY%+consumerGroup”的重试队列中。



#### 2.9 消息重投

生产者在发送消息时，同步消息失败会重投，异步消息有重试。消息重投保证消息尽可能发送成功，但是可能会造成消息重复消费。

消息重投策略：

- retryTimesWhenSendFailed:同步发送失败重投次数，默认为2，因此生产者会最多尝试发送retryTimesWhenSendFailed + 1次。不会选择上次失败的broker，尝试向其他broker发送，最大程度保证消息不丢。超过重投次数，抛出异常，由客户端保证消息不丢。当出现RemotingException、MQClientException和部分MQBrokerException时会重投。
- retryTimesWhenSendAsyncFailed:异步发送失败重试次数，异步重试不会选择其他broker，仅在同一个broker上做重试，不保证消息不丢。
- retryAnotherBrokerWhenNotStoreOK:消息刷盘（主或备）超时或slave不可用（返回状态非SEND_OK），是否尝试发送到其他broker，默认false。十分重要消息可以开启。



#### 2.10 流量控制

生产者流控，因为broker处理能力达到瓶颈；消费者流控，因为消费能力达到瓶颈。

生产者流控：

- commitLog文件被锁时间超过osPageCacheBusyTimeOutMills时，参数默认为1000ms，返回流控。
- 如果开启transientStorePoolEnable == true，且broker为异步刷盘的主机，且transientStorePool中资源不足，拒绝当前send请求，返回流控。
- broker每隔10ms检查send请求队列头部请求的等待时间，如果超过waitTimeMillsInSendQueue，默认200ms，拒绝当前send请求，返回流控。
- broker通过拒绝send 请求方式实现流量控制。

注意，生产者流控，不会尝试消息重投。

消费者流控：

- 消费者本地缓存消息数超过pullThresholdForQueue时，默认1000。
- 消费者本地缓存消息大小超过pullThresholdSizeForQueue时，默认100MB。
- 消费者本地缓存消息跨度超过consumeConcurrentlyMaxSpan时，默认2000。

消费者流控的结果是降低拉取频率。



#### 2.11 死信队列

死信队列用于处理无法被正常消费的消息。当一条消息初次消费失败，消息队列会自动进行消息重试；达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时，消息队列 不会立刻将消息丢弃，而是将其发送到该消费者对应的特殊队列中。

在RocketMQ中，可以通过使用console控制台对死信队列中的消息进行重发来使得消费者实例再次进行消费。	



## 二 架构设计

### 1. 架构

在 RocketMQ 中主要分为四部分：生产者、消费者、BrokerServer、NameServer。

- 生产者：发布消息的角色；通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递，投递的过程支持快速失败并且低延迟。
- 消费者：消费消息的角色；支持 push、pull 模式消费消息，也支持广播和订阅机制。
- NameServer：简单的 Topic 路由注册中心，相当于 zk，支持 broker 的动态注册和发现。主要包括两个功能：broker 管理，保存 broker 集群的全量注册信息，提供心跳检测；路由信息管理：保存关于Broker集群的整个路由信息和用于客户端查询的队列信息，消费者和生产者通过 NameServer 获取整个 Broker 集群的路由信息，用于消息投递和消费。
- BrokerServer：Broker主要负责消息的存储、投递和查询以及服务高可用保证，为了实现这些功能，Broker包含了以下几个重要子模块。
  - Remoting Module：整个Broker的实体，负责处理来自clients端的请求。
  - Client Manager：负责管理客户端(Producer/Consumer)和维护Consumer的Topic订阅信息。
  - Store Service：提供方便简单的API接口处理消息存储到物理硬盘和查询功能。
  - HA Service：高可用服务，提供Master Broker 和 Slave Broker之间的数据同步功能。
  - Index Service：根据特定的Message key对投递到Broker的消息进行索引服务，以提供消息的快速查询。

### RocketMQ 网络部署特点

- NameServer是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。
- Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave 的对应关系通过指定相同的BrokerName，不同的BrokerId 来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer。 注意：当前RocketMQ版本在部署架构上支持一Master多Slave，但只有BrokerId=1的从服务器才会参与消息的读负载。
- Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic 服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。
- Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，消费者在向Master拉取消息时，Master服务器会根据拉取偏移量与最大偏移量的距离（判断是否读老消息，产生读I/O），以及从服务器是否可读等因素建议下一次是从Master还是Slave拉取。



### 2. 消息存储

![](..\images\mq\rocketmq_design.png)

 RocketMQ 消息存储主要包括以下三个文件：

- **commitLog：**存储生产端写入的消息主体及元数据，单个文件大小默认1G，文件名为起始偏移量。

- **consumeQueue：**消息消费队列，主要作用是为了提高消息消费的性能，消费者可以根据 consumeQueue 查找待消费的消息。其中保存了指定 Topic 下的队列消息在 commitLog 中的起始物理偏移量 offset（8个字节），消息大小 size（4字节）和消息 Tag 的 hashCode（8个字节）值。

- **indexFile**：索引文件，提供了通过 key 或时间区间查询消息的方式。IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引。**索引文件由索引文件头（IndexHeader）+（ 槽位 Slot ）+（消息的索引内容）三部分构成。**

  

在上面的RocketMQ的消息存储整体架构图中可以看出，RocketMQ采用的是混合型的存储结构，即为Broker单个实例下所有的队列共用一个日志数据文件（即为CommitLog）来存储。RocketMQ的混合型存储结构(多个Topic的消息实体内容都存储于一个CommitLog中)针对Producer和Consumer分别采用了数据和索引部分相分离的存储结构，Producer发送消息至Broker端，然后Broker端使用同步或者异步的方式对消息刷盘持久化，保存至CommitLog中。只要消息被刷盘持久化至磁盘文件CommitLog中，那么Producer发送的消息就不会丢失。正因为如此，Consumer也就肯定有机会去消费这条消息。当无法拉取到消息后，可以等下一次消息拉取，同时服务端也支持长轮询模式，如果一个消息拉取请求未拉取到消息，Broker允许等待30s的时间，只要这段时间内有新消息到达，将直接返回给消费端。这里，RocketMQ的具体做法是，使用Broker端的后台服务线程—ReputMessageService不停地分发请求并异步构建ConsumeQueue（逻辑消费队列）和IndexFile（索引文件）数据。





### 3. 消息刷盘

![](..\images\mq\rocketmq_design_2.png)

消息刷盘即将消息写入到磁盘中，防止服务器宕机丢失消息。

RocketMQ 中的消息刷盘：

- **同步刷盘：**在消息持久化到磁盘后 RocketMQ 才会返回给生产者响应，性能较低，消息可靠性较好。
- **异步刷盘：**能够充分利用 OS 的 PageCache 的优势，只要消息写入PageCache即可将成功的ACK返回给Producer端。消息刷盘采用后台异步线程提交的方式进行，降低了读写延迟，提高了MQ的性能和吞吐量。

**页缓存（PageCache)是OS对文件的缓存，用于加速对文件的读写。一般来说，程序对文件进行顺序读写的速度几乎接近于内存的读写速度，主要原因就是由于OS使用PageCache机制对读写访问操作进行了性能优化，将一部分的内存用作PageCache**。

另外，RocketMQ主要通过MappedByteBuffer对文件进行读写操作。其中，利用了NIO中的FileChannel模型将磁盘上的物理文件直接映射到用户态的内存地址中（这种Mmap的方式减少了传统IO将磁盘文件数据在操作系统内核地址空间的缓冲区和用户应用程序地址空间的缓冲区之间来回进行拷贝的性能开销），将对文件的操作转化为直接对内存地址进行操作，从而极大地提高了文件的读写效率（正因为需要使用内存映射机制，故RocketMQ的文件存储都使用定长结构来存储，方便一次将整个文件映射至内存）。



> #### 零拷贝刷盘

以 Linxu 系统文件下载为例，服务端的主要任务是：将服务端主机磁盘中的文件不做修改地从已连 接的socket发出去。操作系统底层I/O过程如下图所示：

![](..\images\mq\zero-copy-1.png)

在这个过程中，总共设计到了四次拷贝：

1.  将磁盘文件拷贝到内核空间的页缓存中；
2.  将文件拷贝到用户空间的缓存中；
3.  将用户空间缓存中的文件拷贝写入到 Socket 缓冲区；
4.  将 Socket 缓冲区中的文件拷贝进行网络传输。

**DMA也就是直接内存访问（Direct Memory Access）技术，在进行 I/O 设备和内存的数据传输的时候，数据搬运的工作全部交给 DMA 控制器，而 CPU 不再参与任何与数据搬运相关的事情，这样 CPU 就可以去处理别的事务**。

零拷贝主要的任务就是避免CPU将数据从一块存储拷贝到另外一块存储，主要就是利用各种零拷贝技术，避免让CPU做大量的数据拷贝任务，减少不必要的拷贝，或者让别的组件来做这一类简单的数据传输任务，让CPU解脱出来专注于别的任务。这样就可以让系统资源的利用更加有效。 

原理是磁盘上的数据会通过DMA被拷贝的内核缓冲区，接着操作系统会把这段内核缓冲区与应用程序共享，这样就不需要把内核缓冲区的内容往用户空间拷贝。应用程序再调用 write(),操作系统直接将内核缓冲区的内容拷贝到socket缓冲区中，这一切都发生在内核态，最后，socket缓冲区再把数据发到网卡去。

![](..\images\mq\zero-copy-2.png)

### 4. 负载均衡

RocketMQ中的负载均衡都在Client端完成，具体来说的话，主要可以分为Producer端发送消息时候的负载均衡和Consumer端订阅消息的负载均衡。

#### 4.1 Producer的负载均衡

Producer端在发送消息的时候，会先根据Topic找到指定的TopicPublishInfo，在获取了TopicPublishInfo路由信息后，RocketMQ的客户端在默认方式下selectOneMessageQueue()方法会从TopicPublishInfo中的messageQueueList中选择一个队列（MessageQueue）进行发送消息。具体的容错策略均在MQFaultStrategy这个类中定义。这里有一个sendLatencyFaultEnable开关变量，如果开启，在随机递增取模的基础上，再过滤掉not available的Broker代理。所谓的"latencyFaultTolerance"，是指对之前失败的，按一定的时间做退避。例如，如果上次请求的latency超过550Lms，就退避3000Lms；超过1000L，就退避60000L；如果关闭，采用随机递增取模的方式选择一个队列（MessageQueue）来发送消息，latencyFaultTolerance机制是实现消息发送高可用的核心关键所在。

#### 4.2 Consumer的负载均衡

在RocketMQ中，Consumer端的两种消费模式（Push/Pull）都是基于拉模式来获取消息的，而在Push模式只是对pull模式的一种封装，其本质实现为消息拉取线程在从服务器拉取到一批消息后，然后提交到消息消费线程池后，又“马不停蹄”的继续向服务器再次尝试拉取消息。如果未拉取到消息，则延迟一下又继续拉取。在两种基于拉模式的消费方式（Push/Pull）中，均需要Consumer端在知道从Broker端的哪一个消息队列—队列中去获取消息。因此，有必要在Consumer端来做负载均衡，即Broker端中多个MessageQueue分配给同一个ConsumerGroup中的哪些Consumer消费。



### 5. 事务消息

Apache RocketMQ在4.3.0版中已经支持分布式事务消息，这里RocketMQ采用了2PC的思想来实现了提交事务消息，同时增加一个补偿逻辑来处理二阶段超时或者失败的消息，如下图所示。

![](..\images\mq\rocketmq_tx.png)

**事务消息的大致分为两个流程：正常事务消息的发送及提交、事务消息的补偿流程。**

- **事务提交和发送**

1. 发送 half 消息；
2. 服务端响应消息写入结果；
3. 根据结果执行本地事务；
4. 根据本地事务执行 commit 或 rollback；

- **事务消息补偿**

1. 对没有 commit 或 rollback 的事务进行一次回查；
2. 生产者收到回查消息，检查本地事务消息状态；
3. 根据本地事务状态重新 commit 或 rollback。



> #### 事务消息的限制

1. 事务消息不支持延时消息和批量消费；
2. 事务性消息可能不止一次被检查或消费；
3. 事务消息的生产者 ID 不能和其他类型消息生产者 ID 共享。与其他类型的消息不 15 同，事务消息允许反向查询、MQ服务器能通过它们的生产者 ID 查询到消费者。
4. 事务消息将在 Broker 配置文件中的参数 transactionMsgTimeout 这样的特定时间之后被检查。当发送事务消息时，用户还可以通过设置用户属性 CHECK_IMMUNITY_TIME_I N_SECONDS 来改变这个限制，该参数优先于 transactionMsgTimeout 参数。
5. 为了避免单个消息被检查太多次而导致半队列消息累积，默认单个消息的检查次数限制为 15 次，但是用户可以通过 Broker 配置文件的 transactionCheckMax参数来修改。如果已经检查某条消息超过 N 次的话（ N = transactionCheckMax ） 则 Broker 将丢弃此消息，并在默认情况下同时打印错误日志。用户可以通过重写 AbstractTransactionCheckListener 来修改这个行为。



## 三 消息可靠性

### 1. 生产者可靠性

RocketMQ 生产者发送消息会回调`SendCallback`接口，同步发送会返回`SendRuslt`，根据返回信息或回调判断是否发送成功，同时提供了失败重试机制。



### 2. Broker持久化

RocketMQ 持久化提供了同步和异步两种方式，同时主从信息同步也分为同步和异步。



### 3. 消费者

与 RabbitMQ 类型，RocketMQ 中也需要手动确认消息。



## 四 消息堆积

1. 提高消息并行度，通过增加消费数量，或者增加消费者中消费线程数量；
2. 批量消费消息；
3. 跳过不重要的消息；对于消费者消费消息一直较慢，且对业务数据要求不高，可以丢弃不重要的消息；
4. 优化消息消费过程。



## 参考

https://github.com/apache/rocketmq/tree/master/docs/cn