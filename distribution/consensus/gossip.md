# Gossip 协议

:::tip Gossip

Trying to squash a rumor is like trying to unring a bell.

:::right

—— [Shana Alexander](https://en.wikipedia.org/wiki/Shana_Alexander)，American Journalist

:::

**Paxos、Raft、ZAB 等分布式算法经常会被称作是“强一致性”的分布式共识协议**，其实这样的描述抠细节概念的话是很别扭的，会有语病嫌疑，**但我们都明白它的意思其实是在说“尽管系统内部节点可以存在不一致的状态，但从系统外部看来，不一致的情况并不会被观察到，所以整体上看系统是强一致性的”。**与它们相对的，**还有另一类被冠以“最终一致性”的分布式共识协议**，这表明系统中不一致的状态有可能会在一定时间内被外部直接观察到。**一种典型且极为常见的最终一致的分布式系统就是[DNS 系统](/architect-perspective/general-architecture/diversion-system/dns-lookup.html)，在各节点缓存的 TTL 到期之前，都有可能与真实的域名翻译结果存在不一致**。在本节中，笔者将介绍在比特币网络和许多重要分布式框架中都有应用的另一种具有代表性的“最终一致性”的分布式共识协议：**Gossip 协议**。



Gossip 最早由[施乐公司](https://en.wikipedia.org/wiki/Xerox)（**Xerox，现在可能很多人不了解施乐了，或只把施乐当一家复印产品公司看待，这家公司是计算机许多关键技术的鼻祖，图形界面的发明者、以太网的发明者、激光打印机的发明者、MVC 架构的提出者、RPC 的提出者、BMP 格式的提出者……**） Palo Alto 研究中心在论文《[Epidemic Algorithms for Replicated Database Maintenance](http://bitsavers.trailing-edge.com/pdf/xerox/parc/techReports/CSL-89-1_Epidemic_Algorithms_for_Replicated_Database_Maintenance.pdf)》中提出的一种用于分布式数据库在多节点间复制同步数据的算法。**从论文题目中可以看出，最初它是被称作“流行病算法”（Epidemic Algorithm）的，只是不太雅观，今天 Gossip 这个名字已经用得更为普遍了，除此以外，它还有“流言算法”、“八卦算法”、“瘟疫算法”等别名，这些名字都是很形象化的描述，反应了 Gossip 的特点：要同步的信息如同流言一般传播、病毒一般扩散。**



笔者按照习惯也把 Gossip 也称作是“共识协议”，**但首先必须强调它所解决的问题并不是直接与 Paxos、Raft 这些共识算法等价的，只是基于 Gossip 之上可以通过某些方法去实现与 Paxos、Raft 相类似的目标而已**。一个最典型的例子是比特币网络中使用到了 Gossip 协议，用它来在各个分布式节点中互相同步区块头和区块体的信息，这是整个网络能够正常交换信息的基础，但并不能称作共识；**比特币使用[工作量证明](https://en.wikipedia.org/wiki/Proof_of_work)（Proof of Work，PoW）来对“这个区块由谁来记账”这一件事情在全网达成共识，这个目标才可以认为与 Paxos、Raft 的目标是一致的**。



下面，我们来了解 Gossip 的具体工作过程。相比 Paxos、Raft 等算法，Gossip 的过程十分简单，它可以看作是以下两个步骤的简单循环：

- 如果有某一项信息需要在整个网络中所有节点中传播，那从信息源开始，选择一个固定的传播周期（譬如 1 秒），随机选择它相连接的 k 个节点（称为 Fan-Out）来传播消息。

- **每一个节点收到消息后，如果这个消息是它之前没有收到过的，将在下一个周期内，选择除了发送消息给它的那个节点外的其他相邻 k 个节点发送相同的消息，直到最终网络中所有节点都收到了消息，尽管这个过程需要一定时间，但是理论上最终网络的所有节点都会拥有相同的消息。**

:::center

![](./images/gossip.gif)

Gossip 传播示意图（[图片来源](https://managementfromscratch.wordpress.com/2016/04/01/introduction-to-gossip/)）

:::

上图是 Gossip 传播过程的示意图，根据示意图和 Gossip 的过程描述，**我们很容易发现 Gossip 对网络节点的连通性和稳定性几乎没有任何要求**，它一开始就将网络某些节点只能与一部分节点[部分连通](https://en.wikipedia.org/wiki/Network_topology#Partially_connected_network)（Partially Connected Network）而不是以[全连通网络](https://en.wikipedia.org/wiki/Network_topology#Fully_connected_network)（Fully Connected Network）作为前提；**能够容忍网络上节点的随意地增加或者减少，随意地宕机或者重启，新增加或者重启的节点的状态最终会与其他节点同步达成一致**。Gossip 把网络上所有节点都视为平等而普通的一员，没有任何中心化节点或者主节点的概念，这些特点使得 Gossip 具有极强的鲁棒性，而且非常适合在公众互联网中应用。



同时我们也很容易找到 **Gossip 的缺点，消息最终是通过多个轮次的散播而到达全网的，因此它必然会存在全网各节点状态不一致的情况，而且由于是随机选取发送消息的节点，所以尽管可以在整体上测算出统计学意义上的传播速率，但对于个体消息来说，无法准确地预计到需要多长时间才能达成全网一致**。`另外一个缺点是消息的冗余，同样是由于随机选取发送消息的节点，也就不可避免的存在消息重复发送给同一节点的情况，增加了网络的传输的压力，也给消息节点带来额外的处理负载`。



达到一致性耗费的时间与网络传播中消息冗余量这两个缺点存在一定对立，如果要改善其中一个，就会恶化另外一个，**由此，Gossip 设计了两种可能的消息传播模式：反熵（Anti-Entropy）和传谣（Rumor-Mongering），这两个名字都挺文艺的**。熵（Entropy）是生活中少见但科学中很常用的概念，**它代表着事物的混乱程度。反熵的意思就是反混乱，以提升网络各个节点之间的相似度为目标，所以在反熵模式下，会同步节点的全部数据，以消除各节点之间的差异，目标是整个网络各节点完全的一致**。但是，在节点本身就会发生变动的前提下，这个目标将使得整个网络中消息的数量非常庞大，给网络带来巨大的传输开销。**而传谣模式是以传播消息为目标，仅仅发送新到达节点的数据，即只对外发送变更信息，这样消息数据量将显著缩减，网络开销也相对较小。**



**redis-cluster的分片消息广播是走的流言协议**。

