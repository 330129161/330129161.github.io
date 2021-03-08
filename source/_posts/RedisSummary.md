title: redis总结

author: yingu

thumbnail: http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/cs/cs-20.jpg

toc: true 

tags:

  - redis
  - 工具

categories: 

  - [工具,redis] 

date: 2020-05-04 13:15:42

---

## 结构

### String

常用命令: `set,get,decr,incr,mget` 等。

特点：redis种最常用的一种数据结构

应用场景： 通过String的bitmap可以实现：布隆过滤器、AO权限、用户签到、活跃用户、用户在线状态。

通过自增、自减实现计数器、分布式锁

<!-- more -->

### Hash

常用命令：`hget,hset,hgetall `等

特点：存放的是结构化的对象，可以对对象中属性单独修改

应用场景：比较适合存储一些详情信息、例如购物车、对象

### List

常用命令：`lpush,rpush,lpop,rpop,lrange`等

特点：链表，按照插入顺序排序

应用场景：消息队列、最新列表、排行榜(定时计算)、利用lrange命令做分页

### Set

常用命令： `sadd,spop,smembers,sunion` 等

特点：不可重复、无序

应用场景：通过交集实现共同好友功能、通过差集实现推荐好友功能、去重

### zset(sorted set)

特点：可以通过权重参数score来对set中的元素进行排序

应用场景：排行榜(实时计算)、评论+动态分页

## 过期策略

### 定时删除

通过定时器监听所有key的过期，可以立即清除过期的key但是比较占用CPU资源，如果有大量的key需要清除，那么可能会影响数据的读写效率

### 惰性删除

当访问指定key时，才会检测key是否需要清除，如果大量过期的key没有被访问到，那么会占用大量内存

### 定期过期

每隔一定的时间，执行一次删除过期key操作。该策略是前两者的一个折中方案。通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同情况下使得CPU和内存资源达到最优的平衡效果。(expires字典会保存所有设置了过期时间的key的过期时间数据，其中，key是指向键空间中的某个键的指针，value是该键的毫秒精度的UNIX时间戳表示的过期时间。键空间是指该Redis集群中保存的所有键。)

**redis默认使用定期删除+惰性删除策略**：redis默认每100ms随机对redis中key进行检查，如果key过期那么删除，未删除的key在下一次使用时，会通过惰性删除策略检查并删除。如果用户一直不访问未被删除的key，那么最后过期的key会越来越多，那么此时就该采用**内存淘汰策略**了。

## 内存淘汰策略

### noeviction

当内存不足以容纳新写入数据时，新写入操作会报错。

### allkeys-lru

当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。

### allkeys-random

当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。

### volatile-lru

当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。

### volatile-random

当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。

### volatile-ttl

当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。

​		`redis`默认提供以上六种策略，用户可以通过`redis.conf`中的`maxmemory-policy`来配置，例如：`maxmemory-policy allkeys-lru`。如果没有设置 `expire` 的`key`, 那么 `volatile-lru`, `volatile-random` 和 `volatile-ttl` 策略的行为和 `noeviction`基本上一致。

## 缓存穿透、缓存击穿、缓存雪崩

### 缓存穿透

特点：去请求缓存中不存在的数据，导致所有的请求都打到数据库，从而造成数据库连接异常。

解决方案：

1. 布隆过滤器：内部维护数据库中所有数据的key，通过判断请求key是否有效，来决定是否需要去数据库中查询。
2. 异步更新：无论key是否存在都返回，随后另开一个线程异步去数据库中查询。

### 缓存击穿 

特点：缓存穿透的一种情况，当前一条数据失效。(单个key请求量较大)

解决方案：如果当前数据不是热点数据且访问量小那么不需要解决，如果为热点数据或访问量大，那么可以通过分布式锁来解决。

### 缓存雪崩

特点：缓存同一时间大面积的失效，这个时候又来了一波请求，结果请求都打到数据库上，从而导致数据库连接异常。

解决方案：给缓存的失效时间，加上一个随机值，避免集体失效。

## 持久化

### RDB

RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储。**redis默认开启**。

优点：恢复速度快、使用单独的子线程进行持久化，主进程不会进行任何IO操作

缺点：持久化之间发生故障后丢失的数据较多

参数配置：

```
#dbfilename：持久化数据存储在本地的文件
dbfilename dump.rdb
#dir：持久化数据存储在本地的路径，如
dir ./
##snapshot触发的时机，save     
#在900秒(15分钟)之后，如果至少有1个key发生变化，则dump内存快照。
save 900 1   
 #在300秒(5分钟)之后，如果至少有10个key发生变化，则dump内存快照。
save 300 10
#在60秒(1分钟)之后，如果至少有10000个key发生变化，则dump内存快照。
save 60 10000
##当snapshot时出现错误无法继续时，是否阻塞客户端"变更操作"，"错误"可能因为磁盘已满/磁盘故障/OS级别异常等  
stop-writes-on-bgsave-error yes  
##是否启用rdb文件压缩，默认为"yes"，压缩往往意味着"额外的cpu消耗"，同时也意味这较小的文件尺寸以及较短的网络传输时间  
rdbcompression yes 
```

### AOF

AOF持久化以日志的形式记录服务器所处理的每一个写、删除操作，查询操作不会记录，以文本的方式记录，可以打开文件看到详细的操作记录。

优点：持久化之间发生故障后丢失数据少

缺点：恢复速度较慢

```
##此选项为aof功能的开关，默认为"no"，可以通过"yes"来开启aof功能  
##只有在"yes"下，aof重写/文件同步等特性才会生效  
appendonly yes  
 
##指定aof文件名称  
appendfilename appendonly.aof  
 
##指定aof操作中文件同步策略，有三个合法值：always everysec no,默认为everysec  
appendfsync everysec  
##在aof-rewrite期间，appendfsync是否暂缓文件同步，"no"表示"不暂缓"，"yes"表示"暂缓"，默认为"no"  
no-appendfsync-on-rewrite no  
 
##aof文件rewrite触发的最小文件尺寸(mb,gb),只有大于此aof文件大于此尺寸是才会触发rewrite，默认"64mb"，建议"512mb"  
auto-aof-rewrite-min-size 64mb  
 
##相对于"上一次"rewrite，本次rewrite触发时aof文件应该增长的百分比。  
##每一次rewrite之后，redis都会记录下此时"新aof"文件的大小(例如A)，那么当aof文件增长到A*(1 + p)之后  
##触发下一次rewrite，每一次aof记录的添加，都会检测当前aof文件的尺寸。  
auto-aof-rewrite-percentage 100  

AOF 是文件操作，对于变更操作比较密集的 server，那么必将造成磁盘 IO 的负荷加重；此外 linux 对文件操作采取了“延迟写入”手段，即并非每次 write 操作都会触发实际磁盘操作，而是进入了 buffer 中，当 buffer 数据达到阀值时触发实际写入(也有其他时机)，这是 linux 对文件系统的优化，但是这却有可能带来隐患，如果 buffer 没有刷新到磁盘，此时物理机器失效(比如断电)，那么有可能导致最后一条或者多条 aof 记录的丢失。通过上述配置文件，可以得知 redis 提供了 3 中 aof 记录同步选项：
always：每一条 aof 记录都立即同步到文件，这是最安全的方式，也以为更多的磁盘操作和阻塞延迟，是 IO 开支较大。
everysec：每秒同步一次，性能和安全都比较中庸的方式，也是 redis 推荐的方式。如果遇到物理服务器故障，有可能导致最近一秒内 aof 记录丢失(可能为部分丢失)。
no：redis 并不直接调用文件同步，而是交给操作系统来处理，操作系统可以根据 buffer 填充情况 / 通道空闲时间等择机触发同步；这是一种普通的文件操作方式。性能较好，在物理服务器故障时，数据丢失量会因 OS 配置有关。
```

### 混合持久化

Redis 4.0 开始支持 rdb 和 aof 的混合持久化(默认关闭)。如果把混合持久化打开，`aof rewrite` 的时候就直接把 rdb 的内容写到 aof 文件开头。也就是说日志只用来做增量恢复， 而快照用来使用恢复。打开混合持久化的配置：

```
appendonly yes
aof-use-rdb-preamble yes  
```

## 集群

### 主从复制(Replication)

当你的业务需要对redis进行大量读请求，而你的redis服务器性能又不够时，就可以通过主从复制的方式对redis进行一个集群配置。

特点：

1. 读写分离：master提供读写服务，slave(可以为多个)提供读服务
2. master挂了后，无法自动重新选举master,那么此时无法提供写服务

Redis的主从复制可以根据是否全量分为全量同步和增量同步。

主从复制的过程：

​		`redis`的主从复制策略是通过其持久化的RDB文件来实现的，当配置好`Slave`后，`Slave`与`Master`建立连接，然后发送sync命令。无论是第一次连接还是重新连接，`Master`都会启动一个后台进 程，将数据库RDB快照保存到文件中，同时`Master`主进程会开始收集新的写命令并缓存。后台进程完成写文件后，`Master`就发送文件给  `Slave`，`Slave`将文件保存到硬盘上，再加载到内存中，接着`Master`就会把缓存的命令转发给`Slave`，后续`Master`将收到的写命令发送给 `Slave`。如果`Master`同时收到多个`Slave`发来的同步连接命令，`Master`只会启动一个进程来写数据库镜像，然后发送给所有的`Slave`。

`redis.conf`参数配置：

```
Master节点中配置读写分离 slave-read-only yes
Slave节点中配置 slaveof master_ip master_port 6379 #指定master的ip和端口
```

### 哨兵模式(Sentinel)

 Redis从2.8开始正式提供了`Redis Sentinel` 架构，`Sentinel`用于监控redis集群中`Master`主服务器工作的状态，当前`Master`发生故障后，可以实现Master和Slave服务器的切换。

工作方式：

1. 每个`Sentinel`以每秒钟一次的频率向它所知的`Master`，`Slave`以及其他 `Sentinel` 实例发送一个PING命令。
2. 如果一个实例（instance）距离最后一次有效回复PING命令的时间超过 `own-after-milliseconds` 选项所指定的值，则这个实例会被`Sentinel`标记为主观下线。 
3. 如果一个`Master`被标记为主观下线，则正在监视这个`Master`的所有 `Sentinel` 要以每秒一次的频率确认`Master`的确进入了主观下线状态。 
4. 当有足够数量的`Sentinel`（大于等于配置文件指定的值）在指定的时间范围内确认`Master`的确进入了主观下线状态，则`Master`会被标记为客观下线。
5. 在一般情况下，每个`Sentinel` 会以每10秒一次的频率向它已知的所有Master，`Slave`发送 INFO 命令。
6. 当Master被Sentinel标记为客观下线时，`Sentinel` 向下线的 `Master` 的所有`Slave`发送 INFO命令的频率会从10秒一次改为每秒一次。 
7. 若没有足够数量的`Sentinel`同意`Master`已经下线，Master的客观下线状态就会被移除。 若 `Master`重新向`Sentinel` 的PING命令返回有效回复，Master的主观下线状态就会被移除。

哨兵的主要功能

- 集群监控：负责监控 Redis master 和 slave 进程是否正常工作。

- 消息通知：如果某个 **Redis** 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。

- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。

- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

  

`sentinel.conf` 参数配置：

```
# Example sentinel.conf

# *** IMPORTANT ***
# 绑定IP地址
# bind 127.0.0.1 192.168.1.1
# 保护模式（是否禁止外部链接，除绑定的ip地址外）
# protected-mode no

# port <sentinel-port>
# 此Sentinel实例运行的端口
port 26379

# 默认情况下，Redis Sentinel不作为守护程序运行。 如果需要，可以设置为 yes。
daemonize no

# 启用守护进程运行后，Redis将在/var/run/redis-sentinel.pid中写入一个pid文件
pidfile /var/run/redis-sentinel.pid

# 指定日志文件名。 如果值为空，将强制Sentinel日志标准输出。守护进程下，如果使用标准输出进行日志记录，则日志将发送到/dev/null
logfile ""

# sentinel announce-ip <ip>
# sentinel announce-port <port>
#
# 上述两个配置指令在环境中非常有用，因为NAT可以通过非本地地址从外部访问Sentinel。
#
# 当提供announce-ip时，Sentinel将在通信中声明指定的IP地址，而不是像通常那样自动检测本地地址。
#
# 类似地，当提供announce-port 有效且非零时，Sentinel将宣布指定的TCP端口。
#
# 这两个选项不需要一起使用，如果只提供announce-ip，Sentinel将宣告指定的IP和“port”选项指定的服务器端口。
# 如果仅提供announce-port，Sentinel将通告自动检测到的本地IP和指定端口。
#
# Example:
#
# sentinel announce-ip 1.2.3.4

# dir <working-directory>
# 每个长时间运行的进程都应该有一个明确定义的工作目录。对于Redis Sentinel来说，/tmp就是自己的工作目录。
dir /tmp

# sentinel monitor <master-name> <ip> <redis-port> <quorum>
#
# 告诉Sentinel监听指定主节点，并且只有在至少<quorum>哨兵达成一致的情况下才会判断它 O_DOWN 状态。
#
#
# 副本是自动发现的，因此您无需指定副本。
# Sentinel本身将重写此配置文件，使用其他配置选项添加副本。另请注意，当副本升级为主副本时，将重写配置文件。
#
# 注意：主节点（master）名称不能包含特殊字符或空格。
# 有效字符可以是 A-z 0-9 和这三个字符 ".-_".
sentinel monitor mymaster 127.0.0.1 6379 2

# 如果redis配置了密码，那这里必须配置认证，否则不能自动切换
# Example:
#
# sentinel auth-pass mymaster MySUPER--secret-0123passw0rd

# sentinel down-after-milliseconds <master-name> <milliseconds>
#
# 主节点或副本在指定时间内没有回复PING，便认为该节点为主观下线 S_DOWN 状态。
#
# 默认是30秒
sentinel down-after-milliseconds mymaster 30000

# sentinel parallel-syncs <master-name> <numreplicas>
#
# 在故障转移期间，多少个副本节点进行数据同步
sentinel parallel-syncs mymaster 1

# sentinel failover-timeout <master-name> <milliseconds>
#
# 指定故障转移超时（以毫秒为单位）。 它以多种方式使用：
#
# - 在先前的故障转移之后重新启动故障转移所需的时间已由给定的Sentinel针对同一主服务器尝试，是故障转移超时的两倍。
#
# - 当一个slave从一个错误的master那里同步数据开始计算时间。直到slave被纠正为向正确的master那里同步数据时。
#
# - 取消已在进行但未生成任何配置更改的故障转移所需的时间
#
# - 当进行failover时，配置所有slaves指向新的master所需的最大时间。
#   即使过了这个超时，slaves依然会被正确配置为指向master。
#
# 默认3分钟
sentinel failover-timeout mymaster 180000

# 脚本执行
#
# sentinel notification-script和sentinel reconfig-script用于配置调用的脚本，以通知系统管理员或在故障转移后重新配置客户端。
# 脚本使用以下规则执行以进行错误处理：
#
# 如果脚本以“1”退出，则稍后重试执行（最多重试次数为当前设置的10次）。
#
# 如果脚本以“2”（或更高的值）退出，则不会重试执行。
#
# 如果脚本因为收到信号而终止，则行为与退出代码1相同。
#
# 脚本的最长运行时间为60秒。 达到此限制后，脚本将以SIGKILL终止，并重试执行。

# 通知脚本
#
# sentinel notification-script <master-name> <script-path>
#
# 为警告级别生成的任何Sentinel事件调用指定的通知脚本（例如-sdown，-odown等）。
# 此脚本应通过电子邮件，SMS或任何其他消息传递系统通知系统管理员 监控的Redis系统出了问题。
#
# 使用两个参数调用脚本：第一个是事件类型，第二个是事件描述。
#
# 该脚本必须存在且可执行，以便在提供此选项时启动sentinel。
#
# 举例:
#
# sentinel notification-script mymaster /var/redis/notify.sh

# 客户重新配置脚本
#
# sentinel client-reconfig-script <master-name> <script-path>
#
# 当主服务器因故障转移而变更时，可以调用脚本执行特定于应用程序的任务，以通知客户端，配置已更改且主服务器地址已经变更。
#
# 以下参数将传递给脚本：
#
# <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
#
# <state> 目前始终是故障转移 "failover"
# <role> 是 "leader" 或 "observer"
#
# 参数 from-ip, from-port, to-ip, to-port 用于传递主服务器的旧地址和所选副本的新地址。
#
# 举例:
#
# sentinel client-reconfig-script mymaster /var/redis/reconfig.sh

# 安全
# 避免脚本重置，默认值yes
# 默认情况下，SENTINEL SET将无法在运行时更改notification-script和client-reconfig-script。
# 这避免了一个简单的安全问题，客户端可以将脚本设置为任何内容并触发故障转移以便执行程序。
sentinel deny-scripts-reconfig yes

# REDIS命令重命名
#
#
# 在这种情况下，可以告诉Sentinel使用不同的命令名称而不是正常的命令名称。
# 例如，如果主“mymaster”和相关副本的“CONFIG”全部重命名为“GUESSME”，我可以使用：
#
# SENTINEL rename-command mymaster CONFIG GUESSME
#
# 设置此类配置后，每次Sentinel使用CONFIG时，它将使用GUESSME。 请注意，实际上不需要尊重命令案例，因此在上面的示例中写“config guessme”是相同的。
#
# SENTINEL SET也可用于在运行时执行此配置。
#
# 为了将命令设置回其原始名称（撤消重命名），可以将命令重命名为它自身：
#
# SENTINEL rename-command mymaster CONFIG CONFIG
```

### Redis Sharding(客户端分片)

redis 3.0以下使用**客户端sharding**分片技术，它是一主多备的实现方式。采用一致性Hash算法来实现数据的分片。通过将数据分散到多个Redis实例中，进而减轻单台redis实例的压力。

优点：服务端独立，降低了服务器集群的复杂性、减轻单台redis实例的压力、横向扩展可存储更多数据

缺点：需要实时获取集群节点的联系信息，动态添加节点需要客户端支持动态sharding，并且需要重启服务器

### Redis-Cluster(服务端分片)

redis 3.0推出的Redis-Cluster是基于Redis Sharding的一种集群方式。一个Redis Cluster由多个Redis节点组成。不同的节点组服务的数据无交集，每个节点对应数据sharding的一个分片。节点组内部分为主备2类，对应前面叙述的master和slave。两者数据准实时一致，通过异步化的主备复制机制保证。一个节点组有且仅有一个master，同时有0到多个slave。只有master对外提供写服务，读服务可由master/slave提供。

原理：redis cluster 默认分配了 16384(2^14) 个slot，当我们set一个key  时，会用CRC16算法来取模得到所属的slot，然后将这个key 分到哈希槽区间的节点上，具体算法就是：CRC16(key) %  16384。必须要3个以上的主节点，否则在创建集群时会失败。

- 为什么RedisCluster有16384个槽？CRC16算法产生的hash值有16bit，该算法可以产生2^16-=65536个值。换句话说，值是分布在0~65535之间。那作者在做mod运算的时候，为什么不mod65536，而选择mod16384？
  1. 因为节点在通讯的时候会携带消息头。如果槽位为65536，发送心跳信息的消息头达8k，发送的心跳包过于庞大。
  2. 一般情况下节点不会超过1000个，因此16384个槽位足够我们使用。
  3. 如果节点太少，而槽位有很多的情况下，压缩效率较低。

#### Redis Cluster原理

1. node1和node2首先进行握手meet，知道彼此的存在
2. 握手成功后，两个节点会定期发送ping/pong消息，交换数据信息(消息头，消息体)
3. 消息头里面有个字段：unsigned char myslots[CLUSTER_SLOTS/8]，每一位代表一个槽，如果该位是1，代表该槽属于这个节点
4. 消息体中会携带一定数量的其他节点的信息，大约占集群节点总数量的十分之一，至少是3个节点的信息。节点数量越多，消息体内容越大。
5. 每秒都在发送ping消息。每秒随机选取5个节点，找出最久没有通信的节点发送ping消息。
6. 每100毫秒都会扫描本地节点列表，如果发现节点最近一次接受pong消息的时间大于cluster-node-timeout/2,则立即发送ping消息

## 一致性hash算法

一致性hash算法是基于hash算法实现的，`Redis Sharding`使用了此算法。

原理：首先求出服务器的哈希值，并将其映射到0~2^32个槽的圆环上。不同的服务器经过hash计算后，分配到圆环上不同的槽。当用户存储数据时，也会通过计算数据键的哈希值的方式将数据映射到圆环上，接着从数据映射的位置开始顺时针查找，将数据保存到找到的第一个服务器上，如果找了一圈还没找到，那么最就会保存到第一台服务器上。

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/%E4%B8%80%E8%87%B4%E6%80%A7hash%E7%AE%97%E6%B3%95.png)

一致性hash算法的好处很明显，当需要新增节点时只会影响顺时针的下一个节点，但是这种方式也会带来一种问题，那么就是数据倾斜，如图：

![](http://yingu-blog.oss-cn-hangzhou.aliyuncs.com/%E4%B8%80%E8%87%B4%E6%80%A7hash%E7%AE%97%E6%B3%95%E5%80%BE%E6%96%9C.png)

从上面图就可以看出，当服务器分布不均时，可能会导致大量的数据存储到同一个数据库中，产生一个数据倾斜的问题，因此为了解决这种问题，一致性hash算法引入了虚拟节点机制，即对每一个服务器节点计算多个hash，将每个计算结果都放置一个此服务器节点，称为虚拟节点。实际应用中，通常将虚拟节点数设置为32甚至更大，因此即使很少的服务节点也能做到相对均匀的数据分布。

## 常见优化

1. Master最好不要做任何持久化工作，如RDB内存快照和AOF日志文件
2. 如果数据比较重要，某个Slave开启AOF备份数据，策略设置为每秒同步一次
3. 为了主从复制的速度和连接的稳定性，Master和Slave最好在同一个局域网内
4. 尽量避免在压力很大的主库上增加从库
5. 主从复制不要用图状结构，用单向链表结构更为稳定，即：Master <- Slave1 <- Slave2 <- Slave3...

## 常见问题

- 为什么redis采用单线程？

因为Redis是基于内存的操作，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。并且单线程更容易实现，不用考虑线程切换消耗性能的问题，也就不需要考虑线程安全问题。

redis使用多路复用技术，可以处理并发的连接。非阻塞IO 内部实现采用epoll，采用了epoll+自己实现的简单的事件框架。epoll中的读、写、关闭、连接都转化成了事件，然后利用epoll的多路复用特性，绝不在io上浪费一点时间。

- Redis如何做内存优化？

尽可能使用散列表（hashes），散列表（是说散列表里面存储的数少）使用的内存非常小，所以你应该尽可能的将你的数据模型抽象到一个散列表里面。

参考资料

https://www.jianshu.com/p/65765dd10671

https://blog.csdn.net/u010647035/article/details/90553596

https://www.cnblogs.com/williamjie/p/9477852.html

https://blog.csdn.net/fouy_yun/article/details/81590252

https://www.cnblogs.com/Eugene-Jin/p/10819601.html

https://blog.csdn.net/qq_41453285/article/details/106383157

​	





































