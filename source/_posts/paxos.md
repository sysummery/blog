---
title: Basic Paxos 与 Multi Paxos
date: 2020-03-14 22:25:13
tags:
   - 分布式
photos:
   - ["http://sysummery.top/yizhi.jpg"]
---
Paxos算法是Leslie Lamport于1990年提出的一种基于消息传递且具有高度容错特性的一致性算法，是目前公认的解决分布式一致性问题最有效的算法之一。
<!--more-->
这篇文章是基于[Paxos算法详解](http://zhuanlan.zhihu.com/p/31780743)加上自己的理解和总结完成的。
## Basic Paxos
### 角色
![](http://sysummery.top/paxosjuese.jpg)
1. proposer，负责接收客户端的请求并向acceptor提prepare请求和accept请求。
2. acceptor，接受proposer的请求并在特定情况下给与回复。
3. learner，不参与决策，学习最终达成一致的提案，一旦有多数acceptor对值达成了共识那么就写入值。

### 流程
![](http://sysummery.top/paxos%E6%B5%81%E7%A8%8B.jpg)
在说流程前先确定几个名词的意义
* max_proposal_id acceptor目前通过prepare请求收到的最大的proposal的id
* accept_id acceptor目前决定要accept的proposal的id

下面是流程：
1. 一个客户端请求将一个字段值变成value。proposer为每一个客户端的请求封装成一个proposal，并为这个proposal生成一个去全局唯一的proposal_id。
2. proposal向所有的acceptor发送prepare请求，请求中包含proposal_id，可以不包含value。
3. acceptor对prepare请求进行处理。如果请求中的proposal_id大于max_proposal_id，那么acceptor是接受这个proposal的，会把自己的max_proposal_id更新成proposal_id同时会返回给proposer当前自己已经accept的proposal的value，如果自己之前没有accept任何proposal则会返回空（让proposer自己去决定下一步accept的时候使用哪个value）。如果请求中的proposal_id小于max_proposal_id，则不做回应。
4. 如果proposer收到了多一半的acceptor的回应时，会挑出最大的value，如果所有的value都为空就用自己的value。
5. proposer用第4步获得的value和自己的proposal_id发起accept请求。
6. acceptor处理accept请求。如果请求中的proposal_id大于等于acceptor的max_proposal_id，那么会更新自己的accept_id、max_proposal_id，同时accept这个proposal。如果请求中的proposal_id小于acceptor的max_proposal_id那么会直接返回max_proposal_id。
7. proposal收到了多一半acceptor的结果后，如果发现有结果大于自己的proposal_id的则说明accept了别的proposal，会从步骤1开始重新执行。

下面想做几点说明

1. prepare返回的结果中是acceptor希望尽快达成共识的value（也就是一个acceptor已经accept的value），所以一个proposer发起accept的value不一定使自己的value。
2. prepare通俗点来说的话就是acceptor“决定”讨论哪个proposal，决定的尺度很简单就是要大于acceptor之前已经决定讨论过的proposal_id的最大值。accept通俗来讲就是决定接受一个value。当多一半acceptor接受一个value后整个流程也就结束了。
3. basic paxos有局限性。他必须走完一轮流程后才能走下一轮流程，因为它没有日志的概念。
4. basic paxos会有活锁的可能。

下面是几个案例。s1提出把值更新为x，s5提出把值更新成y。P代表prepare阶段，A代表accept阶段。s1的proposal_id是3.1，s5的proposal_id是4.5。
![](http://sysummery.top/paxos%E6%A1%88%E4%BE%8B1.jpg)

s1的proposal已经被大多数accept了，此时s5的prepare发出后，如果有大多数acceptor返回那么返回的结果中要不是x要不是空而且x一定存在。所以s5的accept请求中的value是想而不是自己的y。最终的结果是x被写入。s5的y只能走下一轮的流程。

![](http://sysummery.top/paxos%E7%A4%BA%E4%BE%8B2.jpg)

如果s1的proposal只被一个acceptor请求，即s3，同时s5的prepare请求收到了s3的回应，那么s5的accept请求中的值还是x

![](http://sysummery.top/paxos%E7%A4%BA%E4%BE%8B3.jpg)
这个案例的情况与上面的正好相反，s1的proposal只被s1自己accept了，而s5的prepare返回的结果中不包含s1的结果，此时s5就会用自己的值y来发起accept请求。

## Multi Paxos
multi paxos算法中在所有proposers中选举一个leader，由leader唯一地提交proposal给acceptors进行表决。这样没有proposer竞争，解决了活锁问题。在系统中仅有一个leader进行value提交的情况下，prepare阶段就可以跳过，从而将两阶段变为一阶段，提高效率。

既然所有的选举都走leader，那么leader就负责为每一个proposal生成一个全局唯一的proposal_id。leader的选举算法可以使用basic paxos,需要两段请求，而且有活锁的可能。一单选出leader就不用再经历prepare阶段了。

## 总结
上面的两个算法是强一致性的。虽然这两个算法中也有“大多数”的概念，但是这个大多数与raft和zab中的又不大一样。paxos中的大多数的作用是决定写入哪个值，一单给客户端返回结果那么各个节点的值一定是最新的，即强一致性。raft与zba中只要大多数节点的值写入就返回，是最终一致性，他们的“大多数”是为了高可用。

