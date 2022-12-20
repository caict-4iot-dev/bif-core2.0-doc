# 6.共识

## 6.1 概述

共识是区块链的核心模块，其目的是使得各个节点的数据状态达成一致。其原理是各节点有共同的初始状态，然后通过共识算法形成一致的中间交易，最后各节点形成一致的最终状态。综合考虑目前市面共识算法的先进性以及星火链对性能的要求，计划基于hotstuff构建星火链的共识算法，hotstuff共识算法的先进性如下：

1）通过在投票过程中引入门限签名实现了的消息验证复杂度；

2）使用了流水线共识加快了共识效率；

3）具有乐观响应性（Optimistic Responsiveness）；

4）安全性（Safety）与活性（Liveness）解耦。

## 6.2 安全性（Safety）

共识算法经过三阶段达成一致，分别为Prepare阶段、Precommit阶段以及Commit阶段。流程如下：

<img src="../_static/images/7.2-1安全性说明.png">

流程描述前对术语定义如下：

**1）Type**，有五种消息类型，NEWVIEW、PREPARE、PRECOMMIT、COMMIT及DECIDE，其中NEWVIEW为试图切换即leader重新选举阶段，其他可参照上图对应各阶段的消息类型;

**2）Node**, 节点可以理解为需要提交的交易集合，其数据格式为<parent, cmds>，nodes通过parent指针进行连接，组成一棵树。Parent可以用hash实现。

**3）QC**，QC为Quorum Certificate仲裁证书，是一种加密证据，各个Replica节点对消息进行签名后，由Leader节点汇总后的一种加密数据，数据格式为<type，view_number,node, [signature]>; signature的格式为对<type, view_number, node>的签名数据。

**4）Msg**，由于hotstuff是基于消息传递的一致性算法，所以完整的消息格式为：<Type, view_number, node, justify>,Type为消息类型，view_number为视图编号，node为？，justify为QC。

**5）voteMsg**, 投票类型的消息，格式为<Msg, partialSig>, 最终格式为<Type,view_number, node,justify, partialSig=signature<Type, view_numnber,node>>；

**6）prepareQC**, 由大多数的replica对PREPARE消息投票成功后的仲裁证书，其格式为<PREPARE, view_number, node_hash, [signature]>，其中node应为<cmd>原始交易值。

流程解释如下：

<img src="../_static/images/7.2-2流水线说明.png">

在Chained HotStuff协议中，prepare阶段的副本的投票由leader节点加以收集，并且储存在状态变量genericQC里。接下来，genericQC被转发给下一个视图的leader，实质上是将下一阶段的职责（这曾经由pre-commit阶段负责的）委托给下一个leader节点。然而，下一位leader实际上并不单独进行pre-commit阶段，而是启动一个新的prepare阶段，添加自己的提议。视图v+1的prepare阶段同时充当视图的pre-commit阶段。视图v+2的prepare阶段同时充当视图v+1的pre-commit阶段和视图v的commit阶段。

针对Chained HotStuff，星火的共识流程如下：

1）由星火核心模块调用共识模块，将共识cons_value值传递到共识模块；

2）共识模块将Cons_value追加到最新树叶子节点，封装为cur_node，接下来判断当前的Replica是否为leader，如果为leader则签名并发送Proposal消息至各个Replica，Proposal消息为<PROPOSAL, cur_node<cons_value, view_number, highQC>, signature>；

3）Replica接收Proposal消息，此刻对树分支做判断，更新当前的highQC为前一个节点的genericQC，并且追溯到父节点的祖父节点为commit状态，此时可以对其做**执行处理**；

4）Replica和Leader，此时更新当前的view高度为proposal消息的高度（对应论文中的安全性检测和或许检测），接下来，对对cur_node签名后，发送VOTE消息到Leader，VOTE消息为<VOTE, cure_node_hash, signature<VOTE, cur_node_hash>>；

5）Leader接收各Replica的VOTE消息后，判断VOTE消息是否大于绝大多数，如果正好为绝大多数，则将合并后的genericQC更新到当前的node中。

## 6.3 活性（Liveness)

为保证共识算法的活性（liveness），Chained Hotstuff算法提出PaceMaker（起搏器）机制，其工作流程分为两种情况：启动正常共识（Beat）以及超时处理流程。

启动正常共识是指区块链节点接收到客户提交的交易时，如何提交到Safety模块。其流程如下：

1）节点接收用户提交的交易，交易入到交易缓存集，PaceMaker定时收集交易序列，判断上一共识流程是否完成，如果完成即可启动下一共识流程。

2）节点启动时，定时检查没有达到终态的decision，发起节点轮换；将replica id 为view_number + 1 / replica_count的节点设置为下一轮共识的leader节点。接下来依次发起新三轮共识（因为是流水线作业），这三轮共识的交易集为空即可。使得最新一次正常共识达到decision状态。

## 6.4 共识流程

从交易提交到交易被确认的整体流程如下：

<img src="../_static/images/7.4-1共识流程.png">

1）用户提交交易后，交易入到交易池。定时任务启动后，将交易集合及部分区块头信息序列化，并提交到共识模块。

2）共识模块按照三阶段chained hotstuff流程进行共识。

3）区块共识完成后，通知ledger核心模块，ledger模块进行预执行，执行结果只保留在内存。所以该模块需要实现一个memory block tree，因为根据hotstuff共识算法，由于节点作恶、超时，某些分支最后可能会被丢弃，只有主分支被保留。