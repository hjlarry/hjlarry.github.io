---
title: "raft"
draft: false
---

# Raft


概述
-------

对于分布式存储系统，通常会通过维护多个副本来进行容错，提高系统的可用性，那么就带来一个问题，如何维护多个副本的一致性？raft就是解决这个问题的算法。

在一个具有一致性、容错性的集群中，同一时刻所有节点对存储在其中的某个值应该有相同的结果，且当少数节点失效的时候，不影响集群的正常工作，当大多数集群中的节点失效的时候，集群则会停止服务（而非返回一个错误的结果）。

我们以[MIT6.824的lab3](https://pdos.csail.mit.edu/6.824/labs/lab-kvraft.html)中构建的容错键值服务为例，大概看一下基于raft算法的工作流程：

![raft](./images/raft_summary.png)

客户端发送Put/Get命令到集群中leader的k/v层，leader会把这个命令先添加到自己的日志中，同时把这个命令通过`AppendEntries`RPC请求发送给自己的follower，并等待绝大多数follower的回复。如果大多数follower(这里也需要算上leader自己)都回复日志已提交，意味着即使失败了这条日志也不会丢失。那么leader就去可以执行这条命令，并把执行结果返回给客户端。在下一次`AppendEntries`请求中，leader会把自己已提交该日志(之前只是添加)附加进去，follower就会去执行这条命令。