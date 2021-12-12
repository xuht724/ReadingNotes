# Towards Saving Money in Using Smart Contracts

## 作者

Ting Chen†‡, Zihao Li†, Hao Zhou‡, Jiachi Chen‡, Xiapu Luo‡§, Xiaoqi Li‡ , Xiaosong Zhang†

## 阅读笔记

### Abstract

提出了24种gas不高效的范式，并且设计了一个名为 GasReducer 的工具来自动检测这些智能合约的字节码并且将其替换成更高效的代码。

### Introduction

交易执行的费用的计算方式是：gas_price * gas_cost

gas_price由当前市场决定

gas_cost 为本次指令执行所需要消耗的gas，每个指令的消耗由以太坊黄皮书给出

本篇论文的贡献：

- 提出了24种gas不高效的pattern，并且提出了对应的3种检测方案
- 设计了一个tool：GasReducer，用来分析字节码并且将其转换为efficient code
- 对于部署的合约进行了实验

### Anti-pattern

智能合约中anti-pattern的定义：一个 指令序列 (Operation sequence ) OS是 Anti-pattern（反面范式）的：存在另一个指令序列和该指令序列消耗等同的gas，但是语义逻辑上是相等的。

本篇论文获取指令序列的方法：添加输出指令日志的代码到EVM中，即放到了Apply-transaction() （/core/state_processor.go函数）中

每一个非0的bytecode在部署的时候要消耗68单位的gas。如SWAP1为0x90，需要消耗136单位的gas

检验出来的anti-pattern（提出了24种，文章只列出了其中的5种）

```c
{swap(x),swap(x)}->delete
{M consecutive} jumpdests -> {jumpdests} //jumpdests表示有效跳转
{OP,pop} -> {pop} //此处OP表示{iszero, not, balance, calldataload,extcodesize, blockhash, mload, sload}等需要消耗栈顶元素的元素，计算出结果再压入栈顶的操作
{swap1,swap(x),OP,dup(x),OP}->{dup2,swap(x+1),OP,OP}//此处的OP表示{add,mul,and,or,xor}等指令
{OP,stop}->{stop}//此处的OP表示除了{jumpdest,jump,jumpi和其他可以改变storage}
```





