---
tag: zookeeper
---

ZAB协议全称是（Zookeeper Atomic Broadcast）zookeeper原子广播协议。

协议不同于其他的二阶段提交的方式，去除了中断回退的逻辑，而采用了消息广播+崩溃恢复的方式。

## 1. 消息广播

它的核心是定义了那些修改数据的逻辑：

即zookeeper节点是主从架构的，所有的写请求由leader节点处理
（读请求可以由follower处理，但不能保证强一致性，只能保证最终一致性）

leader节点处理的时候，采用类似二阶段提交的方式（2pc）

1. 收到客户端请求，leader生成一个请求proposal
2. leader给所有的follower维持一个队列，按照FIFO的方式广播proposal
3. follower收到proposal之后，先写入本地的事务日志，然后返回leader一个ack确认通知
4. leader收到超过一半的ack确认，就广播commit通知，并自己commit。
5. follower收到commit通知后，就commit事务。

## 2. 崩溃恢复

崩溃恢复有两个核心的目标：

1. leader上已经commit的事务，所有follower都要提交。
2. leader上还未commit的事务，所有follower都要丢弃。

### leader选举

新的leader选举很简单，从follower中选取具有最大事务id（ZXID）的，作为新的leader。
这样可以确保其具有最完整的事务记录，同时也简化了数据恢复的过程。

### 数据恢复

leader会将逐个对比follower的事务状况，并将还未提交的事务发给follower，并紧随发送一个commit。
在等待follower全部提交了事务之后，leader就把它加入到可用的follower中去。

### 事务丢弃

事务id（ZXID）采用的是一个64位的数字表示，其中低32位表示递增的事务id，高32位表示leader的周期。

当一个新的leader被选举出来后，就会将当前最大的事务id的高32位+1，表示新的一轮选举周期，并将低32位置0。

这样，当原来的leader服务器以follower的身份加入集群的时候。

1. 因为事务id比新的leader小，所以肯定不会成为leader。
2. 新的leader会比对事务id，并要求其进行一个回退操作。

## 深入leader选举过程

leader选举发生在服务集群刚启动的时候，以及运行时leader服务器挂了的时候。

它们的过程本质上是**一样**的。

1. 状态变更

每个follower将自己的状态变成LOOKING状态

2. 发起第一轮投票

每个服务器向集群广播投票，消息形式为`(SID, ZXID)`，其中SID为服务器id，
ZXID为当前服务器最大的事务id。

3. 接收投票，并更新投票结果

对比的时候，先对比ZXID，再对比SID，并将大的投票更新为自己新的投票，并再次广播。

4. 确定leader，改变状态

在收到过半的投票之后（n // 2 + 1），服务器改变状态，变为LEADING或FOLLOWING。
