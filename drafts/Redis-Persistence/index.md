+++
title = "Redis Persistence"
description = ""
date = "2021-04-22T21:29:08+08:00"
updated = "2021-04-22T21:29:08+08:00"
draft = true
[taxonomies]
tags = ["Redis", "Database"]
+++

既然都读了一些 `Redis` 的源码了, 可以继续再读一读, 今天整理一下之前简单了解过 `Redis` 持久化的内容.

## 基础介绍

`Redis` 是一个很快的, 基于内存的数据库. 而在内存中则代表着易失性. `Redis` 提供了一些内置的功能, 让我们可以把数据保存到硬盘中. 对于 `Redis` 来说, 这些硬盘中的数据是一种 **备份**, 即在提供服务的过程中, `Redis` 只会通过某些方式构建并维护这些数据, 而不会读取这些数据. 这些数据当且仅当 `Redis` 进行数据恢复的时候使用.

## 背景知识

首先, 说到持久化, 必须了解什么是 **持久化**. 即答: 当然是写到硬盘上就是持久化啦. 粗略的来说这样并没有错, 但是描述并不准确.

可以先看一下 `Antirez` 做的总结:

> The first thing to consider is what we can expect from a database in terms of durability. In order to do so we can visualize what happens during a simple write operation: 
>
> 1: The client sends a write command to the database (data is in client's memory).
>
> 2: The database receives the write (data is in server's memory).
>
> 3: The database calls the system call that writes the data on disk (data is in the kernel's buffer).
>
> 4: The operating system transfers the write buffer to the disk controller (data is in the disk cache).
>
> 5: The disk controller actually writes the data into a physical media (a magnetic disk, a Nand chip, ...)

这个流程描述了一个写操作的流程, 当然, 这个流程也是经过了相当的简化, 尤其是在步骤二和步骤三中, 为了提高效率, 操作系统内部会有复杂的, 多层次的缓存机制.

如果只考虑数据库软件崩溃或者进程被终结, 而没有在操作系统内核上的丢失, 那么在完成步骤三之后, 就可以认为这个写操作是安全. 当操作被提交给了内核之后, 即使软件退出, 内核仍然会负责将数据传输到磁盘控制器.

而当我们考虑到断电这样的情况时, 只有到达步骤五, 我们才能认为数据是安全的, 即数据存储在物理存储设备上.

所以数据安全最重要的三, 四, 五阶段可以被总结为以下三个问题:

* 数据库软件使用系统调用将其用户空间缓冲区转移到内核缓冲区的频率是多少 - 对应第三阶段
* 内核什么时候会将缓冲区刷新 (flush) 到磁盘控制器 - 对应第四阶段
* 磁盘控制器什么时候向物理介质写入一次数据 - 对应第五阶段

## RDB

RDB (Redis Databse) 持久化按指定的时间间隔执行数据集的时间点快照

## AOF

AOF (Append Only File) 持久化记录服务器接收到的每个写操作, 这些操作将在服务器启动时重放, 重建之前的数据集. 命令的日志格式与 `Redis` 协议本身相同, 不过带有附加格式. 当日志变得太大时, `Redis` 可以在后台重写它.

## 参考资料

1. [Redis Persistence - Redis](https://redis.io/topics/persistence)
2. [Redis persistence demystified](http://oldblog.antirez.com/post/redis-persistence-demystified.html)
