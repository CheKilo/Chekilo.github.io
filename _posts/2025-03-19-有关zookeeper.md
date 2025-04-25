---
layout: post
title: 有关zookeeper
header-img: img/in-post/head/14.jpg
header-style: text
catalog: true
tags:
  - zookeeper
  - 中间件
  - 学习
---
首先简单介绍下 Zookeeper 集群，一个 Zookeeper 集群通常由一组机器组成，一般3~5台集群就可以组成一个 Zookeeper 集群。集群拓扑图基本如下：

![img.png](/img/in-post/2025-03-19/img.png)

Zookeeper 集群中每一个节点都会在内存中维护当前的节点状态，并且彼此之间保持着通信

**Leader**

Leader 节点整个 Zookeeper 集群工作机制中的核心，主要工作是处理客户端的读写请求，及集群内部各服务的调度。注意只有 leader 能够处理写请求。

**Follower**

处理客户端的读请求；如果有写请求，则将写请求转发给 leader；参与 leader 选举投票等。

**Observer**

这是自 Zookeeper 3.3.0 版本引入的一个新的角色，主要是为了解决大规模 Server 场景下因 leader 选举投票成本增加导致写性能下降的问题。Observer 的工作原理和 follower 基本一致。处理客户端的读请求，将写请求转发给leader。和 follower 唯一的区别在于，Observer 不参与任何形式的选举，包括 leader 选举。

一般而言，中小型规模的 Zookeeper 集群中只包含 leader 和 follower 两个角色，这容易让我们忽略 observer 角色的存在

Zookeeper 的数据模型是一棵类似 Unix 文件系统的 ZNode Tree 即 ZNode 树，但是没有引入传统文件系统的目录或者文件等概念，而是使用了称为 “数据节点” 的概念，术语叫做 ZNode。ZNode 是 Zookeeper 存储数据的最小单元，每个 ZNode 可以保存数据，也可以挂载子节点，其中根节点是 /。

示意图如下：

![zk示意图](/img/in-post/2025-03-19/img_1.png)

使用过 Zookeeper 的同学应该都知道，Zookeeper 主要提供了两个核心功能：
* 管理（存储、读取）客户端提交的数据；
* 为客户端提供数据节点的监听服务；

这里就涉及到 Zookeeper 的两个重要特性，就是它的 ZNode 模型与 Watcher 机制。

**ZNode 模型**

前面讲到 Zookeeper 是由数据节点 ZNode 构成的，Zookeeper 中的每个数据节点都是有生命周期的，其生命周期的长短取决于 ZNode 的节点类型。ZNode 根据其生命周期和特点可分为 4 类

![zk节点生命周期](/img/in-post/2025-03-19/img_2.png)

* 持久性节点（PERSISTENT）：客户端与 Zookeeper 断开会话后，该节点依旧存在，直到执行删除操作才会清除节点。
* 持久性顺序节点（PERSISTENT_SEQUENTIAL）：另一种持久节点，Zookeeper 会给该节点名称加上一个数字后缀，进行顺序编号。
* 临时节点（EPHEMERAL）：节点的生命周期和客户端的会话绑定在一起，客户端与 Zookeeper 断开会话后，该节点就会被自动删除。各个场景中很多都是利用 Zookeeper 临时节点这个特性的。
* 临时顺序节点（EPHEMERAL_SEQUENTIAL）：概念和上面类似，Zookeeper 也会给该节点进行顺序编号。

前面提及了 ZNode 是存储数据的最小单元，除了存储用户数据外，ZNode 还有以下特点：
* 包含 ZNode 修改/访问的时间、事务id(zxid)，ACL 权限、版本等状态信息；
* 所有的事务请求在 ZNode 端都是顺序和原子性的；
* 数据主要存储在内存中，磁盘中保存事务日志、快照数据等；

**Watcher 机制**

Watcher 机制也称监听机制，它是 Zookeeper 的关键特性，是通过 ZooKeeper 实现分布式发布/订阅、分布式锁、集群管理等功能的基础。

![watcher](/img/in-post/2025-03-19/img_3.png)

如上图所示，Zookeeper 允许客户端向服务端注册一个 Watcher 监听器，当服务端的一些指定事件触发了该监听，比如节点创建、删除，节点数据变更等事件，Zookeeper 就会向注册了监听器的客户端发送相应的事件通知。
