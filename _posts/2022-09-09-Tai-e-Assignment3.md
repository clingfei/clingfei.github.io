---
declare: true
title: Tai-e Assignment 3 死代码检测
categories: [程序分析]
tags:
- 程序分析


---

> 『 无论你是玩游戏多，还是成绩不太好，觉得自己没有别人毕业有优势，这都是因为浮躁、去比较产生的事情。这门课，老师希望你重新的审视自己，不断地认识自己、挖掘自己，看看真的什么东西能让你你快乐起来，什么东西能让你花时间去搞。哪怕你以后只是开了一家奶茶店，哪怕是一个和计算机毫无相关的行业，也希望你能从这门课中清楚的认识到自己喜欢的是什么，你不是没有别人优秀，你只是选择了你喜欢的事情。 』 					——李樾

## Overview

死代码指的是程序中不会执行的代码或是执行结果不会被其他计算过程用到的代码。本次作业所要检测的代码只包括两种：不可达代码和无用赋值

### 不可达代码

考虑两种不可达代码，即控制流不可达和条件分支不可达。

控制流不可达代码指的是不存在从程序入口到该段代码的控制流路径。例如:

```java
int controlFlowUnreachable() {
    int x = 1;
    return x;
    int z = 42; // control-flow unreachable code
    foo(z); // control-flow unreachable code
}
```

检测方式：遍历cfg，标记可达代码，剩下的就是不可达代码。

### 分支不可达代码

Java中仅有两种分支语句：switch和if。

对于if语句来说，如果条件值是常数，那么必然会产生不可达代码。例如：

```java
int unreachableIfBranch() {
    int a = 1, b = 0, c;
    if (a > b)
        c = 2333;
    else
        c = 6666; // unreachable branch
    return c;
}
```

对于switch语句，如果条件是常数那么是否会产生不可达代码取决于case和default的具体实现。例如：

```java
int unreachableSwitchBranch() {
    int x = 2, y;
    switch (x) {
        case 1: y = 100; break; // unreachable branch
        case 2: y = 200;
        case 3: y = 300; break; // fall through
        default: y = 666; // unreachable branch
    }
    return y;
}
```

检测方式：对if语句，判断条件值是否为常量，如果是常量，则根据常量值来选择将then或else语句加入死代码。对于switch语句，同样判断条件值是否为常量，switch相比if更为复杂，将在后文进一步阐述。

### 无用赋值

局部变量被赋值后没有被后续语句读取，那么将不会影响后续的计算结果，因而可以被消除：

```Java
int deadAssign() {
    int a, b, c;
    a = 0; // dead assignment
    a = 1;
    b = a * 2; // dead assignment
    c = 3;
    return c;
}
```

检测方式：进行活跃变量分析，如果左侧的变量是not live的，那么这条语句就是死代码。存在一种特殊情况：右边是函数调用，过程内分析无法判断函数是否对变量值产生影响，因此将函数调用统一视为活跃代码。

## 具体设计

采用广度优先搜索来遍历整个控制流图。对于控制流图的每一条边，检查其target结点是否已经在isReached（HashSet，其中存储已经访问过的statement）中或已经被检测出为deadCode，前者是为了防止在环路中陷入死循环，后者是为了防止出现漏报。

无用赋值检测非常容易，因为已经实现了活跃变量检测，因此只需要对赋值语句检查左侧变量是否在当前语句的活跃变量集合中。

对于if语句，首先需要通过ConstantPropagation.evaluate计算条件值，并判断是否为常量。如果是常量0，那么if-then语句将不会被执行， 将其加入deadCode。注意此时if语句的后续两条边IF_TRUE和IF_ELSE已经加入了队列，但是在将IF_TRUE加入到deadCode后，队列中的这一语句将会被忽略，因此这一条件分支的代码也将不会被加入到isReached集合中，并在后续的分支不可达检测中被加入到deadCode中。

对于switch语句，类似地，计算条件值，并判断执行哪一条case语句。switch的难点在于，即使case条件没有满足，如果上一条满足条件的case没有break，那么该条指令依然可以执行。例如:

```java
int unreachableSwitchBranch() {
    int x = 2, y;
    switch (x) {
        case 1: y = 100; break; // unreachable branch
        case 2: y = 200; break;
        case 3: y = 300;  // fall through
        default: y = 666; // reachable branch
    }
    return y;
}
```

但是，上一条case语句只有全部执行完之后才能够判断是否有break跳转，看起来是深度优先，而我使用的是广度优先，那么要如何将两者融合在一起？

比较一下二者的控制流图，可以发现两者的主要不同在于在case8后有一条到default的路径（第一张图为break，第二张图为无break，由于大小原因，仅截取部分）：

![image-20220909202258793.png](https://s2.loli.net/2022/09/09/yBhEilTce8nC4dG.png)

![image-20220909202544515.png](https://s2.loli.net/2022/09/09/BUV7Jpa6qyC5kuK.png)

因此，我们可以将当前switch的所有指向case或default的case语句从待遍历队列的首部删掉，而仅仅插入case与switch条件值一致的语句，并按照控制流图继续向下遍历，如果该case没有break，那么后续的case或default最终也可以作为当前case的后继而遍历到。而没有被遍历到的语句将在可达性检测中加入到deadCode中。

最后的可达性检测，依次遍历控制流图的每个结点并判断是否被访问过，如果没有被访问过，则说明是不可达代码，加入到deadCode中。

## 总结

deadCode是课程设置中DFA分析的最后一个实验，基于前面的常量传播分析和活跃变量分析，主要的难度在于比较抽象，使用Graphviz来绘制分析出的cfg图能够更好的帮助debug和理解测试用例。

