# GasChecker: Scalable Analysis for Discovering Gas-Inefficient Smart Contracts

## 作者

Ting Chen†‡, Zihao Li†, Hao Zhou‡, Jiachi Chen‡, Xiapu Luo‡§, Xiaoqi Li‡ , Xiaosong Zhang†

## 阅读笔记

### Abstract

提出了第一个用于自动监测智能合约中的 gas-inefficient 的部分的代码，并且对于现有已经部署的代码进行了分析。

提出了 10 中 gas 不高效的范式

提出一种新的基于符号化执行（Symbolic Execution）的方式去分析智能合约的字节码

### Introduction

以太坊引入 Gas 机制的原因：为了防止资源的滥用（比如 DoS 攻击来瘫痪节点的资源）

Gas Inefficient smart Contract：消耗了比必要 gas 更多的 gas 就是 gas-inefficient（有优化空间的 Gas）

会因为本项工作收益的人：智能合约的开发者，能够通过避免一些 gas-inefficient 的范式来优化自己的智能合约；编译器的开发者，可以通过替换掉 gas 不高效的代码来优化编译过程；以太坊的开发者可以通过开发 JIT Compiler 并且将其嵌入到 EVM 中；激励一种新的服务：将 gas 不高效的转变成 gas 高效的

### Background

以太坊虚拟机是一个栈式虚拟机。

在 ETH 的设计中，有一些操作是便宜的：ADD，AND，EQ，POP，因为这些都只是一些简单的栈操作，有一些操作是很昂贵的，SSTORE 用来更新存储的值，CREATE 用来创建一个新的合约。

```solidity
contract ExpensiveStorage{
	uint num;//in storage
	function setVar(uint x){
		for(uint i = 0;i<x;i++){
			num++; //expenseive
		}
	}
};//x次读，x次写

contract CheapStorage{
	uint num;
	function setVar(uint x){
		uint y = num;
		for(uint i=0;i<x;i++){
			y++;//cheap
		}
		num=y;
	}//1次读写
}
```

需要注意的是从传统的编译器优化的角度是不会考虑对于上述代码的优化的。

1. 在传统的程序中，对于局部变量的访问和全局变量的访问速度上是没有区别的
2. 在第二个合约中，需要使用更多的内存存储本地变量
3. 第二个合约要包含更多的指令，所以要浪费更多的磁盘空间存储 Code，更多的内存会被消耗来加载 code，更多的指令会被执行

### Gas-Infeeicient programming patterns

#### Useless Code

##### Pattern1 Opaque predicate

指的是一个 comparison 只有明确的一个结果（true or false），那么这个 comparison 应该被移除掉

```solidity
function p1(uint x){
	if(x>5){
		if(x>1)
	}
}
```

##### Pattern2 Dead Code

指的是一些在运行的时候永远不会被访问到的代码

```solidity
function p2(uint x){
	if(x>5){
		//Opaque predicate
		if(x*x<20){
		}
	}
}
```

#### LOOP

##### Pattern3 Expensive Operation in Loop

循环中的昂贵操作，比如说下面这个合约的代码。

通过将昂贵的操作移动到循环之外，可以降低 gas 的消耗。然而优化后的字节码的规模就有可能被提升

```solidity
contract ExpensiveStorage{
	uint num;//in storage
	function setVar(uint x){
		for(uint i = 0;i<x;i++){
			num++; //expenseive
		}
	}
};//x次读，x次写
```

##### Pattern4 Fusible Loops

可融合的循环

如果两个循环代码可以被放到一个循环中，那么部署智能合约的 gas 消耗就能被降低，这是因为合约字节码将会变小，从而使得调用该合约所需要的字节码的花费会减少很多。也一定程度上会降低 local Vairable 的生成

```solidity
function p4(uint x){
	uint m=0;
	uint n=0;
	for(uint i=0;i<x;i++) m+=i;
	for(uint j=0;j<x;j++) v-=j;
}

function p4Improve(uint x){
	uint m = 0;
	uint n = 0;
	for(uint i = 0;i<x;i++){
		m+=i;
		v-=j;
	}
}
```

##### Pattern5 Repeated computation in a loop

循环中的重复计算。比如下面的这个合约，x 和 y 的值在每一次循环的时候都会被 Load，改进方法就是将 x 和 y 在循环之外进行 Load，用临时变量存起来

```solidity
uint x=1;
uint y=1;
function p5(uint x, uint y){
	uint sum = 0;
	for(uint i = 0;i<=k;i++){
		sum = sum + x + y;
	}
}

uint x=1;
uint y=1;
function p5Improve(uint x, uint y){
	uint sum = 0;
	uint local_x = x;
	uint local_y = y;
	for(uint i = 0;i<=k;i++){
		sum = sum + local_x + local_y;
	}
}
```

##### Pattern6 Unilateral Comparison in a Loop

这种编程范式的错误表示的是在循环中出现了不会因为循环而改变的固定的比较结果，这样的比较应该被提升到循环之外去，比如下面这样的合约。

```solidity
function p6(uint x, uint y) returns(uint){
	for(uint i = 0;i<100;i++){
		if(x>0) y+=x;
	}
	return y;
}

function p6Improve(uint x, uint y) returns(uint){
	if(x>0){
		for(uint i = 0;i<100;i++){
			y += x;
		}
	}
	return y;
}
```

#### Wasted Disk Space

##### Pattern7 Redundant SSTORE

这个范式表明了一个在定义之后从未被使用到的 Storage 空间，编译器应该检测到多余的变量定义

#### Gas-inefficient Operation Sequence

##### Pattern8 SWAP1/DUP2/SWAP1

在以太坊虚拟机中，SWAP 交换了栈中的元素，DUP 操作复制了栈中的某个元素。那么根据下图可以看到的是 SWAP1，DUP2，SWAP1 和 DUP1，SWAP2 操作得到的最后结果是一样的 。需要注意的是 SWAP 和 DUP 操作都是消耗 3 单位的 gas。

除了调用时候节约的 gas 之外，上面这一种 gas 不高效的字节码是 0x908190，下面这一种的字节码是 0x8091，因此和 gas 不高效的范式相比，下面这一种在部署的时候可以节约 68 单位的 gas。应该说，每一种节约了 operation 数的都

![](https://raw.githubusercontent.com/xuht724/blog-img/main/img/1636697023.png)

##### Pattern9 PUSHx/POP

PUSHx 操作指的是将一个 xbytes 大小的数据放到栈中，x 最大为 32，因此 PUSHx/POP 操作并不会影响栈中的数据，因此对于这样的序列应该在编译的时候被移除以节约 gas。通过移除这样的操作，可以在执行合约函数的时候节约 5 单位的 gas。

##### Pattern10 PUSH1/NOT

NOT 指令的作用是将栈中最顶层的元素进行按位取反，PUSH1 指的是将一个 1byte 的数据压入栈中，这个 1byte 的数据是一个不变量，因此这个数据可以提前计算出来，从而 PUSH1/NOT 可以被一个单独的 PUSH 替代。通过该方法可以节省 3gas 的调用费和 68 单位的部署费用。

### GasChecker

这部分主要讲述的是为什么使用符号分析技术，以及它的 Map Reduce 程序是怎么写的，以及在 SE 中如何监测上述提出的这几种 pattern 的。由于目前对于 SE 了解不多，暂时按下先不写。

### Results of The empirical study

数据集：下载收集了 2015 年 7 月 30 号到 2017 年 6 月 10 号的合约代码，并且按照一定标准（some criteria）挑选出了代表性的合约。下载代码的方式式调用了 ETH 的 GetCode 的 API。下载了 599,934 个合约的字节码。最终挑选出了 1500 个。

挑选的标准：unique，803 个 Open Source Code（在 etherscan），500 个 most popular，197 个随机挑选。

![](https://raw.githubusercontent.com/xuht724/blog-img/main/img/%E6%9C%AA%E5%91%BD%E5%90%8D1636701596.png)
