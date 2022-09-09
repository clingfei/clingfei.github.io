---
declare: true
title: Tai-e Assignment 2 常量传播和Worklist求解器
categories: [程序分析]
tags:
- 程序分析
---

> 『 无论你是玩游戏多，还是成绩不太好，觉得自己没有别人毕业有优势，这都是因为浮躁、去比较产生的事情。这门课，老师希望你重新的审视自己，不断地认识自己、挖掘自己，看看真的什么东西能让你你快乐起来，什么东西能让你花时间去搞。哪怕你以后只是开了一家奶茶店，哪怕是一个和计算机毫无相关的行业，也希望你能从这门课中清楚的认识到自己喜欢的是什么，你不是没有别人优秀，你只是选择了你喜欢的事情。 』 					——李樾

## Overview

实现int类型的常量传播，boolean、byte、char和short类型同样被视为int，其他类型可以被忽略。

语句处理只需要关注赋值语句，主要包括三种：

- 常量赋值， x = 1
- 变量赋值，x = y
- 表达式赋值，x = a+b

在表达式中可能会出现的二元运算符：

| 运算类型   |      运算符       |
| ---------- | :---------------: |
| Arithmetic |    `+ - * / %`    |
| Condition  | `== != < > <= >=` |
| Shift      |    `<< >> >>>`    |
| Bitwise    |      `| & ^`      |

其中Java中逻辑运算符以分支跳转形式实现，因此不需要处理。

## 设计

### 初始化

在newBoundaryFact中，对于具有参数的函数来说，应该将参数插入到CPFact中，并将其值设置为NAC，因为在函数看来参数的值未知，所以是NAC（not a constant)。而如果返回空CPFact，变量的值将会被视为UNDEF。

### meetInto

这里的meetInto使用了与Assignment类似的设计，调用meetValue函数来将target和fact两者合并。

根据PPT中的说明，合并规则为:

```
NAC ^ v = NAC
UNDEF ^ v = v
c ^ c = c
c1 ^ c2 = NAC
```

### evaluate

计算给定表达式的值，容易出错的地方在于%和/在除数是0时，无论另一个操作数的值是什么都要返回Undef。

### transferNode

常量传播分析的语句为DefinitionStmt，因此首先需要判断stmt的类型，并将stmt显式转换为DefinitionStmt，可以使用模板来实现自动类型推导：

```java
DefinitionStmt<?, ?> definitionStmt = (DefinitionStmt<?, ?>)stmt;
```

### doSolveForward

常量传播使用前向分析的workList算法，将workList初始化为队列，并将控制流图中的结点全部加入到workList。workList中每个结点所有前驱的OUTFact集合的交集作为该节点的IN集合，并且通过比较新的OUT与原本的OUT是否一致来判断是否将该节点的后继加入到workList。由于OUT的改变仅依赖于IN的改变，所以最终会收敛到一个不动点，算法结束，常量分析完成。

## 后记

完成Assignment3之后才想起来没有写Assignment2的总结，debug时遇到的很多细节已经模糊，只记得过程非常痛苦。Assignment 2是所有8个实验中通过率最低的一个，截至我完成时通过率只有7%。骥恺给了我两个参考文档：[Testcase about A2 · Issue #2 · pascal-lab/Tai-e-assignments (github.com)](https://github.com/pascal-lab/Tai-e-assignments/issues/2)和 [SPA-Freestyle-Guidance/Assignment 2.md at main · RicoloveFeng/SPA-Freestyle-Guidance (github.com)](https://github.com/RicoloveFeng/SPA-Freestyle-Guidance/blob/main/assignments/Assignment 2.md)以及自己通过测试的代码，经过反复比对输出结果才得以在较短的时间内通过这个实验，即使如此依然WA多次。截至今天，完成了全部DFA的学习，即将进入过程间分析，希望能够顺利完成全部课程。

