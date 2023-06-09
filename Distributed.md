# CAP理论

CAP原则又称CAP定理，指的是在一个分布式系统中，Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性），三者不可兼得。

* 一致性（C）：对某个指定的客户端来说，读操作能返回最新的写操作结果
* 可用性（A）：非故障节点在合理的时间返回合理的响应
* 分区容错性（P）：分区容错性是指当网络出现分区（两个节点之间无法连通）之后，系统能否继续履行职责

CAP理论就是说在分布式系统中，最多只能实现上面的两点。而由于当前的网络硬件肯定会出现延迟丢包等问题，所以考虑最差情况，分区容忍性是一般是需要实现的

虽然 CAP 理论定义是三个要素中只能取两个，但放到分布式环境下来思考，我们会发现必须选择 P（分区容忍）要素，因为网络本身无法做到 100% 可靠，有可能出故障，所以分区是一个必然的现象。如果我们选择了 CA 而放弃了 P，那么当发生分区现象时，为了保证 C，系统需要禁止写入，当有写入请求时，系统返回 error（例如，当前系统不允许写入），这又和 A 冲突了，因为 A 要求返回 no error 和 no timeout。因此，分布式系统理论上不可能选择 CA 架构，只能选择 CP 或者 AP 架构。

BASE理论的核心思想是：即使无法做到强一致性，但每个应用都可以根据自身业务特点，采用适当的方式来使系统达到最终一致性。

# 锁

## 为什么用

在分布式环境中，需要一种跨JVM的互斥机制来控制共享资源的访问。

例如，为避免用户操作重复导致交易执行多次，使用分布式锁可以将第一次以外的请求在所有服务节点上立马取消掉。如果使用事务在数据库层面进行限制也能实现的但会增大数据库的压力。

例如，在分布式任务系统中为避免统一任务重复执行，某个节点执行任务之后可以使用分布式锁避免其他节点在同一时刻得到任务。

## 实现方式

### 数据库

在数据库中创建一个表，表中包含方法名等字段，并在方法名字段上创建唯一索引，想要执行某个方法，就使用这个方法名向表中插入数据，成功插入则获取锁，执行完成后删除对应的行数据释放锁。

```sql
DROP TABLE IF EXISTS `method_lock`;
CREATE TABLE `method_lock` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `method_name` varchar(64) NOT NULL COMMENT '锁定的方法名',
  `desc` varchar(255) NOT NULL COMMENT '备注信息',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uidx_method_name` (`method_name`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8 COMMENT='锁定中的方法';
```

执行某个方法后，插入一条记录

```sql
INSERT INTO method_lock (method_name, desc) VALUES ('methodName', '测试的methodName');
```

因为我们对method_name做了唯一性约束，这里如果有多个请求同时提交到数据库的话，数据库会保证只有一个操作可以成功，那么我们就可以认为操作成功的那个线程获得了该方法的锁，可以执行方法体内容。

成功插入则获取锁，执行完成后删除对应的行数据释放锁：

```sql
delete from method_lock where method_name ='methodName';
```

优点：易于理解实现

缺点：

1. 没有锁失效自动删除机制，因为有可能出现成功插入数据后，服务器宕机了，对应的数据没有被删除，当服务恢复后一直获取不到锁，所以，需要在表中新增一列，用于记录失效时间，并且需要有定时任务清除这些失效的数据
2. 吞吐量很低
3. 单点故障问题
4. 轮询获取锁状态方式太过低效

### 基于Redis

NX是Redis提供的一个原子操作，如果指定key存在，那么NX失败，如果不存在会进行set操作并返回成功。我们可以利用这个来实现一个分布式的锁，主要思路就是，set成功表示获取锁，set失败表示获取失败，失败后需要重试。再加上EX参数可以让该key在超时之后自动删除。

```java
public void lock(String key, String request, int timeout) throws InterruptedException {
    Jedis jedis = jedisPool.getResource();

    while (timeout >= 0) {
        String result = jedis.set(LOCK_PREFIX + key, request, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, DEFAULT_EXPIRE_TIME);
        if (LOCK_MSG.equals(result)) {
            jedis.close();
            return;
        }
        Thread.sleep(DEFAULT_SLEEP_TIME);
        timeout -= DEFAULT_SLEEP_TIME;
    }
}
```

优点

1. 吞吐量高
2. 有锁失效自动删除机制，保证不会阻塞所有流程

缺点

1. 单点故障问题
2. 锁超时问题：如果A拿到锁之后设置了超时时长，但是业务还未执行完成且锁已经被释放，此时其他进程就会拿到锁从而执行相同的业务。如何解决？Redission定时延长超时时长避免过期。为什么不直接设置为永不超时？为了防范业务方没写解锁方法或者发生异常之后无法进行解锁的问题
3. 轮询获取锁状态方式太过低效

### 基于ZooKeeper

1. 当客户端对某个方法加锁时，在Zookeeper上的与该方法对应的指定节点的目录下，生成一个临时有序节点
2. 判断该节点是否是当前目录下最小的节点，如果是则获取成功。如果不是，则获取失败，并获取上一个临时有序节点，对该节点进行监听，当节点删除时通知唯一的客户端

优点：

1. 解决锁超时问题。因为Zookeeper的写入都是顺序的，在一个节点创建之后，其他请求再次创建便会失败，同时可以对这个节点进行Watch，如果节点删除会通知其他节点抢占锁
2. 能通过watch机制高效通知其他节点获取锁，避免惊群效应
3. 有锁失效自动删除机制，保证不会阻塞所有流程

缺点：

1. 性能不如Redis
2. 强依赖Zookeeperz，如果原来系统不用Zookeeperz那就需要维护一套Zookeeper

# 事务方案

## 两阶段提交方案/XA方案

所谓的 XA 方案，即：两阶段提交，有一个**事务管理器**的概念，负责协调多个数据库（资源管理器）的事务，事务管理器先问问各个数据库你准备好了吗？如果每个数据库都回复 ok，那么就正式提交事务，在各个数据库上执行操作。如果任何其中一个数据库回答不 ok，那么就回滚事务。

这种分布式事务方案，比较适合单块应用里，跨多个库的分布式事务，而且因为严重依赖于数据库层面来搞定复杂的事务，效率很低，绝对不适合高并发的场景。如果要玩儿，那么基于 `Spring + JTA` 就可以搞定，自己随便搜个 demo 看看就知道了。

这个方案，我们很少用，一般来说**某个系统内部如果出现跨多个库**的这么一个操作，是**不合规**的。我可以给大家介绍一下， 现在微服务，一个大的系统分成几十个甚至几百个服务。一般来说，我们的规定和规范，是要求**每个服务只能操作自己对应的一个数据库**。

如果你要操作别的服务对应的库，不允许直连别的服务的库，违反微服务架构的规范，你随便交叉胡乱访问，几百个服务的话，全体乱套，这样的一套服务是没法管理的，没法治理的，可能会出现数据被别人改错，自己的库被别人写挂等情况。

如果你要操作别人的服务的库，你必须是通过**调用别的服务的接口**来实现，绝对不允许交叉访问别人的数据库。

## TCC 方案

TCC 的全称是：Try、Confirm、Cancel。

- Try 阶段：这个阶段说的是对各个服务的资源做检测以及对资源进行**锁定或者预留**。
- Confirm 阶段：这个阶段说的是在各个服务中**执行实际的操作**。
- Cancel 阶段：如果任何一个服务的业务方法执行出错，那么这里就需要**进行补偿**，就是执行已经执行成功的业务逻辑的回滚操作。（把那些执行成功的回滚）

## 本地消息表

本地消息表其实是国外的 ebay 搞出来的这么一套思想。

这个大概意思是这样的：

1. A 系统在自己本地一个事务里操作同时，插入一条数据到消息表。
2. 接着 A 系统将这个消息发送到 MQ 中去。
3. B 系统接收到消息之后，在一个事务里，往自己本地消息表里插入一条数据，同时执行其他的业务操作，如果这个消息已经被处理过了，那么此时这个事务会回滚，这样**保证不会重复处理消息**。
4. B 系统执行成功之后，就会更新自己本地消息表的状态以及 A 系统消息表的状态。
5. 如果 B 系统处理失败了，那么就不会更新消息表状态，那么此时 A 系统会定时扫描自己的消息表，如果有未处理的消息，会再次发送到 MQ 中去，让 B 再次处理。
6. 这个方案保证了最终一致性，哪怕 B 事务失败了，但是 A 会不断重发消息，直到 B 那边成功为止。

## 可靠消息最终一致性方案

这个的意思，就是干脆不要用本地的消息表了，直接基于 MQ 来实现事务。比如阿里的 RocketMQ 就支持消息事务。

大概的意思就是：

1. A 系统先发送一个 prepared 消息到 mq，如果这个 prepared 消息发送失败那么就直接取消操作别执行了。
2. 如果这个消息发送成功过了，那么接着执行本地事务，如果成功就告诉 mq 发送确认消息，如果失败就告诉 mq 回滚消息。
3. 如果发送了确认消息，那么此时 B 系统会接收到确认消息，然后执行本地的事务。
4. mq 会自动**定时轮询**所有 prepared 消息回调你的接口，问你，这个消息是不是本地事务处理失败了，所有没发送确认的消息，是继续重试还是回滚？一般来说这里你就可以查下数据库看之前本地事务是否执行，如果回滚了，那么这里也回滚吧。这个就是避免可能本地事务执行成功了，而确认消息却发送失败了。
5. 这个方案里，要是系统 B 的事务失败了咋办？重试咯，自动不断重试直到成功，如果实在是不行，要么就是针对重要的资金类业务进行回滚，比如 B 系统本地回滚后，想办法通知系统 A 也回滚。或者是发送报警由人工来手工回滚和补偿。
6. 这个还是比较合适的，目前国内互联网公司大都是这么玩儿的，要不你举用 RocketMQ 支持的，要不你就自己基于类似 ActiveMQ？RabbitMQ？自己封装一套类似的逻辑出来，总之思路就是这样子的。

## 最大努力通知方案

这个方案的大致意思就是：

1. 系统 A 本地事务执行完之后，发送个消息到 MQ。
2. 这里会有个专门消费 MQ 的**最大努力通知服务**，这个服务会消费 MQ 然后写入数据库中记录下来，或者是放入个内存队列也可以，接着调用系统 B 的接口。
3. 要是系统 B 执行成功就 ok 了。要是系统 B 执行失败了，那么最大努力通知服务就定时尝试重新调用系统 B，反复 N 次，最后还是不行就放弃。

# 消息队列

## 优点

1. 减少请求响应时间。比如注册功能需要调用第三方接口来发短信，如果等待第三方响应可能会需要很多时间
2. 服务之间解耦。主服务只关心核心的流程，其他不重要的、耗费时间流程是否如何处理完成不需要知道，只通知即可
3. 流量削锋。对于不需要实时处理的请求来说，当并发量特别大的时候，可以先在消息队列中作缓存，然后陆续发送给对应的服务去处理

## 缺点

1. 系统可用性降低。系统引入的外部依赖越多，越容易挂掉
2. 系统复杂度提高。保证消息没有重复消费？处理消息丢失的情况？保证消息传递的顺序性？

## 消息重复消费问题

1. 消费端处理消息的业务逻辑保持幂等性，在消费端实现
2. 利用一张日志表来记录已经处理成功的消息的ID，如果新到的消息ID已经在日志表中，那么就不再处理这条消息。消息系统实现，也可以消费端实现

## 消息丢失问题

### 生产者弄丢了数据

生产者将数据发送到 RabbitMQ 的时候，可能数据就在半路给搞丢了。

RabbitMQ 提供的事务功能，就是生产者发送数据之前开启 RabbitMQ 事务。然后发送消息，如果消息没有成功被 RabbitMQ 接收到，那么生产者会收到异常报错。但吞吐量会下来，因为太耗性能。

可以开启confirm模式，你每次写的消息都会分配一个唯一的 id，然后如果写入了 RabbitMQ 中，RabbitMQ 会给你回传一个ack消息，告诉你说这个消息 ok 了。事务机制是同步的，但confirm机制是异步的，发送个消息之后就可以发送下一个消息，RabbitMQ 接收了之后会异步回调你一个接口通知你这个消息接收到了。所以用confirm机制。

### RabbitMQ 弄丢了数据

RabbitMQ 自己挂掉导致数据丢失。

开启 RabbitMQ 的持久化，消息写入之后会持久化到磁盘，哪怕是 RabbitMQ 自己挂了，恢复之后会自动读取之前存储的数据。

### 消费端弄丢了数据

RabbitMQ 如果丢失了数据，主要是因为你消费的时候，刚消费到，还没处理，结果进程挂了，比如重启了，RabbitMQ 认为你都消费了，这数据就丢了。

关闭 RabbitMQ 的自动ack，可以通过一个 api 来调用就行，然后每次你自己代码里确保处理完的时候，再在程序里ack一把。RabbitMQ 就认为你还没处理完，这个时候 RabbitMQ 会把这个消费分配给别的 consumer 去处理，消息是不会丢的。

## 集群模式

### 普通集群模式

在多台机器上启动多个 RabbitMQ 实例。你创建的 queue，只会放在一个 RabbitMQ 实例上，但是每个实例都同步 queue 的元数据（元数据可以认为是 queue 的一些配置信息，通过元数据，可以找到 queue 所在实例）。你消费的时候，实际上如果连接到了另外一个实例，那么那个实例会从 queue 所在实例上拉取数据过来。

缺点是不能保证高可用、还有拉去数据的开销、以及单实例的性能瓶颈，所以这个方案是为了提高吞吐量的。

### 镜像集群模式

每个 RabbitMQ 节点都有这个 queue 的一个完整镜像，包含 queue 的全部数据的意思。然后每次你写消息到 queue 的时候，都会自动把消息同步到多个实例的 queue 上。任何一个机器宕机了，没事儿，其它机器（节点）还包含了这个 queue 的完整数据，别的 consumer 都可以到其它节点上去消费数据。

缺点是即时满足了高可用，但因为同步数据量太重导致难以扩展节点，也没有在架构上实现负载均衡，可以参考Redis的集群模式进行优化。

### Kafka 的高可用性

Kafka 一个最基本的架构认识：由多个 broker 组成，每个 broker 是一个节点。你创建一个 topic，这个 topic 可以划分为多个 partition，每个 partition 可以存在于不同的 broker 上，每个 partition 就放一部分数据。

每个 partition 的数据都会同步到其它机器上，形成自己的多个 replica 副本。所有 replica 会选举一个 leader 出来，那么生产和消费都跟这个 leader 打交道，然后其他 replica 就是 follower。写的时候，leader 会负责把数据同步到所有 follower 上去，读的时候就直接读 leader 上的数据即可。

如果某个 broker 宕机了，没事儿，那个 broker上面的 partition 在其他机器上都有副本的，如果这上面有某个 partition 的 leader，那么此时会从 follower 中重新选举一个新的 leader 出来，大家继续读写那个新的 leader 即可。这就有所谓的高可用性了。

# ID生成方式

## UUID

优点

1. 本地生成没有了网络之类的消耗，效率非常高。

缺点

1. 不易于存储：UUID太长，16字节128位，通常以36长度的字符串表示，很多场景不适用。
2. 信息不安全：基于MAC地址生成UUID的算法可能会造成MAC地址泄露，这个漏洞曾被用于寻找梅丽莎病毒的制作者位置。

## snowflake

这种方案把64-bit分别划分成多段（机器、时间）

优点

1. 毫秒数在高位，自增序列在低位，整个ID都是趋势递增的。
2. 本地生成没有了网络之类的消耗，效率非常高。
3. 可以根据自身业务特性分配bit位，非常灵活。

缺点

1. 强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务会处于不可用状态。

## 数据库

可以利用 MySQL 中的自增属性 auto_increment 来生成全局唯一 ID，也能保证趋势递增。 但这种方式太依赖 DB，如果数据库挂了那就非常容易出问题。

优点

1. 非常简单，利用现有数据库系统的功能实现，成本小，有DBA专业维护。
2. ID号单调自增，可以实现一些对ID有特殊要求的业务。

缺点

1. 强依赖DB，当DB异常时整个系统不可用，属于致命问题。配置主从复制可以尽可能的增加可用性，但是数据一致性在特殊情况下难以保证。主从切换时的不一致可能会导致重复发号。
2. ID发号性能瓶颈限制在单台MySQL的读写性能。

# 一致 Hash 算法

分布式缓存中，如何将数据均匀的分散到各个节点中，并且尽量的在加减节点时能使受影响的数据最少。

## Hash 取模

随机放置就不说了，会带来很多问题。通常最容易想到的方案就是 `hash `取模了。

可以将传入的 Key 按照 `index = hash(key) % N` 这样来计算出需要存放的节点。其中 hash 函数是一个将字符串转换为正整数的哈希映射方法，N 就是节点的数量。

这样可以满足数据的均匀分配，但是这个算法的容错性和扩展性都较差。

比如增加或删除了一个节点时，所有的 Key 都需要重新计算，显然这样成本较高，为此需要一个算法满足分布均匀同时也要有良好的容错性和拓展性。

## 一致 Hash 算法

一致性哈希算法也是使用取模的方法，但是取模算法是对服务器的数量进行取模，而一致性哈希算法是对 2^32 取模，具体步骤如下：

步骤一：一致性哈希算法将整个哈希值空间按照顺时针方向组织成一个虚拟的圆环，称为 Hash 环。
步骤二：接着将各个服务器使用 Hash 函数进行哈希，具体可以选择服务器的IP或主机名作为关键字进行哈希，从而确定每台机器在哈希环上的位置。
步骤三：最后使用算法定位数据访问到相应服务器：将数据key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针寻找，第一台遇到的服务器就是其应该定位到的服务器。
