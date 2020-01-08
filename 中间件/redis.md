# redis
###简介

```
单线程、高性能、key-value数据库
1.支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用
2.支持多种数据结构，如string、list、set、zset、hash等。
3.支持数据的备份，即master-slave模式的数据备份
4.所有操作都是原子性的，要么成功执行要么失败完全不执行。单个操作是原子性的。多个操作也支持事务，即原子性，通过MULTI和EXEC指令包起来
5.支持 publish/subscribe, 通知, key 过期等等特性
...
```
###数据结构
* `String`


```
1.常用命令
SET key value                  //设置字符串键值
MSET key value [key value...]  //批量设置字符串键值
SETNX key value                //设置一个不存在的字符串键值（key存在就会设置失败）
GET key                        //获取字符串键值
MGET key [key...]              //批量获取键值
DEL key                        //删除key
EXPIRE key seconds             //设置key的过期时间
GETRANGE key start end          //返回key中字符串值的子字符
GETSET key value                //将给定 key 的值设为 value ，并返回 key 的旧值(old value)
INCR key                        //将 key 中储存的数字值+1,原子操作
INCRBY key increment            //将 key 所储存的值加上increment,只能是整数
INCRBYFLOAT key increment       //将 key 所储存的值加上increment,只能是浮点型
DECR key                        //将 key 中储存的数字值-1,原子操作
DECRBY key decrement            //将 key 所储存的值减去increment,只能是整数
APPEND key value                //如果 key 已经存在并且是一个字符串， APPEND 命令将指定的 value 追加到该 key 原来值（value）的末尾


2.数据结构
key - value
例如： set name zhangsan 


3.应用场景
    1.单值缓存 如：计数器、浏览量, 原子+1操作：INCR命名
    2.对象缓存 如：缓存对象json字符串
    3.简单的分布式锁 SETNX命令
    4.分布式系统全局序列号 INCRBY 命令 
        4.1 例如：INCR orderid ，原子+1操作，并发最多几万
        4.2 例如：INCRBY orderid 1000 ，每次生成1000个id，业务系统拿到这1000个id后自己在内部消化掉，完了再继续拿继续消化，提高并发能力
    5.全局session
```

* `hash`


```
1.常用命令
HSET key field value                    //存储一个哈希表key的键值
HSETNX key field value                  //存储一个不存在的哈希表key的键值（key存在就会设置失败）
HMSET key field value [field value...]  //存储一个哈希表key的多个键值
HGET key field                          //获取哈希表key对应的field键值
HMGET key field [field...]              //批量获取哈希表key中多个键值
HDEL key field [field...]               //删除哈希表key中的键值
HLEN key                                //返回哈希表key中键值的数量
HGETALL key                             //返回哈希表中key中的所有键值
HINCRBY key field increment             //为哈希表key中field的值加上increment，只能是整数
HINCRBYFLOAT key field increment        //为哈希表 key 中的指定字段的浮点数值加上增量 increment 
HKEYS keys                              //获取所有哈希表中的字段field
HEXISTS key field                       //查看key中指定字段field是否存在


2.数据结构
key -- field value | field value |field value ... 
例如：hset user 1:name zhangsan 1:age 12 

3.应用场景
    1.对象存储
4.优缺点
    1.同类数据归类整合存储，方便数据管理（管理同类对象存储）
    2.相比string操作消耗内存与cpu更小
    3.相比string更节省空间
    
    4.只能设置key的过期时间，field不行
    5.集群架构下不适合使用（redis会根据key计算出分片所在槽位，key相同会导致数据过于集中，性能下降）
```

* `list`


```
1.常用命令
LPUSH key value[value...]   //将一个或者多个value插入到列表的表头
LPUSHX key value            //将一个value插入到已存在的列表头部
RPUSH key value[value...]   //将一个或者多个value插入到列表的表尾
RPUSHX key value            //将一个value插入到已存在的列表尾部
LPOP key                    //移除并返回列表的头部元素
RPOP key                    //移除并返回列表的尾部元素
LRANGE key start stop       //获取指定区间内的元素，偏移量为：start stop
BLPOP key[key...] timeout   //从表头取出一个元素，若列表中没有元素，则阻塞等待timeout秒，如果timeout=0就会一直阻塞
BRPOP key[key...] timeout   //从表尾取出一个元素，若列表中没有元素，则阻塞等待timeout秒，如果timeout=0就会一直阻塞
RPOPLPUSH source destination //移除列表的最后一个元素，并将该元素添加到另一个列表并返回
BRPOPLPUSH source destination timeout   //从列表中取出一个值，将取出的元素插入到另外一个列表destination中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可取出元素为止。
LINDEX key index            //通过索引(从0开始从右至左)获取列表中的元素
LLEN key                    //获取列表长度
LREM key count value        //移除列表元素
    1.count > 0 : 从表头开始向表尾搜索，移除与 VALUE 相等的元素，数量为 COUNT 
    2.count < 0 : 从表尾开始向表头搜索，移除与 VALUE 相等的元素，数量为 COUNT 的绝对值
    3.count = 0 : 移除表中所有与 VALUE 相等的值
LSET key index value        //通过索引设置value
LTRIM key start stop        //保留指定区间内的元素，其他元素都会被移除


2.数据结构
key -  [value,value,value....] 
比如：lpush user lisi zhangsan wangwu
简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）

3.应用场景
    3.1 数据列表、消息列表等，如微博列表、微信公众号列表、商品列表等
    3.2 实现stack栈，先进后出，LPUSH+LPOP
    3.3 实现队列，先进先出，LPUSH+RPOP
    3.4 阻塞队列，先进先出, LPUSH+BRPOP
```

* `set`


```
1.常用命令
SADD key member[member...]              //存入元素，元素存在则忽略，反之新建
SREM key member[member...]              //删除元素
SMEMBERS key                            //获取所有元素
SCARD   key                             //获取元素个数
SISMEMBER key member                    //判断元素是否存在集合中
SRANDMEMBER KEY [count]                 //从集合中选择出count个元素且不从集合中删除
STOP key [count]                        //从集合中选择出count个元素且从集合中删除，不加count则移除并返回集合中的一个随机元素

SINTER key [key...]                     //交集运算，返回给定所有集合的交集
SINTERSTORE destination key [key...]    //将交集结果存入新集合destination中
SUNION key [key...]                     //并集运算
SUNIONSTORE destination key [key...]    //将并集结果存入新集合destination中
SDIFF key [key...]                      //差集运算
SDIFFSTORE destination key [key...]     //将差集结果存入新集合destination中

SMOVE source destination member         //将member元素从source集合移动到 destination集合】
SRANDMEMBER key [count]                 //获取集合中一个或多个随机数
SSCAN key cursor [MATCH pattern] [COUNT count]  //迭代集合中的元素

2.数据结构
key - [value , value ,value ]
set是string类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据

3.应用场景
    3.1 抽奖、点赞、关注、收藏
```

* `zset`

```
1.常用命令
ZADD key sroce value[sroce value...]        //添加一个或多个元素
ZCARD key                                   //获取元素个数
ZCOUNT key min max                          //计算在指定区间分数内的元素个数，偏移量为min max
ZRANK key member                            //返回有序集合中指定元素member的索引
ZRANGE key start stop [WHITHSCORE]          //通过索引(从0开始，-1表示最后一个元素索引)区间返回集合指定区间内的元素,默认按分数升序，带上WHITHSCORE会展示分数，下同
ZREVRANGE key start stop [WHITHSCORE]       //返回集合指定区间内的元素，按索引、分数 降序,带上WHITHSCORE会展示分数
ZINCRBY key increment member                //有序集合中对指定成员member的分数加上increment
ZINTERSTORE destination numkeys key [key ...]   //计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中      
ZRANGEBYLEX key min max [LIMIT offset count]    //通过字典区间返回有序集合的元素
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT]  //通过分数返回有序集合指定区间内的元素
ZREM key member [member ...]                //移除有序集合中的一个或多个成员
ZSCORE key member                           //返回元素分数
ZUNIONSTORE destination numkeys key [key ...]   //计算给定的一个或多个有序集的并集，并存储在新的 destination 中
ZSCAN key cursor [MATCH pattern] [COUNT count]  //迭代有序集合中的元素（包括元素成员和元素分值）


2.数据结构
key - [value:sroce , value:sroce ,value:sroce ] 
zset和set一样也是string类型元素的集合,且不允许重复的成员。
不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。
有序集合的成员是唯一的,但分数(score)却可以重复

3.应用场景
    3.1 排行榜等

```

###发布订阅

```
1.常用命令
PSUBSCRIBE pattern [pattern ...]            //订阅一个或多个符合给定模式的频道
PUBSUB subcommand [argument [argument ...]] //查看订阅与发布系统状态
PUBSUB channels                             //查看所有频道
PUBLISH channel message                     //将信息发送到指定的频道
PUNSUBSCRIBE [pattern [pattern ...]]        //退订所有给定模式的频道
SUBSCRIBE channel [channel ...]             //订阅给定的一个或多个频道的信息
UNSUBSCRIBE [channel [channel ...]]         //指退订给定的频道

例：
1.首先订阅频道
SUBSCRIBE test
//返回如下结果
Reading messages... (press Ctrl-C to quit)
1) "subscribe"  //类型
2) "test"       //频道名称 
3) (integer) 1 //返回1表示订阅成功

2.向频道内发送消息
PUBLISH test test-message
(integer) 1 //返回1 表示发送成功，如果向一个不存在的频道发送消息会返回0，表示发送失败
//订阅方会收到发送的消息
1) "message" //消息类型
2) "test"    //频道名称
3) "test-message" //消息内容
```

###多数据库

```
redis下，数据库是由一个索引标识，而不是数据库名称，默认情况下，一个客户端连接到数据库0
数据库总数是在redis配置文件下的databse参数控制的

常用命令
1.select index      //切换数据库
2.move key index    //将key移动到另一个数据库
3.flushdb           //清空当前数据库所有key
3.flushall          //清空整个redis的key
```

###事务

```
Redis 事务可以一次执行多个命令， 并且带有以下三个重要的保证：
1.批量操作在发送 EXEC 命令前被放入队列缓存。
2.收到 EXEC 命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行。
3.在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中。
也就是说，开始事务之后（multi），输入的命令都会依次放入命令队列，不会执行，直到输入exec提交事务后才会依次执行命令

一个事务从开始到执行会经历以下三个阶段：
1.开始事务。
2.命令入队。
3.执行事务。

常用命令
DISCARD     //取消事务，只能在multi后 ，exec前执行
EXEC        //执行所有事务块内的命令
MULTI       //标记一个事务块的开始
UNWATCH     //取消 WATCH 命令对所有 key 的监视
WATCH key [key ...] //监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断

应用场景：商品秒杀，转账等

```

###数据淘汰策略


```
如果redis内存不够时，会根据配置的缓存淘汰策略淘汰部分key，当没有淘汰策略或没有找到合适淘汰的key时，会返回out of memory错误

最大缓存配置   redis.conf 配置maxnmemory 

redis提供了8种淘汰策略:
1. volatile-lru：从已设置过期时间的keys中挑选最近最少使用的key淘汰
2. volatile-lfu：从已设置过期的keys中挑选一段时间内使用次数最少使用的key淘汰
3. volatile-ttl：从已设置过期时间的keys中挑选最近将要过期的key淘汰
4. volatile-random：从已设置过期时间的keys中随机挑选key淘汰
5. allkeys-lru：从keys中挑选最近最少使用的key淘汰
6. allkeys-lfu：从keys中挑选最近一段时间最少使用的key淘汰
7. allkeys-random：从keys中随机挑选key淘汰
8. no-enviction：默认配置，不淘汰任何key，如果内存满了还有写操作，会抛出内存溢出的错误


建议平时在使用redis过程中，尽量设置key的过期时间，主动剔除掉不活跃的旧数据，有助于性能提升

```
###持久化方式


```
redis提供了2种持久化方案

1.RDB 默认持久化配置
这种方式是将内存中的数据以快照的方式写入到二级制文件中，默认的文件名为dump.rdb
优点：保存、还原数据极快，适用于灾难备份
缺点：小内存机器不适合使用，只要符合要求就会进行快照操作，会占用机器内存
快照条件：
    1.服务器正常关闭，比如shutdown
    2.key满足一定条件，默认15分钟至少1个key发生变化，就会产生快照
        2.1 redis.conf中可配置此快照条件，更改:/save配置即可
        2.2 :/save配置格式：save xx秒 xx个key发生变化，如：save 90 1  表示90秒内至少1个key发生变化就会产生快照
        
2.AOF
快照方式是隔一段时间做一次的，如果期间意外宕机就会丢失最后一次快照后的所有修改，如果要求不能丢失任何修改的话，可采用AOF持久化方案

Append-only file:aof  redis会将每一个收到的写命令通过write函数追加到文件中(默认是appendonly.aof)，当redis重启时会通过执行文件中保存的命令将数据加载到内容中

有3种方式：
    1.appendfsync always：收到写命令就立即写入磁盘，最慢，但能保证完全的持久化
    2.appendfsync everysec：每秒钟写入磁盘一次，性能和持久化折中，默认是此配置
    3.appendfsync no：完全依赖操作系统，性能最好，持久化没保证
缺点：持久化的文件会越来越大，占用磁盘空间，还原数据比较慢，比如我们incr test 100次，文件中就会保存100条写命令，除了最后一条，其余99条都是多余的

```
###缓存与数据库一致性

```
1.实时同步
对强一致性要求比较高，可采用此方式，即在缓存中查询不到数据就从DB中查询并保存到保存中
注：更新DB后，建议直接删除缓存，再保存一份新的缓存，防止产生脏数据
     更新缓存时，建议先更新DB，再将缓存设置过期或删除此缓存，防止产生脏数据。
     
2.异步更新
通过异步的方式更新到DB中，比如rabbitmq kafka等。。
比如一件商品的浏览量，不可能每次+1都更新DB，可以1分钟或者一段时间更新一次即可。

3.定时任务（如果时效性要求不高）
```

###缓存雪崩

```
指在某个时刻某个key过期，突然有大量请求查询此key的数据，导致大量请求直接落到DB
1.缓存预热，如果知道此key快过期，可以手动提前预热一波数据
2.可以加锁让请求排队，先让一个线程从数据库取到数据放到缓存中后，其他线程直接从缓存取数据，但是这样可能会造成部分请求阻塞等待
3.缓存失效时间尽可能的均匀分布，比如不同类目下的数据可以视情况设置不同的过期时间，热点数据 缓存时间就长一点，反之时间就短一点
4.双缓存，可以用本地缓存来缓解可能发生的缓存雪崩问题，比如请求进来可以先访问本地缓存（本地缓存是长期有效的），本地缓存没有才会访问redis获取数据，redis没有再从DB查询出数据并且设置到双缓存中，这样的话，需要维护redis与本地缓存的一致性，会涉及到更多的复杂操作（比如分布式集群部署下，DB、redis怎么通知到本地缓存更新）
```
###缓存穿透

```
指查询一个一定不存在的key，首先从缓存中查询这个数据，查询不到则会从DB中查询，DB查询不到数据就会不写入缓存，那么每次查询这个不存在的key，请求都会落到DB，造成缓存穿透
解决方法：如果在DB层查询不到此数据，则在缓存中缓存一份空数据，下次请求进来查询此key的缓存是存在的，直接返回空，请求就不会落到DB层。
注：如果此key下次有新数据存入DB，要更新缓存数据（上述解决方法缓存此key空数据时设置过期时间也行）。不然请求永远落不到DB，也就永远查不到此key的数据了。
```

###高可用

```
1.主从复制
master---slave、slave、slave....
master处理写请求，slave处理读请求，提高并发量

主从配置方法：
    配置文件： 在从服务器的配置文件中加入：slaveof <masterip> <masterport>
    启动命令： redis-server启动命令后加入 --slaveof <masterip> <masterport>
    客户端命令： Redis服务器启动后，直接通过客户端执行命令：slaveof <masterip>
<masterport>，则该Redis实例成为从节点。
变回主：slaveof no one
变回从：slaveof ip port

查看主从信息命令：info replication 
优点：
    1.数据冗余：热备份数据
    2.故障恢复：从备份数据恢复
    3.读写分离：提高吞吐量
缺点：
    由于只有一个主节点，主节点挂掉后只能读不能写，会导致数据写不进来，只能临时将其中一个从节点升级为主节点


2.集群部署
redis3.0版本后支持redis-cluster集群，至少需要3master+3slave才能建立集群，采用无中心结构，每个节点保存数据和整个集群状态，每个节点都和其他节点连接
搭建方法自行查阅资料吧 太多了 懒得写，也写不明白
``

