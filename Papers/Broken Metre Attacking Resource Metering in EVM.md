# Broken Metre: Attacking Resource Metering in EVM

## 论文笔记

### Abstract

本篇论文证明了执行交易的gas消费和实际所消耗的资源（比如CPU和内存）之间有比较低的关联。（little correlation）

基于此，本篇文章提出了一种新型的DoS攻击，称为资源耗尽攻击（Resource Exhaustion attack）。本篇文章设计了一种遗传算法来生成比一般的合约慢100倍的合约。并且提出了针对这一种攻击的短期和可能的长期解决方案，并且将其提交给了以太坊基金会，得到了5000USD的回报（有点抠门啊我去）。

### Introduction

根据以太坊黄皮书最初的设计初衷，是设计的1gas/µs。（没找到原话，奇怪🤔）

> Tips：虽然没有在以太坊黄皮书上找到这一句话。但是我确实在以太坊基金会主导的[Ewasm Docs](https://ewasm.readthedocs.io/en/mkdocs/determining_wasm_gas_costs/)中的gas cost找到过相关资料，不过和这里说的1gas/µs有一些区别，不过这里说的应该是webassembly中的指令码gas的定价。不过可以说明应该是gas cost和CPU运行时间相关的。
>
> Ewasm Docs中的表述如下：
>
> **Assumption 1**: This specific **Haswell CPU** represents a good average of the Ethereum nodes. We assume that a 2.2 Ghz model is the average.According to Intel, the 2.2 Ghz clock rate roughly equals to `2 200 000 000` cycles per second.
>
> **Assumption 2: 1 second of CPU execution equals to 10 million gas (i.e. 1 gas equals to 0.1 us).**
>
> This equals to **`0.0045` gas per cycle**. (`10 000 000 / 2 200 000 000`)
>
> To put this in perspective, the average block gas limit as of August 2016 is around 4.7 million. With this assumption we allow contract execution to take up to 0.5 second of the 15 seconds block time, which also has to include other processing, including PoW and network roundtrip times.
>
> **Assumption 3**: The gas costs are adjusted on a *regular* basis, at least every 3 years.

本篇文章的贡献：

1. 重放了2.5个月的以太坊智能合约。找到了其中（1）EVM指令集中gas费用和实际资源消耗相比较低的指令；（2）缓存影响程序执行时间的情况。
2. 提出了一种资源耗尽攻击（REA）的策略：提出了一种基于遗传算法的代码生成策略去提供这类攻击。该策略结合了empirical data和遗传算法。
3. 实际实验：证明了REA攻击会滥用EVM的Gas设计中的缺陷。该方法生成的一个最慢的合约是在243代得到的，该代码消耗了9.9 Million gas，EVM执行需要消耗93秒。（目前的区块的gas limit）。实验生成的合约比一般的合约要慢100倍。
4. 文章作者将发现的这个问题提交给了以太坊基金会，提出了一些开发者可以在后续解决这些问题的方案。

### Background

本篇文章中衡量ETH的具体花费的时候使用的是1 ETH = 145 USD（真便宜啊～）。

Gas是主要的抵御DoS的机制，也用于处理激励矿工处理交易。

Gas cost的计算：固定的21000 gas 加上合约的执行花费

EVM的设计者将EVM的指令分成很多类：Zero Tier，base tier（2 gas），very low tier（3 gas），low tier（5 gas），high tier（10 gas）和special tier

**EIP150**: 在早期的时候，以太坊虚拟机中IO heavy的指令都比较便宜，因此以太坊之前遭受了DoS攻击。EIP150提出提升了其中一些指令都花费。[关于EIP150详情可以看这里。](https://eips.ethereum.org/EIPS/eip-150)

在撰写该论文的时候，单一block中的gas limit是10,000,000。这意味着每一个区块大概打包20笔左右的交易。

> 根据etherscan，目前区块的gas limit大概为30,000,000

之前有两个非常经典的因为指令定价不正确而导致以太坊遭受的Dos攻击

1. EXTCODESIZE attack EXTCODESIZE指令的功能是获取给定contract的code的大小。该指令在当时被严重低估（只要20gas）。然而该指令强制要求client去从磁盘上查找contract。再重放这部分的代码的时候，这些malicious transaction需要20 - 80 s的时间去执行。目前EXTCODESIZE的gas设置从20提高到了700。
2. SUICIDE attack。 SUICIDE指令是kill 一个合约然后将其所有的ether发送到一个特定的address，如果该address不存在，那么将会创建一个新的address，在当时SUICIDE指令是不收费的，所以一个攻击者可以以非常低的成本创建一个新的地址。这样做会过度消耗节点的内存，从而让全节点的执行less efficient。在EIP150之后，SUICIDE指令发送给不存在的address将会正常收费，再之后，SUICIDE指令的gas消耗从0提升到5000。

### Case Study in Metering

#### 实验设置

硬件环境：谷歌云平台：4核心，8线程的Intel Xeon处理器，8GB的RAM，400MB/s的SSD。Ubuntu 18.04，Linux kernel version为4.15.0

挑选该设备是因为该设备最能够代表以太坊的全节点。

软件：使用的是以太坊C++客户端aleth。选择的原因：以太坊基金会发布的官方客户端之一（另一个是geth），并且该客户端没有运行时垃圾回收机制，因此可以是memory usage的测试指标更加使人信服。

为了计算运行时候的程序消耗，作者重载了new和delete operators，记录下来了EVM执行时候的allocation and deallocation。

本文使用的区块数据：Feb-28-2018 —— May-10-2018

#### 算术指令

本文测量了四种简单的arithmetic instruction

![image-20211218203858048](https://raw.githubusercontent.com/xuht724/blog-img/main/img/image-20211218203858048.png)

EXP指令和操作数有关，这里的gas cost使用了average，用的是51。

可以看到DIV指令的gas cost是EXP平均gas cost的10分之一，但是执行时间却比EXP要长，因此可以说明即使是简单的算术指令，当前的gas cost的分配都无法真正反映出指令执行的时间。

#### Gas和真实系统资源的消耗

文章分别随机选取了EIP150前后的100,000个块进行对比，查看Gas消耗设置以及实际资源消耗低皮尔逊相关系数。针对CPU，使用的是时钟时间；针对storage usage，计算的是EVM存储的新的word（256bits）的个数。

![image-20211218205356703](https://raw.githubusercontent.com/xuht724/blog-img/main/img/image-20211218205356703.png)

可以看到的是不管在EIP150前后，gas消耗和实际资源消耗相关性比较大的都是Storage+Memory。并且CPU time和gas消耗的相关性最低。这暗示只要没有使用storage，那么合约就可以又便宜又花费长时间去执行。

#### EVM指令执行时间的高方差

原因一是很多指令都是有参数的。比如说 EXTCODECOPY指令

原因二是许多指令要执行IO access，这就有kennel收到OS或者应用层的cache的影响。这类指令中具有最高方差的是BLOCKHASH指令，该指令的功能是获取block的hash值。并且可以获取目前区块的前256个区块的hash值。在执行该指令的时候，考虑到缓存的存在，可能会导致非常不同的执行时间。目前BlockHash的执行时间是相对便宜的，只要20 gas。这个问题曾经在[EIP210](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-210.md)被提出，但是并没有被接纳。

#### 内存缓存和EVM cost

文章针对page cache如何影响合约执行速度的实验设置如下：

1. 生成一个合约
2. 运行合约n次
3. 每一轮运行后都删除page cache再运行合约n次

作者选取了100个不同的合约，每个合约的平均gas为800000。文章的实验结果显示，在使用page cache的情况下，合约运行速度提高了24-33倍，其中超过半数的合约执行速度提升了27-29倍。这是因为LevelDB的缓存机制存在，LevelDB会讲一部分数据放到内存中，如果在内存中找不到该数据	才会到disk上进行查询。

在速度提升最多的合约中，BLOCKHASH，BALANCE，SLOAD时使用最频繁的指令。

需要注意的是如果使用的合约很小（比如100000gas），那么因为大部分的数据都停留在内存中，那么page cache并不会对执行合约有那么大的提升。

文章设置的关于block中的cache实验：

1. 生成n个block，每个block中是不同的合约
2. 按照顺序执行所有的block，测量执行是阿金
3. 重复执行m次，检测如何变化。

文章发现当n = 16的时候，每一次执行的速度不再变化。这说明之前执行的block的数据没有缓存。

这意味着如果一个部署了的合约函数在16个区块之后再次执行，它将会和第一次执行一样缓慢。

#### 总结

1. 即使是特别简单的算术指令，EVM的gas cost的设置都没有反应这是的资源使用
2. 尽管大多数指令有比较稳定的执行时间，但是有一些指令的执行时间的变化较大，这些指令通常涉及较多的IO operation
3. 客户端的page cache对于智能合约的执行速率有比较大的影响，并且分析了大概缓存存在的区块数目（这部分的实验并不是很严谨）
4. 总体来说，EVM的metering scheme的设计不能够反映出IO操作的复杂程度。

### 针对EVM metering scheme的攻击

文章将这种DoS攻击建模成寻找每秒gas消耗最低的程序。（即程序的执行时间长，但是gas消耗低）。该程序没有具体的意义，但是从以太坊的角度上来说它是valid。

文章提出了一种基于遗传算法的代码生成方法。分别介绍了遗传算法中的Initialization，Cross-over，Mutation等过程的伪代码。

文章分析了最终生成的代码，发现其中虽然BALANCE指令虽然经过EIP-150的调整，从20调整到400，但是该指令依然under-priced

![image-20211218222448432](https://raw.githubusercontent.com/xuht724/blog-img/main/img/image-20211218222448432.png)

该图描述了遗传算法生成的程序的吞吐量（每一秒消耗的gas值），可以看到在200代左右，吞吐量基本不变，大概是每一秒 110,000 的gas 消耗。

文章除了在aleth上做了实验，还在geth和Parity上做了实验（geth说后面要发布关于该攻击的修复，文章采用的版本是在发布修复之前的版本。）

在geth修复之前，geth采用超过75s的时间去执行10Mgas的malicious transaction，Parity表现较好，但是也平均花费了47s。相比之下，Parity和geth相比用更高但是更恒定的IO时间。

之后在geth的改进版上跑了该attack，该版本优化了storage的访问速度。在该修复过后，geth执行交易的速度提升了20倍，使geth可以足够解决这种攻击。

REA作为DoS攻击的一种形式：

1. 矿工花费更多的时间去验证交易，所有的全节点都需要花费更多的时间去同步区块状态，这可能导致用户不信任Eth，从而可能导致币价的降低。
2. 矿工可能通过生成一定数量的这种交易打包到自己的区块中，然后进行一定数量的自私挖矿。包含malicious transactions的区块可能会更慢的同步到全网，从而该矿工可以在下一轮挖矿过程中有先发优势。此外的话，攻击者也有可能通过发布这种攻击到全网从而影响ETH的币价，然后通过做空ETH来赚钱。

攻击的可行性：就是计算这样一笔transaction的实际transaction fee。

攻击的限制：以太坊开发者给的建议是运行一个全节点需要使用SSD，并且需要至少8GB的内存。然而目前只有很少的信息关于全节点具体的一个硬件参数，所以很难评估有多少节点会因此受到影响。不过以太坊官方对此非常重视。

作者最后将这一种攻击方式告诉给了以太坊开发者，得到了迅速的响应，并获得了5000USD的award。

### Towards a better approach

> Gas metering and pricing is a difficult but fundamental problem in Ethereum and other blockchains which use a similar approach to price contract execution.

短期的修复方式：

1. 提升IO操作的gas cost；比如EIP-150和EIP-2200
2. 设置更多级的缓存去降低IO的数目。然而这需要保持更多的数据在内存中，因此需要一个内存消耗和执行速度的trade-off。如何设计缓存机制也值得研究。目前做的一个尝试是在磁盘上维护一个账户状态的动态的snapshot，这样进行一个on-disk look up一个账户的开销只有$O(1)$，代价就是storage的增长。

长期的修复方式：等Ethereum2.0（说屁话）

目前社区已经有一些关于gas机制等研究：

1. changing the gas pricing mechanism
2. changing the way clients store data

未来最好的解决方法还是：使用**无状态客户端和无状态validation，核心的一个idea就是不强迫用户去存储全部的状态**，而是让用户发送transaction的同时发送transaction需要的数据和一个该数据正确的证明，（比如说Merkle Proof）。这可以**让客户端验证所有的交易而不需要去访问IO，从而使得执行和存储更加简单**，以此为代价的是创建交易的时候的复杂度的提升以及更高的带宽消耗。

另一个解决办法是sharding。虽然sharding并没有解决IO operation导致的gas pricing的基本问题，但是它确实让每个节点存储的状态更小了。
