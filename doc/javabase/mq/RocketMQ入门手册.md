# RocketMQ入门手册

RocketMQ是一个分布式、队列模型的开源消息中间件，前身是MetaQ，是阿里研发的一个队列模型的消息中间件，后开源给apache基金会成为了apache的顶级开源项目，具有高性能、高可靠、高实时、分布式特点，

同时，广泛应用于多个领域，包括异步通信解耦、企业解决方案、金融支付、电信、电子商务、快递物流、广告营销、社交、即时通信、移动应用、手游、视频、物联网、车联网等。

具有以下特点：

- 能够保证严格的消息顺序
- 提供丰富的消息拉取模式
- 高效的订阅者水平扩展能力
- 实时的消息订阅机制
- 亿级消息堆积能力



# RocketMQ 架构原理分析

### RocketMQ 架构

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC80LzE0LzE3MTc3YzQ1N2FjMTM2Zjg?x-oss-process=image/format,png)

**NameServer （名称服务器）：**

- 提供轻量级的服务发现和路由。NameServer接受来自Broker群集的注册，并提供检测信号机制以检查Broker是否还存在
-  每个NameServer记录完整的路由信息（Broker 相关 Topic 等元信息，并给 Producer 提供 Consumer 查找 Broker 信息），提供相应的读写服务。

**Broker（消息服务器）：** 消息存储中心，接收来自 Producer 的消息并存储， Consumer 从这里取得消息

- 单个Broker节点与所有的NameServer节点保持长连接及心跳，并会定时将Topic信息注册到NameServer，（其底层通信是基于Netty实现的）
- Broker负责消息存储，以Topic为维度支持轻量级的队列，单机可以支撑上万队列规模，支持消息推拉模型。
- 具有上亿级消息堆积能力，同时可严格保证消息的有序性

**Producer （生产者）：**

- 负责产生消息，生产者向消息服务器发送由业务应用程序系统生成的消息

- 生产者支持分布式部署。 分布式生产者通过多种负载平衡模式将消息发送到Broker集群。 发送过程支持快速失败并且延迟低

- 三种方式发送消息：同步、异步和单向

**Consumer（消费者）：**

- 负责消费消息，消费者从消息服务器拉取信息并将其输入用户应用程序
- 也支持“推和拉”模型中的分布式部署。
-  它还支持集群使用和消息广播。 它提供了实时消息订阅机制，可以满足大多数消费者的需求。 



### Broker Server

Broker Server负责消息的存储和传递，消息查询，HA高可用等，Broker Server几个主要模块组成：

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC80LzE0LzE3MTc3YzQ4YjZmMjJhYzk?x-oss-process=image/format,png)

Remoting Module（远程模块）：broker入口，处理来自客户端的请求

Client Manager（客户端管理）：管理client（生产者/消费者）并维护消费者的主题订阅

Store Service（存储服务）：提供简单的API中数据库中存储或查询消息

HA Service（高可用服务）：提供master broker和slave broker之间的数据同步功能

Index Service（索引服务）：将message建立索引来提供快速的查询能力

### RocketMQ 整体流程

![整体流程](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC80LzE0LzE3MTc3YzQ3M2QyNTY0NmE?x-oss-process=image/format,png)

1. 启动 **NameServer**，NameServer启动后进行端口监听，等待 Broker、Producer、Consumer 连上来，相当于一个路由控制中心
2. **Broker** 启动，跟所有的 Namesrv 保持长连接，定时发送心跳包
   - 心跳包中，包含当前 Broker 信息(IP+端口等)以及存储所有 Topic 信息
   - 注册成功后，Namesrv 集群中就有 Topic 跟 Broker 的映射关系
3. 收发消息前，先创建 Topic 。创建 Topic 时，需要指定该 Topic 要存储在哪些 Broker上。也可以在发送消息时自动创建Topic
4. **Producer** 发送消息
   - 启动时，先跟 Namesrv 集群中的其中一台建立长连接，并从Namesrv 中获取当前发送的 Topic 存在哪些 Broker 上
   - 然后跟对应的 Broker 建立长连接，直接向 Broker 发消息
5. **Consumer** 消费消息
   - 跟其中一台 Namesrv 建立长连接，获取当前订阅 Topic 存在哪些 Broker 上
   - 然后直接跟 Broker 建立连接通道，开始消费消息*RocketMQ的消息领域模型

### RocketMQ Message

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC80LzE0LzE3MTc3YzQ1NzRhMTI0ZTY?x-oss-process=image/format,png)

**Topic（主题）：** 表示消息的第一级类型，是最细粒度的订阅单位（生产者传递消息和消费者提取消息标识）

- 一条消息必须有一个Topic
- 一个Group可以订阅多个Topic的消息
- Topic一般为领域范围，比如交易消息

**Tag（标签）：** 表示消息的第二级类型，可以是使用相同的Topic不同的Tag来表示同一业务模块的不同任务的消息，比如交易消息又可以分为：交易创建消息，交易完成消息等

- 助于保持代码整洁和一致
- 简化RocketMQ提供的查询系统

**Message（消息体）：** 消息是要传递的信息。 Message中必须包含一个Topic，可选Tag和key-vaule键值对

**Message Queue（消息队列）：** 所有消息队列都是持久化

- 一个Topic下可以有多个Queue
- Queue的引入使得消息的存储可以分布式集群化，具有了水平扩展能力

**Group（组）：** 分为Producer Group（生产者组）和Consumer Group（消费者组），具有相同角色组成Group

- 原生产者在交易后崩溃，broker可以联系同一生产者组的不同生产者实例以进行提交或回退交易。

- 消费者组的消费者实例必须具有完全相同的主题订阅

### RocketMQ 特性

 **Message Model（消息模式）：**

- Clustering（集群式）：当使用集群消费模式时，MQ 认为任意一条消息只需要被集群内的任意一个消费者处理即可
- Broadcasting（广播式）：当使用广播消费模式时，MQ 会将每条消息推送给集群内所有注册过的客户端，保证消息至少被每台机器消费一次

**Message Order（消息顺序）**

- 使用DefaultMQPushConsumer时，可以决定按顺序或同时使用消息

  - Orderly：有序地使用消息意味着消息的消费顺序与生产者为每个消息队列发送消息的顺序相同。（ 如果要处理必须强制执行全局顺序的情况，请确保您使用的主题只有一个消息队列）

  > 如果指定按顺序使用，则消息使用的最大并发度是使用者组订阅的消息队列数

  - Concurrently：同时使用消息时，消息使用的最大并发性仅受为每个使用方客户端指定的线程池限制

  > 在此模式下不再保证消息顺序

**Message  Types（消息类型）**

- 事务消息
- 顺序消息
- 延迟消息

## RocketMQ单机版安装

1. **下载编译源码**

   ```
     # 下载$ 
     > wget wget http://mirror.bit.edu.cn/apache/rocketmq/4.6.0/rocketmq-all-4.6.0-source- > 
     # 解压$
     >unzip rocketmq-all-4.7.0-source-release.zip
     > cd rocketmq-all-4.7.0/
     # 编译$
     > mvn -Prelease-all -DskipTests clean install -U
     > cd distribution/target/rocketmq-4.7.0/rocketmq-4.7.0
   ```

   

2. **启动 Name Server**

   ```
    # 启动 Name Server 服务
    > nohup sh bin/mqnamesrv &
    # 启动完成后，查看日志$
    > tail -f ~/logs/rocketmqlogs/namesrv.log
     The Name Server boot success...
   ```

   

3. **启动 Broker**

   在 `conf` 目录下，RocketMQ 提供了多种 Broker 的配置文件：

   - `broker.conf` ：单主，异步刷盘。
   - `2m/` ：双主，异步刷盘。
   - `2m-2s-async/` ：两主两从，异步复制，异步刷盘。
   - `2m-2s-sync/` ：两主两从，同步复制，异步刷盘。
   - `dledger/` ：Dledger 集群，至少三节点

   ```
    # 启动 Broker服务
    > nohup sh bin/mqbroker -n localhost:9876 &
    # 启动完成后，查看日志$
    > tail -f ~/logs/rocketmqlogs/broker.log 
     The broker[%s, 172.30.30.233:10911] boot success...
   ```

   其中，参数：

   - 通过 `-c` 参数，配置读取的主 Broker 配置

   - 通过 `-n` 参数，设置 RocketMQ Namesrv 地址

   

4. **Send & Receive Messages（消息发送与接收）**

   在发送/接收消息之前，我们需要告知client（生产者/消费者）Name Servers的地址。 RocketMQ提供了多种方法来实现：

   - 在代码中设置：producer.setNamesrvAddr("ip:port")
   - java属性配置：rocketmq.namesrv.addr
   - 环境变量配置：NAMESRV_ADDR
   - HTTP Endpoint

   为简单起见，我们使用环境变量：NAMESRV_ADDR，如下所示：

   ```
    # 设置 Name Servers的地址$
    > export NAMESRV_ADDR=localhost:9876
    # 生产消息$
    > sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
    SendResult [sendStatus=SEND_OK, msgId= ...
    # 消费消息$ 
    > sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
    ConsumeMessageThread_%d Receive New Messages: [MessageExt...
   ```


> 各位看官还可以吗？喜欢的话，动动手指点个💗，点个关注呗！！谢谢支持！
>
> 欢迎关注，原创技术文章第一时间推出
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC80LzE0LzE3MTc5MjY5MGIxOWUwYjg?x-oss-process=image/format,png)
