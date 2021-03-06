#redis
##初识
###定义
Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。它通常被称为数据结构服务器，因为值（value）可以是字符串(String), 哈希(Map),列表(list),集合(sets)和有序集合(sorted sets)等类型。
###特点
####优点
* 速度快，由于是全内存操作，所以读写性能很好，读可以达到10w/s的频率，写速度8k/s

* 支持丰富数据类型，支持string，list，set，sorted set，hash


* 支持事务，操作都是原子性，所谓的原子性就是对数据的更改要么全部执行，要么全部不执行；同时Redis还支持对几个操作合并后的原子性执行。


* 支持持久化存储，作为一个内存数据库，最担心的，就是万一机器死机，数据会消失掉。redi使用rdb和aof做数据的持久化存储。主从数据同时，生成rdb文件，并利用缓冲区添加新的数据更新操作做对应的同步。
* 丰富的特性：可用于缓存，消息，按key设置过期时间，过期后将会自动删除
####缺点
redis的缺点主要如下：
* 由于是内存数据库，所以单台机器，存储的数据量，跟机器本身的内存大小。虽然redis本身有key过期策略，但是还是需要提前预估和节约内存。如果内存增长过快，需要定期删除数据。因此Redis适合的场景主要局限在较小数据量的高性能操作和运算上
   
* 如果进行完整重同步，由于需要生成rdb文件，并进行传输，会占用主机的CPU，并会消耗现网的带宽。不过redis2.8版本，已经有部分重同步的功能，但是还是有可能有完整重同步的。比如，新上线的备机。
* 修改配置文件，进行重启，将硬盘中的数据加载进内存，时间比较久。在这个过程中，redis不能提供服务。

###应用场景 
####计数器  
可以对 String 进行自增自减运算，从而实现计数器功能。Redis 这种内存型数据库的读写性能非常高，很适合存储频繁读写的计数量。
####缓存  
将热点数据放到内存中，设置内存的最大使用量以及淘汰策略来保证缓存的命中率。
####查找表
例如DNS记录就很适合使用Redis进行存储。查找表和缓存类似，也是利用了 Redis 快速的查找特性。但是查找表的内容不能失效，而缓存的内容可以失效，因为缓存不作为可靠的数据来源。
####消息队列
List是一个双向链表可以通过lpop和lpush写入和读取消息。不过最好使用Kafka、RabbitMQ等消息中间件。
####会话缓存
可以使用 Redis 来统一存储多台应用服务器的会话信息。当应用服务器不再存储用户的会话信息，也就不再具有状态，一个用户可以请求任意一个应用服务器，从而更容易实现高可用性以及可伸缩性。
####分布式锁实现
在分布式场景下，无法使用单机环境下的锁来对多个节点上的进程进行同步。可以使用 Reids 自带的 SETNX 命令实现分布式锁，除此之外，还可以使用官方提供的 RedLock 分布式锁实现。
####可以发布、订阅的实时消息系统
Redis中Pub/Sub系统可以构建实时的消息系统，比如，很多使用Pub/Sub构建的实时聊天应用。

设计思路：服务端发送消息（含标题，内容），标题按照一定规则存入redis，同时标题（以最少的信息量）推送到客户端，客户点击标题时，获取相应的内容阅读.如果未读取，可以提示多少条未读，redis能够很快记数
根据一定时间清理缓存	
####其它
Set 可以实现交集、并集等操作，从而实现共同好友等功能。
ZSet 可以实现有序性操作，从而实现排行榜等功能。
	
####Redis常见的性能问题都有哪些？如何解决？
* Master写内存快照，save命令调度rdbSave函数，会阻塞主线程的工作，当快照比较大时对性能影响是非常大的，会间断性暂停服务，所以Master最好不要写内存快照。

* Master AOF持久化，如果不重写AOF文件，这个持久化方式对性能的影响是最小的，但是AOF文件会不断增大，AOF文件过大会影响Master重启的恢复速度。Master最好不要做任何持久化工作，包括内存快照和AOF日志文件，特别是不要启用内存快照做持久化,如果数据比较关键，某个Slave开启AOF备份数据，策略为每秒同步一次。

* Master调用BGREWRITEAOF重写AOF文件，AOF在重写的时候会占大量的CPU和内存资源，导致服务load过高，出现短暂服务暂停现象。

* Redis主从复制的性能问题，为了主从复制的速度和连接的稳定性，Slave和Master最好在同一个局域网内

##redis支持的数据类型都有哪些
###String
这个其实没啥好说的，最常规的set/get操作，value可以是String也可以是数字。一般做一些复杂的计数功能的缓存。
###hash
这里value存放的是结构化的对象，比较方便的就是操作其中的某个字段。博主在做单点登录的时候，就是用这种数据结构存储用户信息，以cookieId作为key，设置30分钟为缓存过期时间，能很好的模拟出类似session的效果。
###list
使用List的数据结构，可以做简单的消息队列的功能。另外还有一个就是，可以利用lrange命令，做基于redis的分页功能，性能极佳，用户体验好。
###set
因为set堆放的是一堆不重复值的集合。所以可以做全局去重的功能。为什么不用JVM自带的Set进行去重？因为我们的系统一般都是集群部署，使用JVM自带的Set，比较麻烦，难道为了一个做一个全局去重，再起一个公共服务，太麻烦了。另外，就是利用交集、并集、差集等操作，可以计算共同喜好，全部的喜好，自己独有的喜好等功能。
###sorted set
sorted set多了一个权重参数score,集合中的元素能够按score进行排列。可以做排行榜应用，取TOP N操作
##redis对象
Redis 使用对象来表示数据库中的键和值， 每次当我们在 Redis 的数据库中新创建一个键值对时， 我们至少会创建两个对象， 一个对象用作键值对的键（键对象）， 另一个对象用作键值对的值（值对象）。

    typedef struct redisObject {

    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层实现数据结构的指针
    void *ptr;
    // ...
} robj;

Redis 中的每个对象都由一个 redisObject 结构表示， 该结构中和保存数据有关的三个属性分别是 type 属性、 encoding 属性和 ptr 属性。

对象的 type 属性记录了对象的类型， 这个属性的值可以是表中列出的常量的其中一个。

| 类型常量 | 对象的名字 |
| ------ | ------ |
| REDIS_STRINGN | 字符串对象 | 
| REDIS_LIST | 列表对象 |
| REDIS_HASH | 哈希对象 |
| REDIS_SET | 集合对象 |
| REDIS_ZSET | 有序集合对象 |


不同的对象有不同的编码方式，如下表：

| 类型	| 编码	| 对象 |
| ------ | ------ | ------ |
| REDIS_STRING |	REDIS_ENCODING_INT	| 使用整数值实现的字符串对象 |
| REDIS_STRING	| REDIS_ENCODING_EMBSTR	 | 使用embstr编码的简单动态字符串实现的字符串对象 |
| REDIS_STRING	| REDIS_ENCODING_RAW | 使用简单动态字符串实现的字符串对象 | 
| REDIS_LIST	| REDIS_ENCODING_ZIPLIST | 使用压缩列表实现的列表对象 |
| REDIS_LIST	| REDIS_ENCODING_LINKEDLIST | 使用双端链表实现的列表对象 | 
| REDIS_HASH	| REDIS_ENCODING_ZIPLIST | 使用压缩列表实现的哈希对象 | 
| REDIS_HASH	| REDIS_ENCODING_HT	| 使用字典实现的哈希对象 | 
| REDIS_SET | REDIS_ENCODING_INTSET | 使用整数集合实现的集合对象 | 
| REDIS_SET | REDIS_ENCODING_HT | 使用字典实现的集合对象| 
| REDIS_ZSET	| REDIS_ENCODING_ZIPLIST | 使用压缩列表实现的有序集合对象 | 
| REDIS_ZSET | REDIS_ENCODING_SKIPLIST | 使用跳跃表和字典实现的有序集合对象 |

###字符串对象
字符串对象的编码可以是 int 、 raw 或者 embstr。
####编码转换
int编码的字符串对象和embstr编码的字符串对象在条件满足的情况下，会被转换为raw编码的字符串对象。对于int编码的字符串对象来说，如果我们向对象执行了一些命令，使得这个对象保存的不再是整数值，而是一个字符串值，那么字符串对象的编码将从int变为raw。
###列表对象
列表对象的编码可以是 ziplist 或者 linkedlist。
####编码转换
当列表对象可以同时满足以下两个条件时， 列表对象使用 ziplist 编码：

* 列表对象保存的所有字符串元素的长度都小于 64 字节；
* 列表对象保存的元素数量小于 512 个；

不能满足这两个条件的列表对象需要使用 linkedlist 编码。
###哈希对象
哈希对象的编码可以是ziplist或者hashtable。

ziplist编码的哈希对象使用压缩列表作为底层实现，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的压缩列表节点推入到压缩列表表尾，然后再将保存了值的压缩列表节点推入到压缩列表表尾，因此：

* 保存了同一键值对的两个节点总是紧挨在一起，保存键的节点在前，保存值的节点在后；
* 先添加到哈希对象中的键值对会被放在压缩列表的表头方向，而后来添加到哈希对象中的键值对会被放在压缩列表的表尾方向；

####编码转换
当哈希对象可以同时满足以下两个条件时，哈希对象使用ziplist编码：

* 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节；
* 哈希对象保存的键值对数量小于 512 个；

不能满足这两个条件的哈希对象需要使用hashtable编码。
###集合对象
集合对象的编码可以是intset或者hashtable。
intset编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合里面。
####编码转换
当集合对象可以同时满足以下两个条件时，对象使用intset编码：

* 集合对象保存的所有元素都是整数值；
* 集合对象保存的元素数量不超过512个；

不能满足这两个条件的集合对象需要使用hashtable编码。
###有序集合
有序集合的编码可以是ziplist或者skiplist。

ziplist编码的有序集合对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员（member），而第二个元素则保存元素的分值（score）。

压缩列表内的集合元素按分值从小到大进行排序， 分值较小的元素被放置在靠近表头的方向， 而分值较大的元素则被放置在靠近表尾的方向。
####编码转换
当有序集合对象可以同时满足以下两个条件时，对象使用ziplist编码：

* 有序集合保存的元素数量小于128个；
* 有序集合保存的所有元素成员的长度都小于64字节；

不能满足以上两个条件的有序集合对象将使用 skiplist 编码。
##reids持久化
由于Redis的数据都存放在内存中，如果没有配置持久化，redis重启后数据就全丢失了，于是需要开启redis的持久化功能，将数据保存到磁盘上，当redis重启后，可以从磁盘中恢复数据。redis提供两种方式进行持久化，一种是RDB持久化（原理是将Reids在内存中的数据库记录定时dump到磁盘上的RDB持久化），另外一种是AOF（append only file）持久化（原理是将Reids的操作日志以追加的方式写入文件）
###RDB
RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储。
![](/Users/xuchenhui/Documents/Learning/Blog/redis/RDB.png)
###AOF（append only file）
AOF持久化以日志的形式记录服务器所处理的每一个写、删除操作，查询操作不会记录，以文本的方式记录，可以打开文件看到详细的操作记录。
![](/Users/xuchenhui/Documents/Learning/Blog/redis/AOF.png)
####RDB的优点
RDB是一个非常紧凑（compact）的文件，它保存了Redis在某个时间点上的数据集。这种文件非常适合用于进行备份：比如说，你可以在最近的24 小时内，每小时备份一次RDB文件，并且在每个月的每一天，也备份一个 RDB文件。这样的话，即使遇上问题，也可以随时将数据集还原到不同的版本。RDB 非常适用于灾难恢复（disaster recovery）：它只有一个文件，并且内容都非常紧凑，可以（在加密后）将它传送到别的数据中心，或者亚马逊S3中。RDB可以最大化Redis的性能：父进程在保存RDB文件时唯一要做的就是fork出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘I/O操作。RDB在恢复大数据集时的速度比 AOF的恢复速度要快。
####RDB的缺点
如果你需要尽量避免在服务器故障时丢失数据，那么RDB不适合你。虽然 Redis允许你设置不同的保存点（save point）来控制保存RDB文件的频率，但是，因为RDB文件需要保存整个数据集的状态，所以它并不是一个轻松的操作。因此你可能会至少5分钟才保存一次RDB文件。在这种情况下， 一旦发生故障停机，你就可能会丢失好几分钟的数据。每次保存RDB的时候，Redis都要fork()出一个子进程，并由子进程来进行实际的持久化工作.在数据集比较庞大时，fork()可能会非常耗时，造成服务器在某某毫秒内停止处理客户端；如果数据集非常巨大，并且CPU时间非常紧张的话，那么这种停止时间甚至可能会长达整整一秒。虽然AOF重写也需要进行fork()，但无论AOF重写的执行间隔有多长，数据的耐久性都不会有任何损失。
####AOF的优点
使用AOF持久化会让Redis变得非常耐久（much more durable）：你可以设置不同的fsync策略，比如无fsync，每秒钟一次fsync，或者每次执行写入命令时fsync。AOF的默认策略为每秒钟fsync一次，在这种配置下，Redis仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据（fsync会在后台线程执行，所以主线程可以继续努力地处理命令请求）。AOF文件是一个只进行追加操作的日志文件（append only log），因此对AOF文件的写入不需要进行seek，即使日志因为某些原因而包含了未写入完整的命令（比如写入时磁盘已满，写入中途停机，等等），redis-check-aof工具也可以轻易地修复这种问题。Redis可以在 AOF文件体积变得过大时，自动地在后台对AOF进行重写：重写后的新AOF 文件包含了恢复当前数据集所需的最小命令集合。整个重写操作是绝对安全的，因为Redis在创建新AOF文件的过程中，会继续将命令追加到现有的 AOF文件里面，即使重写过程中发生停机，现有的AOF文件也不会丢失。 而一旦新AOF文件创建完毕，Redis就会从旧AOF文件切换到新AOF文件，并开始对新AOF文件进行追加操作。AOF文件有序地保存了对数据库执行的所有写入操作，这些写入操作以Redis协议的格式保存，因此AOF文件的内容非常容易被人读懂，对文件进行分析（parse）也很轻松。导出（export） AOF文件也非常简单：举个例子，如果你不小心执行了FLUSHALL命令，但只要AOF文件未被重写，那么只要停止服务器，移除AOF文件末尾的FLUSHALL 命令，并重启 Redis，就可以将数据集恢复到FLUSHALL执行之前的状态。
####AOF的缺点
对于相同的数据集来说，AOF文件的体积通常要大于RDB文件的体积。根据所使用的fsync策略，AOF的速度可能会慢于RDB。在一般情况下，每秒fsync 的性能依然非常高，而关闭fsync可以让AOF的速度和RDB一样快，即使在高负荷之下也是如此。不过在处理巨大的写入载入时，RDB可以提供更有保证的最大延迟时间（latency）。AOF在过去曾经发生过这样的bug：因为个别命令的原因，导致 AOF文件在重新载入时，无法将数据集恢复成保存时的原样。（举个例子，阻塞命令BRPOPLPUSH就曾经引起过这样的bug。）测试套件里为这种情况添加了测试：它们会自动生成随机的、复杂的数据集， 并通过重新载入这些数据来确保一切正常。虽然这种bug在AOF文件中并不常见，但是对比来说，RDB几乎是不可能出现这种bug的。
####RDB和AOF应该用哪一个
一般来说,如果想达到足以媲美PostgreSQL的数据安全性，你应该同时使用两种持久化功能。如果你非常关心你的数据,但仍然可以承受数分钟以内的数据丢失，那么你可以只使用RDB持久化。有很多用户都只使用AOF持久化， 但我们并不推荐这种方式：因为定时生成RDB快照（snapshot）非常便于进行数据库备份，并且RDB恢复数据集的速度也要比AOF恢复的速度要快，除此之外，使用RDB还可以避免之前提到的AOF程序的bug。因为以上提到的种种原因，未来我们可能会将AOF和RDB整合成单个持久化模型。

##redis有哪些架构模式
###单机版
![](/Users/xuchenhui/Documents/Learning/Blog/redis/单机版.png)
特点：简单
问题：
1、内存容量有限 2、处理能力有限 3、无法高可用。
###主从复制
![](/Users/xuchenhui/Documents/Learning/Blog/redis/Snip20181003_13.png)
Redis 的复制（replication）功能允许用户根据一个 Redis 服务器来创建任意多个该服务器的复制品，其中被复制的服务器为主服务器（master），而通过复制创建出来的服务器复制品则为从服务器（slave）。只要主从服务器之间的网络连接正常，主从服务器两者会具有相同的数据，主服务器就会一直将发生在自己身上的数据更新同步给从服务器，从而一直保证主从服务器的数据相同。
特点：
master/slave 角色
master/slave 数据相同
降低 master 读压力在转交从库
问题：无法保证高可用，没有解决master写的压力
###哨兵
![](/Users/xuchenhui/Documents/Learning/Blog/redis/哨兵.png)
Redis sentinel 是一个分布式系统中监控 redis 主从服务器，并在主服务器下线时自动进行故障转移。其中三个特性：

* 监控（Monitoring）：Sentinel会不断地检查你的主服务器和从服务器是否运作正常。
* 提醒（Notification）：当被监控的某个Redis服务器出现问题时， Sentinel可以通过API向管理员或者其他应用程序发送通知。
* 自动故障迁移（Automatic failover）：当一个主服务器不能正常工作时，Sentinel会开始一次自动故障迁移操作。

特点：
保证高可用，监控各个节点，自动故障迁移 

缺点：主从模式，切换需要时间丢数据，没有解决 master 写的压力
###集群（proxy）
![](/Users/xuchenhui/Documents/Learning/Blog/redis/代理型集群.png)
Twemproxy是一个Twitter开源的一个redis和memcache快速/轻量级代理服务器；Twemproxy是一个快速的单线程代理程序，支持Memcached ASCII协议和redis协议。

特点：

* 多种hash算法：MD5、CRC16、CRC32、CRC32a、hsieh、murmur、Jenkins

* 支持失败节点自动删除
* 后端 Sharding 分片逻辑对业务透明，业务方的读写方式和操作单个 Redis一致


缺点：

* 增加了新的 proxy，需要维护其高可用。
*failover 逻辑需要自己实现，其本身不能支持故障的自动转移可扩展性差，进行扩缩容都需要手动干预

###集群（直连型）
![](/Users/xuchenhui/Documents/Learning/Blog/redis/直连集群.png)
从redis 3.0之后版本支持redis-cluster集群，Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有节点连接。

特点：

* 无中心架构（不存在哪个节点影响性能瓶颈），少了 proxy 层。
* 数据按照 slot 存储分布在多个节点，节点间数据共享，可动态调整数据分布。
* 可扩展性，可线性扩展到 1000 个节点，节点可动态添加或删除。
* 高可用性，部分节点不可用时，集群仍可用。通过增加 Slave 做备份数据副本
* 实现故障自动 failover，节点之间通过 gossip 协议交换状态信息，用投票机制完成 Slave到 Master的角色提升。


缺点：

* 资源隔离性较差，容易出现相互影响的情况。
* 数据通过异步复制,不保证数据的强一致性

##redis数据淘汰策略
Redis提供了下面几种淘汰策略供用户选择，其中默认的策略为noeviction策略：
  
* noeviction：当内存使用达到阈值的时候，所有引起申请内存的命令会报错。
* allkeys-lru：在主键空间中，优先移除最近未使用的key。
* volatile-lru：在设置了过期时间的键空间中，优先移除最近未使用的key。
* allkeys-random：在主键空间中，随机移除某个key。
* volatile-random：在设置了过期时间的键空间中，随机移除某个key。
* volatile-ttl：在设置了过期时间的键空间中，具有更早过期时间的key优先移除。

##redis与其它类型数据库的不同


Reference：

[https://github.com/CyC2018/CS-Notes/blob/master/notes/Redis.md](https://github.com/CyC2018/CS-Notes/blob/master/notes/Redis.md)

[https://www.cnblogs.com/chenliangcl/p/7240350.html]() 

[http://www.imooc.com/article/74478]()

[https://www.cnblogs.com/rjzheng/p/9096228.html]()