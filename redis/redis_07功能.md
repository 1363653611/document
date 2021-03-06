---
title: 07 redis 自身功能
date: 2020-02-12 09:14:10
tags:
  - redis
categories:
  - redis
topdeclare: true
reward: true
---

# Redis 事物

##　事物介绍

- Redis的事务是通过**MULTI，EXEC，DISCARD和WATCH**这四个命令来完成。
- Redis的单个命令都是**原子性**的，所以这里确保事务性的对象是**命令集合**。
- Redis将命令集合序列化并确保处于一事务的**命令集合连续且不被打断**的执行。
- Redis**不支持回滚**的操作。

## 相关命令

#### MULTI

注：**用于标记事务块的开始**。

　Redis会将后续的命令逐个放入队列中，然后使用**EXEC命令**原子化地执行这个命令序列。
　
　语法： `MULTI`

#### EXEC

在一个**事务中执行所有先前放入队列的命令**，然后恢复正常的连接状态。

语法：`EXEC`

#### DISCARD

清除所有先前在一个事务中放入队列的命令，然后恢复正常的连接状态。

语法：`DISCARD`

#### WATCH

当某个**事务需要按条件执行**时，就要使用这个命令将给定的**键设置为受监控**的**状态**。

语法：**WATCH key [key ....]**

注：该命令可以实现redis的**乐观锁**

##### 语法

```shell
127.0.0.1:6379> set ss1 vv1
QUEUED
127.0.0.1:6379> get ss1
QUEUED
127.0.0.1:6379> exec
1) OK
2) "vv1"


# 示例2
127.0.0.1:6379> type ss1
string
# 监控 ss1
127.0.0.1:6379> watch ss1
OK
127.0.0.1:6379> MULTI
OK
# 若 ss1 发生变化,则事务提交失败,返回: nil. 成功则为:ok
127.0.0.1:6379> set ss1 222
QUEUED
127.0.0.1:6379> EXEC
1) OK
```

## 事务失败处理

- Redis语法错误(编译器错误)

![image-20210208133433195](redis_07功能/image-20210208133433195.png)

-  Redis类型错误(运行期错误)

![image-20210208133510428](redis_07功能/image-20210208133510428.png)

### 为什么redis不支持事务回滚？

1. 大多数事务失败是因为**语法错误或者类型错误**，这两种错误，再开发阶段都是可以避免的
2. Redis为了**性能方面**就忽略了事务回滚

# Redis 持久化方案

## 概述

Redis是一个内存数据库，为了保证数据的持久性，它提供了两种持久化方案。

1. RDB 方式(默认)
2. AOF 方式

## RDB(redis DataBase)方式

- RDB是Redis**默认**采用的持久化方式。
- RDB方式是通过**快照**(snapshotting)完成的，当**符合一定条件**时Redis会自动将内存中的数据进行快照并持久化到硬盘。

### RDB触发条件

1. 符合自定义配置的快照规则
2. 执行save或者bgsave命令
3. 执行flushall命令
4. 执行主从复制操作

### 在redis.conf中设置自定义快照规则

#### RDB持久化条件

格式：save <seconds> <changes>

示例:

　　save 900 1：表示15分钟(900秒)内至少1个键更改则进行快照。

　　save 300 10：表示5分钟(300秒)内至少10个键被更改则进行快照。

　　save 60 10000：表示1分钟内至少10000个键被更改则进行快照。

#### 配置dir指定rdb快照文件的位置

```shell
# Note that you must specify a directory here, not a file name.
dir ./
```

#### 配置dbfilename指定rdb快照文件的名称

```shell
# The filename where to dump the DB
dbfilename dump.rdb
```

#### 说明

1. Redis启动后会读取RDB快照文件，将数据从硬盘载入到内存
2. 根据数据量大小与结构和服务器性能不同，这个时间也不同。通常将记录1千万个字符串类型键，大小为1GB的快照文件载入到内存中需要花费20-30秒钟

### 快照的实现原理

#### 快照过程

1. **redis**使用**fork函数复制**一份**当前进程**的**副本**(子进程)
2. **父进程继续接受**并处理**客户端发来的命令**，而**子进程**开始**将内存中的数据写**入到**硬盘中的临时文件**。
3. 当**子进程写入完**所有**数据后**会**用该临时文件替换旧的RDB文件**，至此，一次快照操作完成。

#### 注意

1. redis在进行**快照的过程中不会修改RDB文件**，只有快照结束后才会将旧的文件替换成新的，也就是说任何时候RDB文件都是完整的。
2. 这就使得我们可以通过定时备份RDB文件来实现redis数据库的备份，**RDB文件是经过压缩的二进制文件**，占用的空间会小于内存中的数据，更加利于传输。

### RDB优缺点

#### 缺点

使用RDB方式实现持久化，一旦redis异常退出，就会**丢失最后一次快照以后更改的所有数据**。这个时候我们就需要根据具体的应用场景，通过组合设置自动快照条件的方式将可能发生的数据损失控制在能够接受范围。如果数据相对来说比较重要，希望将损失降到最小，则可以使用**AOF**方式进行持久化

#### 优点

RDB可以最大化redis的性能：父进程在保存RDB文件时唯一要做的就是fork出一个字进程，然后这个子进程就会处理接下来的所有保存工作，父进程无需执行任何磁盘I/O操作。同时这个也是一个缺点，如果数据集比较大的时候，fork可能比较耗时，造成服务器在一段时间内停止处理客户端的请求。

## AOF (Append Only File) 方式

### 介绍

默认情况下Redis没有开启AOF(append only file)方式的持久化

开启AOF持久化后每执行一条会**更改Redis中的数据命令**，Redis就会将该命令写入硬盘中的AOF文件，这一过程显示会**降低Redis的性能**，但大部分下这个影响是能够接受的，另外使用**较快的硬盘可以提高AOF的性能**。

### 配置redis.conf

设置appendonly参数为yes

```shell
appendonly yes
```

AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的	

```shell
dir ./
```

默认的文件名是appendonly.aof，可以通过appendfilename参数修改

```shell
appendfilename appendonly.aof
```

### AOF重写原理(优化AOF文件)

1. Redis可以在AOF文件体积**变得过大**时，自动地**后台对AOF进行重写**
2. 重写后的新AOF文件包含了恢复当前数据集所需的**最小命令集合**。
3. 整个重写操作是绝对安全的，因为Redis在**创建新的AOF文件**的过程中，会**继续将命令追加到现有的AOF文件**里面，即使重写过程中发生停机，现有的AOF文件也不会丢失。而**一旦新AOF文件创建完毕**，Redis就会从**旧AOF**文件**切换到新AOF**文件，并开始对新AOF文件进行追加操作。
4. AOF文件有序地保存了对数据库执行的所有写入操作，这些写入操作以Redis协议的格式保存，因此AOF文件的内容非常容易被人读懂，对文件进行分析(parse)也很轻松。

### 参数说明

1. \#auto-aof-rewrite-percentage 100：表示当前aof文件大小超过上次aof文件大小的百分之多少的时候会进行重写。如果之前没有重写过，以启动时aof文件大小为基准。
2. \#auto-aof-rewrite-min-size 64mb：表示限制允许重写最小aof文件大小，也就是文件大小小于64mb的时候，不需要进行优化

### 同步磁盘数据

Redis每次更改数据的时候，aof机制都会将命令记录到aof文件，但是实际上由于**操作系统的缓存机制**，**数据**并**没**有**实时写入到硬盘**，而是**进入硬盘缓存**。**再通过硬盘缓存机制去刷新到保存文件中**。

### 参数说明

1. appendfsync always：每次执行写入都会进行同步，这个是最安全但是效率比较低
2. appendfsync everysec：每一秒执行
3. appendfsync no：不主动进行同步操作，由于操作系统去执行，这个是最快但是最不安全的方式

### AOF文件损坏以后如何修复

服务器可能在程序正在对AOF文件进行写入时停机，如果停机造成AOF文件出错(corrupt)，那么Redis在重启时会拒绝载入这个AOF文件，从而确保数据的一致性不会被破坏。

当发生这种情况时，可以以以下方式来修复出错的AOF文件：

　　1、为现有的AOF文件创建一个备份。

　　2、使用Redis附带的redis-check-aof程序，对原来的AOF文件进行修复。

　　3、重启Redis服务器，等待服务器字啊如修复后的AOF文件，并进行数据恢复。

## 如何选择RDB和AOF

1. 一般来说，如果对数据的安全性要求非常高的话，应该同时使用两种持久化功能。
2. 如果可以承受数分钟以内的数据丢失，那么可以只使用RDB持久化。
3. 有很多用户都只使用AOF持久化，但并不推荐这种方式：因为定时生成RDB快照(snapshot)非常便于进行数据库备份，并且**RDB恢复数据集的速度也要比AOF恢复的速度要快**。
4. 两种持久化策略可以同时使用，也可以使用其中一种。如果同时使用的话，那么Redis启动时，会**优先使用AOF**文件来还原数据。

