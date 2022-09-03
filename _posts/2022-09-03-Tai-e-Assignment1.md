---
declare: true
title: Tai-e Assignment 1 活跃变量分析和迭代求解器
categories: [程序分析]
tags:
- 程序分析

---

> 『 无论你是玩游戏多，还是成绩不太好，觉得自己没有别人毕业有优势，这都是因为浮躁、去比较产生的事情。这门课，老师希望你重新的审视自己，不断地认识自己、挖掘自己，看看真的什么东西能让你你快乐起来，什么东西能让你花时间去搞。哪怕你以后只是开了一家奶茶店，哪怕是一个和计算机毫无相关的行业，也希望你能从这门课中清楚的认识到自己喜欢的是什么，你不是没有别人优秀，你只是选择了你喜欢的事情。 』 					——李樾

## Overview

作业1是为Java实现活跃变量分析，其中需要的程序分析接口和数据流信息的表示、程序流图等结构都已经在Tai-e框架中提供，只需要补全几个关键部分。

一共需要实现6个方法：

LiveVariableAnalysis:

- SetFact newBoundaryFact(CFG)
- SetFact newInitialFact()
- void meetInto(SetFact,SetFact)
- boolean transferNode(Stmt,SetFact,SetFact)

Solver:

- Solver.initializeBackward(CFG,DataflowResult)
- IterativeSolver.doSolveBackward(CFG,DataflowResult)

在一开始看到LiveVariableAnalysis时可能会有一头雾水的感觉，我认为从逻辑上来说先实现Solver再实现LiveVariableAnalysis中的几个方法更加合理。

## Algorithm

![Iterative Algorithm](http://tai-e.pascal-lab.net/pa1/iter-alg.png)

我们所需要实现的活跃变量分析算法的伪代码如上图所示，首先执行初始化将CFG中每个结点的IN set和OUT set置为空集。然后对于CFG中的每个基本块B，分别使用meetInto和transferNode计算OUT[B]和IN[B]，一直到CFG中每个basic block的IN set不再改变为止。框架中对这一算法进行了简化，将每条语句作为basic block来处理。

### 结点初始化

`public SetFact<Var> newBoundaryFact(CFG<Stmt> cfg)`用于对exit结点初始化，其参数cfg在assignment1中并没有被使用。

`public SetFact<Var> newInitialFact()`用于对其他结点初始化。

这两个函数实际上功能相同，都只需要返回一个为空的Var集合

### 更新OUT集合

按照实验手册，meetInto的行为是将fact集合合并入target集合，其中target为OUT[B]，fact为IN[S]，S为B的后继。直接将后继的IN集合合并到B的OUT集合可以避免集合的多次拷贝，提高运行效率。对应地我们也需要对算法作一定改动：在对IN集合初始化时一并将set初始化。

SetFact中给出了两个Set合并的接口，直接调用即可。

### 更新IN集合

transferNode函数与前面几个函数不同，返回类型boolean用于在迭代时判断是否应该终止迭代，因此在结束时应该判断新的IN集合与旧的IN集合是否相同。

在这里我犯了一个错误：SetFact给出了copy和set两个方法，我误以为`new_in = out.copy()`就创建了out的一个名为new_in的副本，但是实际上copy执行的是浅拷贝，对new_in的修改实际上一并修改了out，从而影响后面的分析结果。正确的拷贝方法应该是使用set：

```java
SetFact<Var> new_in = new SetFact<>();
new_in.set(out);
```

同时我们执行的是活跃变量分析，只需考虑Var类型，而在语句中可能存在立即数等各种不同类型，因此需要对`stmt.getDef`和`stmt.getUses`的返回结果进行过滤，否则在类型转换时会抛出异常。开始时我使用`use.getClass().toString()`判断是否为class pascal.taie.ir.exp.Var，后来了解到使用java的instanceof特性可以更为优雅地实现。在断定类型为Var后，通过强制类型转换插入到In和OUT中实现更新，完整的代码如下:

```java
public boolean transferNode(Stmt stmt, SetFact<Var> in, SetFact<Var> out) {
        // TODO - finish me
        // IN = use U (OUT - def)
        SetFact<Var> new_in = new SetFact<>();
        new_in.set(out);
        Optional<LValue> def = stmt.getDef();

        if (def.isPresent() && def.get() instanceof Var) {
            new_in.remove((Var) def.get());
        }
        List<RValue> uses = stmt.getUses();
        for (RValue use : uses) {
            if (use instanceof Var)
                new_in.add((Var)use);
        }
        if (in.equals(new_in)) {
            return false;
        } else {
            in.set(new_in);
            return true;
        }
    }
```

### 实现迭代求解器

initializeBackward执行初始化过程，将CFG中的每个结点及利用上述初始化函数初始化后的IN和OUT集合插入到result中即可。

doSolveBackward函数则是算法进行迭代的关键。内层循环遍历CFG的每个结点，分别调用meetInto和transferNode函数来更新IN和OUT两个集合。为了判断IN有没有发生改变，我引入了一个bool类型的变量flag，每次与transferNode的返回值相或，这样一旦flag为false，说明没有发生任何一个结点的IN集合发生改变，跳出循环，分析结束。

在这里我踩了一个大坑。由于PPT中所展示的手动分析过程从CFG的末尾结点开始向上查询，因此我在开始时定义了一个队列，用来实现倒序遍历CFG，但是实际上对本次实验来说并无必要。我认为倒序和正序的主要区别在于，正序遍历时第一趟前面几个结点的IN和OUT两个集合没有发生改变，而倒序从底部出发，一次遍历就可以更改程序路径上的所有节点，因此倒序的效率应该更高，但正序的代码实现相比倒序更加简洁直观，因此最后还是采用了正序遍历。

## Reference

[SPA-Freestyle-Guidance/Assignment 1.md at main · RicoloveFeng/SPA-Freestyle-Guidance (github.com)](https://github.com/RicoloveFeng/SPA-Freestyle-Guidance/blob/main/assignments/Assignment 1.md)

[作业 1：活跃变量分析和迭代求解器 | Tai-e (pascal-lab.net)](http://tai-e.pascal-lab.net/pa1.html)

