# Redis学习记录 #

## 一，了解Redis数据库 ##


学习目的：
1，通过操作常用命令熟悉并会使用redis数据库
2，了解hash，list，set，sortedset等数据结构
3，接触nosql数据库

### 1.Redis是什么 ###
 Redis 是一个高性能的key-value数据库，在部 分场合可以对关系数据库起到很好的补充作用。Redis支持主从同步。数据可以从主服务器向任意数量的从服务器上同步，从服务器可以是关联其他从服务器的主服务器。这使得Redis可执行单层树复制。存盘可以有意无意的对数据进行写操作。由于完全实现了发布/订阅机制，使得从数据库在任何地方同步树时，可订阅一个频道并接收主服务器完整的消息发布记录。同步对读取操作的可扩展性和数据冗余很有帮助。

### 2.Redis的优势 ###

1. Redis属于内存数据库，性能及高，且可以将内存中的数据保存在磁盘中以供下次使用。
2. Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
3. Redis的所有操作都是原子性的，同时Redis还支持对几个操作合并后的原子性执行。（事务）
4. Redis还支持 publish/subscribe, 通知, key 过期等众多特性。
5. Redis支持数据的备份，即master-slave模式的数据备份。

### 3.本次学习Redis使用的版本与客户端工具 ###

服务端下载地址：https://github.com/microsoftarchive/redis/releases/tag/win-3.2.100
客户端工具：Redis Desktop Manager



## 二，Redis常用数据与命令总结 ##

### 1.key命令 ###
- keys * 		--- 获取key，*表示所有key
- TYPE key 		--- 用来获取某key的类型
- del key1   	--- 删除key
- exists key    --- 判断是否存在key
- RANDOMKEY		--- 返回随机的一个key
- move key 1 	--- 将当前的数据库key移动到某个数据库,目标库有，则不能移动
- RENAME key newkey	--- 更改key的名称


### 2.string命令 ###
**最普通的键值对格式** 
 
- set key value --- 设置key
- get key    	--- 获取key的值
- mset key1 value1 key2 value2 key3 value3
- mget key1 key2 key3
- STRLEN key	--- 返回 key 所储存的字符串值的长度

### 3.哈希表命令 ###
**hash表比较类似数据库中表的一条记录**

- Hset key field value --- 设置哈希key
- Hget key field 	--- 获取哈希key的值
- HMSET key field1 value1 field2 value2 	--- 同时将多个(域-值)对设置到哈希表 key 中。
- HMGET key field1 field2		--- 获取多个给定域中的值
- HDEL key field1				--- 删除给定域中的值
- HLEN key						--- 获取哈希表中字段的数量

### 4.List列表命令 ###
**List是链表结构，可以在链表左，右两边分别操作**

- LPUSH key value1 [value2] 	--- 将一个或多个值插入到列表头部
- LPUSHX key value 				--- 将一个值插入到已存在的列表头部
- LPOP key 						--- 移出并获取列表的第一个元素
- LLEN key 						--- 获取列表长度
- LINDEX key index 				--- 通过索引获取列表中的元素

### 5.set集合命令 ###
**Set就是一堆不重复值的组合**

*因为 Redis 为集合提供了求交集、并集、差集等操作，你还可以使用不同的命令选择将结果返回给客户端还是存集到一个新的集合中

- SADD key member1 member2 	--- 向集合添加一个或多个成员
- SDIFF key1 key2			--- 返回给定所有集合的差集
- SINTER key1 key2 			--- 返回给定所有集合的交集
- SUNION key1 [key2] 		--- 返回所有给定集合的并集
- SISMEMBER key member 		--- 判断 member 元素是否是集合 key 的成员
- SMEMBERS key 				--- 返回集合中的所有成员

### 6.有序集合(sorted set)命令 ###
**zset是set的一个升级版本，他在set的基础上增加了一个顺序属性，这一属性在添加修改元素的时候可以指定，每次指定后，zset会自动重新按新的值调整顺序。**

- ZADD key score1 member1 score2 member2	--- 向有序集合添加成员，或更新已存在成员的分数
- ZCARD key 								--- 获取有序集合的成员数
- ZREMRANGEBYLEX key min max 				--- 移除有序集合中给定的字典区间的所有成员

###  7.其他redis命令 ###

- select 0 		--- 选择一个库
- flushdb      	--- 删除当前库的所有key
- BGSAVE		--- 在后台异步保存当前数据库的数据到磁盘
- TIME 			--- 返回当前服务器时间
- DBSIZE 		--- 返回当前数据库的 key 的数量


## 三，Redis常用功能总结 ##

### 1.发布订阅 ###

**定义：Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。**

示例：创建两个订阅者：

    gaoryrds:1>subscribe rdsread
    切换到推送/订阅模式，关闭标签页来停止接收信息。
     1)  "subscribe"
     2)  "rdsread"
     3)  "1"
    
    gaoryrds:1>subscribe rdsread
    切换到推送/订阅模式，关闭标签页来停止接收信息。
     1)  "subscribe"
     2)  "rdsread"
     3)  "1"

然后在另一个频道使用rdsread发布消息：

    gaoryrds:2>PUBLISH rdsread "risagraet"

所有订阅者频道会显示发布的内容：

    gaoryrds:1>subscribe rdsread
    切换到推送/订阅模式，关闭标签页来停止接收信息。
     1)  "subscribe"
     2)  "rdsread"
     3)  "1"
     1)  "message"
     2)  "rdsread"
     3)  "risagraet"


- PUBLISH channel message 		--- 将信息发送到指定的频道。
- SUBSCRIBE channel 	 		--- 订阅给定的一个或多个频道的信息。
- UNSUBSCRIBE channel 	 		--- 指退订给定的频道。


### 2.事务 ###

**redis中事务的主要作用是对多条命令的打包执行**

Redis 事务可以一次执行多个命令，并且带有以下三个重要的保证：

- 批量操作在发送 EXEC 命令前被放入队列缓存。
- 收到 EXEC 命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行。
- 在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中。

一个事务从开始到执行会经历以下三个阶段：开始事务，命令入队，执行事务。
示例：
    gaoryrds:2>MULTI
    "OK"
    gaoryrds:2>set key1 value1
    "QUEUED"
    gaoryrds:2>set key2 value2
    "QUEUED"
    gaoryrds:2>EXEC
     1)  "OK"
     2)  "OK"

- DISCARD 		--- 取消事务，放弃执行事务块内的所有命令。
- EXEC 			--- 执行所有事务块内的命令。
- MULTI 		--- 标记一个事务块的开始。
- UNWATCH 		--- 取消 WATCH 命令对所有 key 的监视。
- WATCH key key1...		--- 监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。


### 3.脚本 ###

**Redis 2.6 版本通过内嵌支持 Lua 环境。执行脚本的常用命令为 EVAL。**

- EVAL script num key key1 ... arg arg1 ... --- 执行 Lua 脚本。
	
    1. eval是执行命令
    2. script是执行的lua脚本或者lua脚本的地址
    3. num是需要键的个数
    4. key key1 ... arg arg1 ...键值的具体内容

- SCRIPT EXISTS script script ... 		--- 查看脚本是否已被保存在缓存中。
- SCRIPT FLUSH 							--- 从脚本缓存中移除所有脚本。
- SCRIPT KILL 							--- 杀死当前正在运行的 Lua 脚本。


### 4.Redis备份与还原 ###

**备份redis库：可以使用SAVE或BGSAVE命令备份当前库，将会在安装目录下创建dump.rdb文件**

**还原redis库：因无法使用教程中重启服务自动还原备份文件，所以需要借助客户端工具还原dump.rdb文件**

![](D:\workFile\SVN\Document\学习帮助文档\DB\jietu.png)