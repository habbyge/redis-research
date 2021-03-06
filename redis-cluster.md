## Ping和Pong数据包内容 ##
Ping和Pong数据包由所有类型（比如发起投票的数据包）的通用的头部，还有一部分特别的内容是Gossip部分，同时这也是Ping和Pong数据包的特色，只有Ping和Pong包含Gossip。
通用的头部包含以下信息：

- Node ID,一个160bit的伪随机字符串，NodeID在集群创建时分配以后保持不变
- currentEpoch和configEpoch两个属性，这两个属性在redis cluster集群中的分布式算法中有很大的用场（细节在下一节介绍），slave节点的configEpoch是该slave和master最近一次通讯得到的master的configEpoch
- 节点的标志之类的属性，例如标志节点是主还是从的标记，和其他标记信息
- 节点负责的哈希槽的bitmap数组，如果节点是从，则是master的哈希槽
- 端口，发送者的tcp基准端口（这个端口是redis用来接收客户端请求，这个端口加上10000就是集群总线端口）
- 集群状态，发送者认为的集群状态，注意这里仅是发送者眼中的集群状态
- 如果发送者是从，则还有master的nodeID

Ping和Pong数据包包含gossip节段。Gossip节段发送者将自己知道的集群节点列表提供给接收节点。redis集群，并不是每次都将所有的其他已知节点列表都发送给接受者，而是随机选取部分节点。
对于每个添加到Gossip部分的节点会将以下信息报告给接受者：

- Node ID
- 节点的ip和端口
- 节点的flags，例如pfail

Gossip阶段允许接收节点获取其他节点的从发送者的角度认为的状态信息（e.g pfail）。Gossip在集群中主要用来失败判定和节点发现。

## 失败判定 ##
Redis集群的失败判定是在一个主节点或者从节点被大多数节点认为是不可达的。判定的结果，或者晋升从成为主，或者如果晋升失败，整个集群处于失败状态从而不再接受客户端的任何请求。
每个节点都从其他已知节点获取一些列的标志位信息，其中有两个标志用作失败判定叫做pfail和fail。pfail意味着可能失败，它是一种不确认的失败类型。FAIL意味着节点是失败而且在一定时间内被大部分节点都认同的。无论是master还是slave都可以标记另外一个节点为PFAIL。


**PFAIL标记：**
一个节点标记另外一个节点为PFAIL当该节点超过node_timeout的时间不可达。

集群节点不可达的概念是指我们使用主动ping以后等待接受pong，如果等待的时间超过NODE_TIME.因此NODE_TIME必须大于网络的RTT这种机制才可行。为了增加可靠性，节点会在等待ping的回应时间超过NODE_TIME的一半以后关闭连接并在随后尝试重新连接。这种机制用来保障连接的保活，因此一般情况下即使因连接断了也不能导致假的节点失败报告。

**FAIL 标记**

PFAIL标记仅仅是单个节点认为其他节点处于fail状态。PFAIL标记不能触发slave的晋升，因为他不是充分的。对于单个节点，只有PFAIL晋升成FAIL才被真正认为是Down。

就像在本文档节点心跳那一节描述的，每个节点都向每个其他节点发送包含一些随机选取已知节点的状态的gossip信息，这样每个节点最终会接收到所有其他节点的状态信息，通过这种方式收集关于某个节点的来自其他节点的失败判定。

当满足以下所有条件时，升级PFAIL成为FAIL:

- 节点A，已知节点列表中包含节点B被标记成PFAIL
- 节点A通过Gossip收集集群大部分主节点自身认为的节点B的状态
- 大多数的主节点标记它为PFAIL以及 在不超过NODE_TIMEOUT * FAIL_REPORT_VALIDITY_MULT time的时间内仍然处于PFAIL

如果以上条件都满足，节点A将会

- 标记节点B为FAIL
- 将这个节点BFAIL的消息广播给整个集群

FAIL消息将会强制接收节点标志失败节点为FAIL

注意FAIL标记大部分情况下是不可逆转的，节点可以从PFAIL升级为FAIL，但也有以下两种可能消除FAIL：

- 节点又可达了，同时节点是从节点，这种情况下FAIL状态可以消除因为从节点不会做failover
- 节点又可达了，同时节点是不处理任何slot的主，这种情况下FAIL状态可以消息因为不处理任何slot的主对集群没有任何影响。
- 节点又可达了，他是主，但是很长时间都没有从晋升成主，说明没有failover，slot还是由它处理，这种情况下FAIL状态可以消除。

PFAIl -> FAIL的迁移使用一种协议的形式，这种协议的方式是弱的

1）节点收集其他节点的信息是在一段时间内的，即便是大部分主节点都认同，实际上这也只是不同节点在不同的时间的认定，由于时间窗口的存在，所以我们不能确定这种状态是稳定的。

2）每个节点判定FAIL以后会使用FAIL消息广播给整个集群的节点并强制执行，但这并不保证集群中的每个节点都会收到这个消息。举个例子，比如存在网络分区。

然而redis集群的失败判定有活跃度要求：所有节点应该认定给定节点的状态，即使出现了网络分区混乱。有两种场景源自经典的脑裂问题，大部分节点相信某节点处于FAIL状态，然而小部分节点却认为不是FAIL状态。无论哪种场景集群最终都给出关于节点状态的单一观点:

场景1: 如果大部分主（处于同一网络分区）标记一个节点为fail，通过链式的效应最终其他节点都会将这个节点标记为FAIL

场景2：




