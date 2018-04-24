---
date: 2015-04-03 00:30:00
layout: post
title: Redis学习笔记
categories:
- Redis
---
现在NoSQL也是特别热门的技术关键词了，近来整理了一些Redis的东西

## 数据对象与键空间 
Redis支持的数据类型比较多： 字符串，列表，哈希表，集合，有序集合

```python
# 对字符串进行操作
redis> set message "hello world"
# 对列表进行操作
redis> rpush alphabet "a" "b" "c"
# 对哈希表进行操作
redis> hset book name "Redis in action"
redis> hset book author "Josiah L. Carlson"
redis> hset book publisher "Manning"
```

像上面的操作，在Redis的某个键空间里有三个键值分别是message, alphabet, book，指针分别指向不同的存储对象

而在book所指向的哈希表对象里面，又有它自己的键空间: name, author, publisher 指向字符串对象

在哈希表对象的键值，可以嵌套其他的数据对象，列表，集合，甚至再嵌套一个哈希表在里面

## 过期键的处理

在Redis里面可以对存储对象设置一个过期时间，当超过一定时间后，Redis会将这个键从数据库中自动删除

```python
# 设置10秒的过期时间
127.0.0.1:6379> expire msg 10
(integer) 1
# 进行查询
127.0.0.1:6379> get msg
"wait expired"
# 等待几秒后再次查询
127.0.0.1:6379> get msg
(nil)
```

## RDB对过期键的处理策略

**生成RDB文件时对过期键的处理:**

执行SAVE或者BGSAVE操作时，会将数据库保存至RDB文件中

Redis会对内存中的Key进行检查，已经过期的Key不会被保存到RDB文件中

**载入RDB文件时对过期键的处理：**

如果以主模式载入，则过期键不会被载入到内存数据库中

如果以从模式载入，过期键先会被载入到内存数据库中，而后主从同步时才对过期键进行清空

## AOF对过期键的处理

当过期键被删除后，向AOF追加一条DEL命令用于标志已删除

如果客户端通过GET message访问一个过期键

1.服务器先删除message键

2.追加一条DEL message到AOF文件

3.返回空给客户端

和RDB相似，重写时会先对内存中的Key进行检查以确认是否写入AOF文件

## RDB持久化方式
默认保存在dump.rdb文件中(/var/lib/redis/dump.rdb)
可以根据配置文件修改
dbfilename dump.rdb

**SAVE和BGSAVE**

SAVE: 由主进程直接写RDB，直到文件创建完毕(主进程阻塞)

BGSAVE: 后台保存, 执行一次fork，令子进程写RDB文件

这两种保存方式都可理解为是服务器被动保存，也就是服务器在接受到客户端发来的上面两条命令之后进行保存

**自动保存的实现**

在/etc/redis/redis.conf配置文件中定义保存条件

```python
save <seconds> <changes>
save 900 1
save 300 10
save 60 10000
```

在seconds时间内 如果发生changes以上次修改时进行自动保存

```
struct redisServer {
    long long dirty;
    ……
};
```

在源码实现中，使用redisServer定义了一个全局变量server，其中有个dirty变量是用来记录修改次数的

那么在实现具体数据结构时，在操作改写存储对象的命令里面，就需要对dirty这个变量进行修改了，通常都是
执行server.dirty++的操作来进行，但是如果是对列表插入了两个元素这种情况，就要执行server.dirty+=n了

在源码实现中，serverCron是一个时间事件处理器(100ms执行一次)，在这里面就需要对配种中定义的保存条件
进行检查操作，如果满足条件时，Redis将创建一个子进程进行保存操作，在执行保存期间，如果客户端发过来保存
命令的话，服务器先检查到有一个子进程在进行保存操作，就不会处理这条命令了

## AOF持久化方式

相关的配置选项

```python
#是否设置为AOF持久化方式
appendonly yes
#默认的文件名
appendfilename "appendonly.aof"
#将aof_buf缓冲区中的所有内容写入并同步到 AOF 文件
appendfsync always
#将aof_buf缓冲区中的所有内容写入到 AOF 文件, 每秒进行同步
appendfsync everysec 
#将aof_buf缓冲区中的所有内容写入到 AOF 文件, 由操作系统决定同步
appendfsync no 
```

aof_buf是一个缓冲区，如果工作在AOF模式下，服务器执行命令时会将该命令先写入缓冲区中，如果缓冲区满了
就将缓冲区的数据写入到AOF文件中，此时再清空缓冲区，继续执行其他命令，如此下去…

由于AOF根据每条命令生成存储记录，因而一个某个数据被重复修改时造成了数据冗余,其中一种
解决方案时定期根据内存中的值进行重写，同时也将过期键过滤掉，这一操作由后台进程运行

**AOF重写**

为了避免重写AOF主进程阻塞的情况，AOF将这一操作放在后台运行

1.创建子进程重写AOF，同时创建AOF缓存

2.子进程写AOF，主进程继续执行请求，同时写入AOF缓存

3.主进程检查子进程状态，子进程写结束后，主进程将AOF缓存追加到AOF文件中

**AOF重写的触发条件**

```python
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
#min-size 表示最小大小，当AOF文件大于这个值时进行第一次重写操作
#percentage表示增量(默认100)，当AOF文件增量超过这个值时触发一次重写操作
```

## 复制
Redis服务器可以复制一个主服务器作为备份的从服务器，复制后的从服务器对客户端而言是只读的
(实际上它只接受主服务器通过命令传播发过来的写操作)

通过下面的一条命令即可完成复制
```python
slaveof ip port
```

同步：将主服务器数据库状态更新到从服务器

命令传播：将主服务器的变更操作更新到从服务器(解决数据不一致问题)

**同步过程大概可以描述如下**

1.从服务器发送同步命令(SYNC)给主服务器

2.主服务器创建子进程发送RDB文件给从服务器，同时继续执行其他命令(保存在缓冲区)

3.主服务器将缓冲区中的所有写命令发送给从服务器

这一过程非常类似于AOF重写的过程

在完成同步后，其他到操作就通过命令传播来进行，就是当主服务器发生变更时，也需要将该条
写命令发送给从服务器去执行

**部分重同步**

从服务器可以在主从服务器之间的连接断开时进行自动重连,在2.8版本前，断线之后重连的从服务
器总要执行一次完整重同步, 2.8以后的版本中从服务器可以根据主服务器的情况来选择执行完整
重同步还是部分重同步

(主服务器ID，复制便宜量OFFSET) 记录在主服务器中

(主服务器ID) 从服务器检查连接的服务器是否和这个ID一致，如果是的话则通知主服务器从OFFSET开始复制数据

## Sentinel故障迁移
Sentinel可以监控服务器的运行状态，并且支持自动选取一个从服务器作为主服务器

启动方式

```python
redis-sentinel /path/to/sentinel.conf
redis-server /path/to/sentinel.conf --sentinel
```

也可以在一台机器上演示这个故障迁移的过程，默认的redis服务器作为主服务器，同时修改配置文件启动一个
工作在6380端口的服务器复制主服务器作为从服务器

然后写一个最简单sentinel配置
```python
sentinel monitor master 127.0.0.1 6379 1
sentinel down-after-milliseconds master 60000
sentinel failover-timeout master 90000
sentinel parallel-syncs master 1 

sentinel monitor resque 127.0.0.1 6380 1
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 1
```

这样先启动主服务器, 再启动复制主服务器的从服务器，然后启动sentinel进行监控； 这时候用kill将主服务器
关掉，就可以观察到6380的从服务器切换成服务器了，之后再启动6379端口的服务器，它也已经作为从服务器存在了

