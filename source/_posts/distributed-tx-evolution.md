---
title: 分布式事务
date: 2016-10-27 23:01:15
tags: 笔记
---
# 分布式事务背景
现在分布式系统一般由多个独立的子系统组成，多个子系统通过进程间通信（[RPC](http://en.wikipedia.org/wiki/Remote_procedure_call)）互相协作配合完成各个功能。有很多用例会跨多个子系统才能完成，比较典型的是电子商务网站的下单支付流程，至少会涉及交易系统和支付系统，而且这个过程中会涉及到事务的概念，即保证交易系统和支付系统的数据一致性。通常我们谈及的事务是指单机资源的[ACID](http://baike.baidu.com/subview/600227/5926023.htm)属性，所以此处我们称这种跨系统的事务为分布式事务。  

# 分布式事务实现方式

对于分布式事务，通常采用[2PC](https://zh.wikipedia.org/wiki/%E4%BA%8C%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4)（两阶段提交）以及相应的变种3PC来实现（因为2PC有致命的问题，[3PC](https://zh.wikipedia.org/wiki/%E4%B8%89%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4)通过拆分2PC的第一阶段避免了极端情况下的问题，详情请参考coolshell），使用两阶段提交来协调所有参与到分布式事务中的各个事务资源，Java中也提供了JTA规范来标准化分布式事务接口，目前Mysql和PostgreSQL也都默认实现XA接口，从而支持分布式事务了。而且对于开发人员来说，使用分布式事务和单机事务并没有太大的差异，最起码从编程模型上来看，也就实现了所谓的“透明”

# 分布式事务的弊端

前文也讲到了，因为JTA的标准化，使得分布式事务和单机事务的编程模型看起来很接近，从而实现了“透明”，但是单机事务和分布式事务最大的差异是性能并不“透明”，单机事务的资源都在一台节点上完成变更，不会出现跨进程、跨网络的情况，使得单机事务的性能很好。反观分布式事务，会跨多个进程，跨网络，甚至会跨数据中心，对于跨网络的开销和本地事务操作的对比，可以参考网上的资料。

在其中一个事务执行成功之后并且还未提交之前，会对资源进行锁定，一般是加上X锁，在锁定期间对被锁定的资源访问是受限的，直到通过协调的其他各个事务资源都提交才会释放，由此可见，资源被锁定的时间相比单机事务大大加长了，也就直接导致了系统的TPS降低，单位时间执行的事务数量减少了，系统的吞吐量也会降低，同时意味着支持同样数量的TPS需要加入更多的节点。

不仅如此，由于各个事务资源对应的子系统必须完全执行成功才能完成整个功能，那么意味着整个功能的可用性降低了，假如需要三个子系统来协调完成功能A，每个系统的可用性是99.9%，这样功能A的可用性就是99.9%*99.9%*99.9%=99.7%，如果有更多的子系统参与进来，后果可想而知，系统可用性会变得不可接受。

另外，根据木桶原理，决定整个协调过程完成时间的是执行最慢的节点，其他节点则只能等待，整个系统都会跟着慢下来。  

# 强一致性模型的必要性

既然分布式事务有诸多缺点，那么为什么我们还在使用呢？有没有更好的解决方案来改进或者替换呢？如果我们只是针对分布式事务去优化的话，发现其实能改进的空间很小，毕竟瓶颈在分布式事务模型本身，“我们无法用我们制造问题的思维方式去解决我们的制造的问题”，爱因斯坦如是说。

那我们回到问题的根源：为什么我们需要分布式事务？因为我们需要各个资源数据一致性。对，看起来合情合理，我们需要，而分布式事务恰好解决这个问题，但是分布式事务提供的是强一致性，试问下，我们真的需要强一致性吗？大多数业务场景都能容忍短暂的不一致，只是不同的业务对不一致的时间窗口要求不同罢了，像银行转账业务，中间有几小时几天的不一致窗口，用户是可以理解和容忍的，而像电商支付业务，用户的容忍度可能只是30秒的样子，其实30秒对于系统而言已经很长了，还有搜索引擎的收录等等。

通过这些例子，可以确定我们可以忍受短暂的不一致，即我们不需要强一致性，只需要最终一致性。对要求的进一步降低，是不是意味着可以有更加合理的方案呢？ 

# 单机事务+异步复制

以订单子系统和支付子系统为例，如下：


payment | trade
---|---
payment_record | trade_record
user_account | tx_duplication
tx_info |

如上图，payment是支付系统，trade是订单系统，两个系统对应的数据库是分开的。支付完成之后，支付系统需要通知订单系统状态变更，从而开始接下来的操作。

对于payment要执行的操作可以用伪代码表示如下：

``` sql
begin tx;
count = update account set amount = amount - ${cash} where uid = ${uid} and amount >= amount
if (count <= 0) return false
update payment_record set status = paid where trade_id = ${tradeId}
commit;
```

对于trade要执行的操作可以用伪代码表示如下：

``` sql
begin tx;
count = update trade_record set status = paid where trade_id = ${trade_id} and status = unpaid
if (count <= 0) return false
do other things ...
commit;
```

但是对于这两段代码如何串起来是个问题，我们增加一个事务表，即图中的tx_info，来记录成功完成的支付事务，那么为了和支付信息一致，需要放入事务中，代码如下：

``` sql
begin tx;
count = update account set amount = amount - ${cash} where uid = ${uid} and amount >= amount
if (count <= 0) return false
update payment_record set status = paid where trade_id = ${tradeId}
insert into tx_info values(${trade_id},${amount}...)
commit;
```

支付系统边界到此为止，简单吧？那么接下来就是订单系统启动时间程序去轮询访问（直接和间接）tx_info，拉取已经支付成功的订单信息，对每一条信息都执行trade系统的逻辑，伪代码如下：

``` sql
foreach trade_id in tx_info
  do trade_tx
save tx_info.id to some store
```

事无延迟取决于时间程序轮询间隔，这样我们做到了一致性，最终订单都会在支付之后的最大时间间隔内完成状态迁移。  

等等，我们好像还差点东西，交易系统每次拉取数据的起点以及消费记录是否得记录下来，这样才能不遗漏不重复地执行，所以需要增加一张表用于排重，即上图中的tx_duplication。但是每次对tx_duplication表的插入要在trade_tx的事务中完成，伪代码如下：

``` sql
begin tx;
c = insert ignore tx_duplication values($trade_id...)
if (c <= 0) return false
count = update trade_record set status = paid where trade_id = ${trade_id} and status = unpaid
if (count <= 0) return false
do other things ...
commit;
```

另外，tx_duplication表中trade_id表上必须有唯一键，这个算是结合之前的幂等篇来保证trade_tx的操作是幂等的，到此为止，我们已经完全实现了异步复制实现多个单机事务一致性的目标。看起来是个完美的通用方案，不是吗？但是仔细看看也会明白，这个方案也会有自己的问题，比如交易系统要访问支付系统的数据库、系统要多增加几张表等等，我们可以抽取组件实现这些，但是依然隐藏不了那些表。那么是不是有更加优雅的方式来改进呢？答案是肯定的，接下来继续。

# 带有事务功能的MQ做中间人角色

其实在上边的方案中，tx_info表所起到的作用就是队列作用，记录一个系统的表更，作为通知给需要感知的系统的事件。而时间程序去拉取只是系统去获取感兴趣事件的一个方式，而对应交易系统的本地事务只是对应消费事件的一个过程。在这样的描述下，这些功能就是一个MQ——消息中间件。如下：

```
graph TB
    subgraph trade
    trade_record --- tx_duplication
    end
    subgraph payment
    payment_reaord --- user_accout
    end
    user_accout-->MQ
    tx_duplication-->MQ
```

这样tx_info表的功能就交给了MQ，消息消费的偏移量也不需要关心了，MQ会搞定的，但是tx_duplication还是必须存在的，因为MQ并不能避免消息的重复投递，这其中的原因有很多，主要是还是分布式的三态造成的，再次不详细描述。

这要求MQ必须支持事务功能，可以达到本地事务和消息发出是一致性的，但是不必是强一致的。通常使用的方式如下的伪代码：  

``` sql
sendPrepare();
isCommit = local_tx()
if (isCommit) sendCommit()
else sendRollback()
```

在做本地事务之前，先向MQ发送一个prepare消息，然后执行本地事务，本地事务提交成功的话，向MQ发送一个commit消息，否则发送一个rollback消息，取消之前的消息。MQ只会在收到commit确认才会将消息投递出去，所以这样的形式可以保证在一切正常的情况下，本地事务和MQ可以达到一致性。  

但是分布式存在异常情况，网络超时，机器宕机等等，比如当系统执行了local_tx()成功之后，还没来得及将commit消息发送给MQ，或者说发送出去了，网络超时了等等原因，MQ没有收到commit，即commit消息丢失了，那么MQ就不会把prepare消息投递出去。如果这个无法保证的话，那么这个方案是不可行的。针对这种情况，MQ会根据策略去尝试询问（回调）发消息的系统（checkCommit）进行检查该消息是否应该投递出去或者丢弃，得到系统的确认之后，MQ会做投递还是丢弃，这样就完全保证了MQ和发消息的系统的一致性，从而保证了接收消息系统的一致性。  

不过这个方案要求MQ的系统可用性必须非常高，至少要超过使用MQ的系统，这样才能保证依赖他的系统能稳定运行。目前提供这种机制的MQ并不多，个人了解到的有阿里的notify和ons。  

# 总结

写在最后，对于数据库实现的异步复制方案是参考[ebay](http://queue.acm.org/detail.cfm?id=1394128)的架构师的一篇论文，详细讲述了实现过程，很经典，可以点链接进去看看。使用MQ的方案是一个比较成熟的方案，而且MQ方案不止带来了这么多好处，带来了异步化、重试机制、系统解耦，还可以在不同访问量的系统之间起到消峰缓冲的作用，可以把系统交互变得更加优雅。  

> 转载 [分布式事务演进](http://yongpoliu.com/distributed-tx-evolution/)

# 2016-10-21 23:07:38 康兴华

> 我们无法用我们制造问题的思维方式去解决我们的制造的问题。--爱因斯坦