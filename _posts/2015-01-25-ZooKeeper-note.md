---
layout: post
title: "ZooKeeper(一)怎么用"
tagline: ""
description: ""
tags: [ThreadPool, ThreadLocal]
---
{% include JB/setup %}
##Zookeeper安装
ZooKeeper的安装入门还是官方文档权威些

* zooKeeper Get started: http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html

* 安装完成后可能会出现这种错误：
Exception:
     Unable to read additional data from server sessionid 0x0, likely server has closed socket, closing socket connection and attempting reconnect

    It is the reason that you have configured even number of zookeepers which directly result in this problem,try to change your number of zookeeper node to a odd one.

##Zookeeper都能做什么

###配置中心

故名思议，就是把数据放在ZK节点上，提供订阅者动态获取数据，实现配置信息的集中化管理和动态更新
应用在启动的时候会主动去zk上获取一次配置，同时在节点上注册一个watcher，这样以后每次配置更新的时候都会通知到订阅的客户(关于watcher后面介绍)

###分布式搜索服务
索引的元信息和服务器集群机器的节点状态存放在ZK节点上

###分布式日志收集系统

收集器按照应用来分配收集任务单元，因此可以在zk上创建一个以应用名作为path的节点，并将这个应用的所有机器以字节点的形式注册到p上，
做到机器变更的时候通知收集模块调整收集目标

###上面说的场景，都必须是在zk上存储数据量小，更新频繁的数据用

###负载均衡

软负载均衡，在分布式环境中为了保证高可用性通常同一个应用或者同一个服务都会有多个对等服务，消费者就是在这些对等的服务中取一个来执行相关的业务逻辑
比如消息中间件的服务提供者和生产者

####生产者消息负载均衡

metaq消息在发送的时候必须选择一台broker上的一个分区来发送消息，因此metaq在运行的过程中会把所有的broker和对应的分区信息全部注册到zk上

####消费者消息负载均衡

在消费过程中，一个消费者消费一个或者多个分区列表中的消息，但是一个分区只会由一个消费者来消费。
某个消费者故障或者重启等情况下，其他消费者通过watcher消费者列表，可以感知到这个变化。

###命名服务

客户端能够根据名字获取到资源或者服务的地址，例如dubbo

###分布式锁
ZooKeeper的数据的强一致性,大致步骤
共享锁在同一个进程中很容易实现，但是在跨进程或者在不同 Server 之间就不好实现了。Zookeeper 却很容易实现这个功能，
实现方式也是需要获得锁的 Server 创建一个 EPHEMERAL_SEQUENTIAL 目录节点，然后调用 getChildren方法获取当前的目录节点列表中最小的
目录节点是不是就是自己创建的目录节点，如果正是自己创建的，那么它就获得了这个锁，如果不是那么它就调用
exists(String path, boolean watch) 方法并监控 Zookeeper 上目录节点列表的变化，一直到自己创建的节点是列表中最小编号的目录节点，
从而获得锁，释放锁很简单，只要删除前面它自己所创建的目录节点就行了。
redis也能实现进程锁

![共享锁](http://mouzt.github.io/static/img/4.gif)

###分布式队列
