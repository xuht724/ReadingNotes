# A Secure Sharding Protocol For Open Blockchains

## 论文阅读

### Abstract

本篇论文提出了一种公链的分布式共识协议称为 Elastico

### Introduction

经典的拜占庭共识协议在开放式的环境（比如说加密货币）中并不适合，因为两个基本的挑战：第一，许多论文都是假设网络中的节点已经在公钥基础设施中建立起了身份，但是这个假设并不存在于开放的环境中，比如Bitcoin；第二，PBFT需要至少节点数平方数的消息数，因此它是有带宽限制的。网络带宽的限制甚至几百个节点的网络的交易吞吐量。

results：Elastico呈现出了随着算力的增加近乎线性的吞吐量的增长，并且随着网络的增加需要平方数的通讯信息(message)。Elastico可以容忍f<n/4个数的恶意节点，其中f和n分别代表恶意节点算力占总算力的比值。本项目针对Bitcoin进行了一些修改，从而可以允许更好的性能参数。

Elastico在每一个分片(c identity)中的通信开销：最好的消息复杂度（message complexity）为$O(c^2)$，最坏的消息复杂度为$O(c^3)$。总体来说，整个系统的通信开销最差为$O(nc^3)$。

Elastico将区块广播的过程和共识的过程解耦，因此每个节点的带宽开销为一个定值，不管网络的大小。

### Problems and Challenges

Problems definition 部分主要是介绍一下problem的定义，threat模型和安全假设

security assumption：1. 诚实的processor之间是network graph是相连的。2. 诚实processor之间的communication channel是同步的；即如果一个诚实的用户广播了任何的message，那么其他诚实的processor将会在一个延时之后接受到这一条交易。

chanllenges：

1. processors没有一个内在的身份或者外在的PKI去信任，一个malicious的processor可能会模拟出很多虚拟的processor从而创造一个大规模的女巫攻击，因此协议必须预先设置一些机制限制可能被malicious的processor创造的进行女巫攻击的身份。
2. 在identity被确定之后，接下来就是如何去设计sharding protocol，让每一个shard中的大多数节点是诚实的。这需要的是一个共享的随机性来达到。然而在一个分布式的网络中达到很好的随机性是一个很难的problem

### Elastico Design

key idea：自动化parallelize所有可获得的算力，将他们划分成几个更小的committee，每一个committee负责处理一些disjoint的transaction。committee的数目和网络中的算力线性相关。所有的committee中的成员个数为c，该committee运行一个经典拜占庭共识协议。一个final committee（consensus committee）负责将其他committe达成的shard计算一个密码摘要（cryptograhic digest）广播到全网。在每一个epoch的最后一步中，final committee需要生成一套共享的公开random bit strings。这些random strings被用来在后续的epoch中提供随机性，来确保adversary不能够从之前的epoch中模拟出任何关于算力分配的有用信息。

在每一个epoch，processors将会按照下列5步的过程执行。

1. 身份的建立和committee的建立：每个processor本地生成一个identity，该identity由公钥，IP地址和一个POW证明组成。该processor必须要解决一个POW难题，这样如果malicious processor想要发起攻击，就会受到自己掌握的computation power所限制，每一个processor会根据他们的identity被分配到一个committee
2. Overlay setup for committees：在这个步骤中，processors会和其他本committee中的processors communicate。一个committee中的节点应该是全连接的。一个简单的方式就是对于其中的每个processor去广播他们的身份和他们的committee的成员到网络中的全部节点，但是这样会导致$O(n^2)$的通信开销，是通信不节约的。本文提出了一种高效的通信方式，可以将信息开销降低至$O(nc)$，就让同一个committe中的成员可以快速验证互相的身份。
3. Intra-committee Consensus：每个committee中的成员通过使用PBFT共识协议对于交易达成共识。存在一种高效的方法去确保不同的committee处理的交易是不相交的，比如说可以根据committee ID来选择交易。每一个committee将挑选出来的区块发送给final committee，该区块会携带超过半数的成员的签名（c/2+1）。
4. Final consensus broadcast：final committee 中运行一个拜占庭共识协议，根据收集到的其他的committee的value计算出最终的value，并且将最终的结果广播到全网。
5. Epoch Randomness Generation：final committee 运行一个分布式的commit-and-xor 方案（scheme）生成一个exponential biased 但是 bounded 的随机数集。该随机数集将会被广播到全网，并且被运用到下一个epoch中的POW过程中。

参数设置：

n：代表在每一个epoch中的所有参与到网络中节点的个数（total number of identity）

f = 1/4 ：代表掌握在malicious users手中的computation power的阈值

c：每一个committee的大小；关于c的大小和安全的参数以及希望的网络延迟有关

$2^s$：代表的是committee的个数，有关系：n = c * $2^s$

Efficiency分析：在上述的过程中，过程2，4，5该协议需要$O(c)$次广播，每一次广播需要的messgae数目为$O(n)$；过程3，4，5需要最多c轮的$c^2$的多播（multicast），对于大小为c的committee。因此，message的总数为$O(nc+nc^3)$可以被近似的认为是$O(n)$因为认为c是一个和n相比相对较小的常数。

Security：在每一个epoch中，对于f = 1/4.可以确保S1-S5的安全性质（关于S1-S5是啥还是要具体看文章）

在接下来的部分会分别详细介绍这5个步骤。

#### Identity Setup and Committee Formation

首先，每一个processor本地会生成他们的身份，包括IP地址和公钥（IP，PK）；此外还需要计算一个PoW难题。其中PoW的seed由在每一轮epoch最后生成一个epochaRandomness来确保PoW不会被提前计算出，每一个processor所做的计算任务如下：

![image-20211207151756873](https://raw.githubusercontent.com/xuht724/blog-img/main/img/image-20211207151756873.png)

其中D是一个网络中预先定义好的参数来决定processor计算PoW难题所需要的工作量。需要注意的是可以使用Proof of Stake，Proof of Space来建立起processor的身份。

接下来就是根据计算出来的身份进行随机的分组，分组的过程一定要随机。该方案使用的是PoW结果的最后s位来确定每个节点所在的分片。

（之后本部分的内容讨论的是PoW的安全性的问题）

#### Overlay Setup for Committees

在身份和committe建立完成之后，committee的成员需要一个方式去建立点对点的connection和committee peers。一个naive的方案就是广播自己的身份给网络中的节点，但是这样做会造成$O(n^2)$的通信开销。

本篇论文采用的方案是让一个特别的committee提供这样的一个“directory”。所以的节点可以根据directory committee提供的目录获取到自己committee中的其他peer，从而建立起点到点的link。这个directory committee可以认为是最初的产生的一个size大小为c的committee。在第一步中，任何的processor找到了一个PoW的解来证明身份，该processor会通知所有的directory members。每一个processor会告诉他在网络中看到的最初的c个identity。一旦某一个committee包括了至少c个identity，directory committee会组播committee list给该committee的成员。

本系统的安全基于两个假设：

1. malicious directory committee member能做的最多就是不发送诚实节点的身份，而是偏爱malicious identity。这是因为PoW的限制，导致恶意节点也无法拥有更多的一个身份。
2. 每个一般committee的成员会根据所有的directory committee的成员发送的list来判断本组的节点，只要在directory committee中超过了2/3的节点是诚实的，那么在一个committee中诚实的节点就可以知道其他committee的身份。

#### Intra-committee Consensus

一旦一个committee已经建立，他们可以使用现存的任意一种拜占庭共识协议去筛选出区块。

一旦共识达成，筛选出来的交易集合会被至少c/2 + 1个签名。每一个committee的成员会将签名后的final value发送给final committee，final committee会验证区块是否有足够的签名。

这个final value可以是一个密码学的digest比如说一个由达成的交易组成的一个merkle hash root。

#### Final Consensus Broadcast

在这一步final committee将各个committee的value汇总，并且创造一个最终共识结果的密码学digest，并且将最终的结果广播。

#### Generating Epoch Randomness

在协议的最后一步中，需要final committee生成一个随机字符串作为下一个epoch的PoW难题。本协议使用的方案氛围两个阶段：

1. 第一个阶段，final committee中的每个成员生成一个r-bit的random string，并且将该string的hash值发送给final committee中的成员。final committe中的成员达成一致筛选出一个hash value集合，并且将其广播。该集合中至少包括2c/3的hash值。
2. 第二个阶段，final committee中的成员广播条包括了random string 的message给网络中的每个节点。这是为了确保final committee已经筛选出了hash value集合并且adversary无法改变该集合。

用户可以选择使用hash值被包含在集合中的合法的字符串。

为了下一个epoch的使用，每一个用户可以对c/2+1个随机字符串进行XOR计算，可以注意的是并不一定每个user可以使用不同的字符串，这些字符串是上述合法字符串集合的子集。

该字符串用于下一个Epoch的PoW证明，用户需要在自己的PoW solution上附加自己的用于生成随机种子的字符串的集合。

### Security Analysis

本篇文章进行的安全分析有：

1. Good Randomness
2. Good Majority
3. Good Views with bounded inconsistency
4. Consensus
5. Bounded Bias to Randomness

### Implementation & Evaluation

基于BTC version0.12.1客户端实现了Elastico组件。

选择了PBFT作为committe内部的共识协议

每个分片中筛选出的transactions的data size是1MB（和BTC一致）

使用了BTC的网络实现作为P2P的网络层

实验设置：在Amazon EC2上运行了该网络，分析了从100-1600个节点的网络的吞吐量。每两个节点共享一个EC2 Instance，包括2个Amazon的vCPU和3.75GB的内存。

![image-20211212150844716](https://raw.githubusercontent.com/xuht724/blog-img/main/img/image-20211212150844716.png)

Figure1 展示了Elastico基本随着网络规模的增加实现吞吐量的线性提升。不过每个epoch的时间也会增加，因为需要更多的时间去找到PoW的解。并且发现在committee构建完成之后，每个committee内部达到共识的时间基本是一个定值，不考虑网络规模的影响。

和相关工作的对比：Bitcoin、Bitcoin-NG、传统的基于PBFT的区块链系统（该方案带宽的消耗和message的数目会随着网络规模的增加而先行增长。）

### Related Work

文章写这部分主要分了三块：

1. 中心化的分片协议：Google Spanner，Scatter，RSCoin，这些分片协议不容忍拜占庭的容错，并且假设会有公钥基础设施以及对外部random seed的访问
2. 区块链扩容协议：
   1. GHOST、Bitcoin-NG这种添加更多区块到链上的方法
   2. 链下扩容：闪电网络，sidechain
   3. 一些公链的项目：Stella、Ripple、Tendermint
