---
layout: post
title: "论文: SAVIOR以Bug导向的混合模糊测试框架"
date: 2020-10-27 00:33:45
tags: ["fuzzing", "bug", "paper"]
---

## 简介

在混合执行有了较大进展的背景下, 针对于漏洞检测场景混合执行的效果并不乐观, 而作者主要认为有两个原因: 首先盲目选择种子用于混合执行以及不加重点地关注所有的代码, 其次就是混合模糊测试注重于让测试过程继续下去而非去检查内部的漏洞缺陷. 于是作者提出了SAVIOR, 它会优先考虑种子的混合执行并验证执行路径上所有易受攻击的程序位置. 

说白了, SAVIOR想解决混合模糊测试里乱选种子的行为, 并且希望以Bug为导向去选择种子. 那么具体的策略就是, 在测试前, SAVIOR会静态分析源代码并标记潜在的易受攻击位置. 此外, SAVIOR会计算每个分支可达的基本块集合, 在动态测试期间, SAVIOR优先考虑可以访问更重要分支的种子进行混合执行. 

除开上述说的能加快漏洞检测速度, SAVIOR还会验证混合执行引擎遍历过路径上标记的漏洞. 具体就是, SAVIOR综合了各个漏洞路径上的约束, 如果该约束在当前路径条件下可以满足, 那么SAVIOR就会求解该约束以构造输入进行测试. 否则SAVIOR会证明该漏洞在此路径上不可行. 

## 设计

### Bug驱动优先级

Bug驱动的关键是找到一个方法去评估某个种子在混合执行时能暴露出的漏洞数量, 这个评估取决于两个先决条件:

* R1 - 种子执行完后评估可访问代码区域的方法
* R2 - 量化代码块中漏洞数量的指标

针对R1, SAVIOR结合动静态分析去评估种子的可探索代码区域. 在编译期间, SAVIOR会从每个分支静态地计算可达的基本块集合, 在运行期间, SAVIOR则会在种子的执行路径上标记未探索的分支, 以及计算这些分支可访问的基本块集合. 

针对R2, SAVIOR则是利用UBSan来标注待测程序里的三种潜在的错误类型. 然后将每个代码区域中的标签计算为R2的定量指标. 同时SAVIOR也采用了一些过滤方法来删除UBSan的无用标签. 

### Bug导向验证

该技术可以确保在到达了漏洞函数路径上能进行可靠的漏洞检测. 从模糊测试处给定种子, SAVIOR会将其执行起来并沿执行路径提取各个漏洞标签. 之后SAVIOR会检查当前路径条件下的可满足性, 满足即漏洞有效. 

## 实现

SAVIOR由多个部分组成: 构建在Clang+LLVM之上的工具链, 基于AFL的Fuzzer, KLEE移植过来的混合执行引擎和负责编排的协调器. 

SAVIOR的编译工具链可用于漏洞标记, 控制流的可达性分析以及不同组件的构建. 

漏洞标记则是基于UBSan, 当然UBSan有一些不如意的地方, SAVIOR对其进行了一些调整. 

可达性分析用于计算CFG中每个基本快可到达的漏洞标签的数量. 它分为两个阶段, 第一步是类似SVF方法去构建过程间CFG, 其首先会为每个函数构建过程内CFG再通过调用关系建立过程间的关系. 为了解决间接调用, 算法会反复执行Andersen的指针分析, 以防止SAVIOR丢失间接调用的函数别名信息, 也使得优先级划分不会漏算漏洞标签数量. 此外通过检查CFG, SAVIOR还提取了基本块和子对象之间的边, 以便后续在协调器的进一步使用. 第二步则是计算过程间CFG中每个基本块可到达的代码区域, 并计算这些区域中UBSan标记的数量, 以此作为该基本块的优先级指标. 

组件构建则是去编译三个binary: 一个用于fuzzer的binary, 一个用于协调器的SAVIOR-binary, 一个则是用于混合执行引擎的LLVM bitcode. 

协调器则是用于挑选优先级高的种子, 以及一些后续处理.  混合执行引擎则采取了一些策略去解决约束问题. 