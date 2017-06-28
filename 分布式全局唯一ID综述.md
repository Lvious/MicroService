### <center>分布式唯一ID生成综述</center>
##### <center>吕晓强 51164500227</center>
#### 1.研究背景
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在分布式系统中，往往需要对大量的数据和消息进行标识。例如团购网站中的餐饮，酒店，电影，金融，支付等产品的支付中。数据日益增长，对于数据分库分表后需要有一个全局的唯一标记来标识一条数据或消息。显然，在高并发的情况下，如果按照传统的生成自增主键的方式下，就会出现重复ID的情况。因此，产生一个全局唯一ID，不仅局限于数据库中的ID主键，也可以适用于其他分布式环境下的唯一标识，比如全局唯一事务，日志追踪时的唯一ID等。全局唯一ID就需要保证这两个需求：1. 全局唯一 2. 趋势有序  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;分布式全局唯一id的具有十分重要的意义，同时也有很多不同的的实现方式。单纯的生成全局ID并不是什么难题，但是生成的ID通常要满足分片的一些要求：
>1. 不能有单点故障。  

>2. 以时间为序，或者ID里包含时间。这样一可以少一个索引，二是冷热数据容易分离。  

>3. 可以控制ShardingId。比如某一个用户的文章放在同一个分片内，查询效率高，修改容易。  

>4. 不要太长，最好64bit。使用long易操作。

#### 2.研究方法  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对分布式系统全局唯一ID的研究一直十分热门，传统技术发展与新技术出现，促进了分布式系统的发展和应用，工业界由实际应用驱动，出现了许多有实用价值的分布式全局唯一ID生成方法。
标识的生成方法有很多，有集中式的，分布式的；有后端的，前端的，当然还有人工的。 并没有一种通用的生成方法来适应各种应用场景。下述为三种实现思路。
##### 2.1 基于数据库生成
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一般包含以下几种
- MySQL AUTO_INCREMENT 特性
- Postgres SEQUENCE 特性
- Oracle SEQUENCE 特性
- Flickr Ticket Servers

一般这种方式都可以设定初始值，以及增量步长。
###### 2.1.1 分布式MySQL
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每台机器设置不同的初始值，且步长和机器数一致。比如有两台机器。设置步长step为2,server1的初始值为1(1,3,5,7,9...),server2的初始值为2（2,4,6,8,10...）。假设需要部署N台机器，步长需设置为N，每台的初始值为0,1，2...N-1。这种架构方式简单易用，容易实现，可以满足一般业务的需求。但是需要系统水平扩容时却比较困难，ID没有了单调递增的特性，为准递增，数据库的压力比较大，每次获取ID都需读写一次数据库。
###### 2.1.2 发号器
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由发号器server管理，client每次请求server，由server返回一段连续的ID，同时更新ID段，client负责消费分配ID,该client消耗完该段ID后，继续请求server。该方式的特点是准连续的自增数字。这种方法下，批量是关键，否则，每个ID都调用一次，无法承受负载。
##### 2.2 划分命名空间并行生成
###### 2.2.1 UUID
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Universally Unique Identifier，简称UUID，UUID的目的，是让分布式系统中的所有元素，都能有唯一的辨识信息，而不需要通过中央控制端来做辨识信息的指定，到目前为止业界一共有5种方式生成UUID.
- UUID 1: 依据当前计算机的MAC地址和时钟来生成uuid。
- UUID 2: 和版本1类似，不过使用域标示符和本地UID代替了版本1中的时钟信息。  
- UUID 3: 根据url，域标示符等标示符做MD5 Hash产生的。  
- UUID 4: 根据产生的随机数来生成。  
- UUID 5: 和版本3类似，只不过替换成了SHA-1算法。  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;目前最广泛应用的UUID，是微软公司的全局唯一标识符（GUID）。UUID是由一组32位数的16进制数字所构成，是故UUID理论上的总数为16^32=2^128，约等于3.4 x 10^38。也就是说若每纳秒产生1兆个UUID，要花100亿年才会将所有UUID用完。UUID的标准型式包含32个16进制数字，以连字号分为五段，形式为8-4-4-4-12的32个字符,示例：
``550e8400-e29b-41d4-a716-446655440000``。
随机产生的UUID的128个比特中，有122个比特是随机产生，4个比特在Randomly generated UUID中被使用，还有两个是在其变体中使用。在这种情况下，两个UUID拥有相同值的概率为约为0.00000000006 ，拥有相同值的UUID的概率极低。
优点：1.本地生成，不需要控制中心管理，成本低 2.性能好  
缺点：1.Id共128bits 2.id没有次序关系，不能隐含信息
###### 2.2.2 Hibernate
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Hibernate的CustomVersionOneStrategy.java，解决了之前UUID 1的两个问题（同一台机器进程并发问题，相同时间戳并发问题）

- 时间戳(6bytes, 48bit)：毫秒级别的，从1970年算起，能用8925年
- 顺序号(2bytes, 16bit, 最大值65535): 没有时间戳过了一秒要归零，相互独立，short溢出到了负数就归0。
- 机器标识(4bytes 32bit): 取localHost的IP地址，IPV4为4个byte，如果是IPV6要16个bytes，只拿前4个byte。
- 进程标识(4bytes 32bit)： 用当前时间戳右移8位再取整数。

机器进程和进程标识组成的64bit Long几乎不变，只变动另一个Long。

###### 2.2.3  MongoDB ObjectID
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MongoDB中每一条记录都有一个id字段用来唯一标示本记录。如果用户插入数据时没有显示提供id字段，那么系统会自动生成一个。ObjectID一共12Bytes，设计的时候充分考虑了分布式环境下使用的情况，所以能保证在一个分布式MongoDB集群中唯一。ObjectID格式为：0~4 Byte是Unix Timestamp。 4~7 Byte是当前机器“hostname/mac地址/虚拟编号”其中之一的MD5结果的前3个字节。 7~9 Byte是当前进程的PID。 9~12Byte是累加计数器或是一个随机数（只有当不支持累加计数器时才用随机数）。 最后生成的仍然是一个用16进制表示的串
优点：1.ObjectID时间上有序。2.ObjectID本身包含了有用的信息，通过直接解码ObjectID即可获取到相应的信息。
缺点：1.当timestamp段一样时，由于MD5只取前3Bytes，有可能造成pc段一样，这样就会有重复的ID出现。2. ID间隙较大（当某段时间不生成ID，那么这个timestamp段浪费许多空间）
###### 2.2.4 Twitter的snowflake派号器
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Snowflake是twitter开源的一款独立的适用于分布式环境的ID生成服务器。snowflake是一个派号器，是一种以划分命名空间来生成ID的一种算法,这种方案把64-bit分别划分成多段，分开来标示机器、时间等，生成的ID是64Bits，同时满足高性能（>10K ids/s），低延迟（<2ms）和高可用。与MongoDB ObjectID类似这里生成的ID也是时间上有序的。编码方式也和ObjectID类似。0-40位bits以毫秒为单位的timestamp，时间起点从2011-01-01开始。41-50 bits为节点ID,51-63 bits为累加计数器。

优点：
- 毫秒数在高位，自增序列在低位，整个ID都是趋势递增的。
- 不依赖数据库等第三方系统，以服务的方式部署，稳定性更高，生成ID的性能也是非常高的。
- 可以根据自身业务特性分配bit位，非常灵活

缺点：
- 强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务会处于不可用状态。

###### 2.2.5 Boundary Flake
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;基于snowflake，使其变成一个网络服务，提供128-bit长的ID生成服务，变化为：
- ID 长度扩展到 128 bits:
- 最高 64 bits 时间戳;
- 然后是 48 bits 的 Worker 号 (和 Mac 地址一样长);
- 最后是 16 bits 的 Seq Number
- 由于它用 48 bits 作为 Worker ID, 和 Mac 地址的长度一样, 这样启动时不需要和 Zookeeper 通讯获取Worker ID. 做到了完全的去中心化
- 基于 Erlang

这样做的目的是用更多的 bits 实现更小的冲突概率, 这样就支持更多的 Worker 同时工作. 同时, 每毫秒能分配出更多的 ID

##### 2.2.6 Instagram 的做法
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;先简单介绍下Instagram的分布式存储方案:
- 先把每个Table划分程多个逻辑切片（logic Shard），数目可以很大
- 制定规则，规定每个逻辑切片存储到数据库实例的位置。
- 每个Table指定一个字段作为分片字段，如用户表的uid。
- 插入新数据时，先跟据分片字段的值，决定划分到数据库实例的位置。   

Instagram的Unique ID的组成：
- 41 bits: Timestamp (毫秒)
- 13 bits: 每个 logic Shard 的代号 (最大支持 8 x 1024 个 logic Shards)
- 10 bits: sequence number; 每个 Shard 每毫秒最多可以生成 1024 个 ID  

生成 unique ID 时, 41 bits 的 Timestamp 和 Snowflake 类似，对于
Logic Shard编号：
- 假设插入一条新的用户记录, 插入时, 根据 uid 来判断这条记录应该被插入到哪个 logic Shard 中.
- 假设当前要插入的记录会被插入到第 1341 号 logic Shard 中 (假设当前的这个 Table 一共有 2000 个 logic Shard)
- 新生成 ID 的 13 bits 段要填的就是 1341 这个数字

sequence number 利用数据库实例每个table的 auto-increment sequence 生成。优点在于利用 logic Shard 号来替换 Snowflake 使用的 Worker 号, 不需要到中心节点获取 Worker 号. 做到了完全去中心化，另外一个附带的好处是，可以直接通过ID直接知道这条记录被存放到的logic Shard的位置，同时, 日后做数据迁移的时, 也是按 logic Shard 为单位做数据迁移, 这种做法不会影响到今后的数据迁移。

##### 2.3 基于分布式集群协调器生成
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在不使用数据库的情况下，通过一个后台服务对外提供高可用的、固定步长标识生成，则需要分布式的集群协调器进行。
一般的，主流协调器有两类：

- 以强一致性为目标的：ZooKeeper为代表

- 以最终一致性为目标的：Consul为代表

ZooKeeper的强一致性，是由Paxos协议保证的；Consul的最终一致性，是由Gossip协议保证的。

#### 3.总结
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;分布式唯一ID的生成方式多种多样，主要列举了三种思路下ID的生成方法。概括起来主要有以下几点：唯一性，时间相关，粗略有序。每种方法既有优点，又有缺点，但最终目的还是为了使分布式系统中的数据实体，消息实体，事务实体做一个全局标识。使其能更好的服务于分布式系统。
#### 4. 参考文献：
[1] [Leaf——美团点评分布式ID生成系统](!http://tech.meituan.com/MT_Leaf.html)  
[2] [高并发分布式系统中生成全局唯一Id汇总](!https://my.oschina.net/baiwasoft/blog/647394)  
[3] [Unique ID generation in distributed systems](!https://www.slideshare.net/davegardnerisme/unique-id-generation-in-distributed-systems)
