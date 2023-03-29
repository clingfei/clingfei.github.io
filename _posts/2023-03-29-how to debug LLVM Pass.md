---
declare: true
title: 如何使用gdb调试LLVM Pass？
categories: [LLVM,gdb]
tags:
- LLVM 
- gdb

---

---

## 前置知识

**本文使用的LLVM 版本为10.0，使用LegacyPassManager管理Pass。**

由于LLVM Pass被编译为运行时动态加载的链接库，因此与普通的可执行文件不同，无法使用gdb在加载文件之后设置断点。

LLVM Pass编译的.so文件通过opt动态加载执行，opt是模块化的LLVM优化器和分析器，使用LLVM源文件作为输入，在源文件上运行指定的优化或分析程序，然后输出优化/分析后的结果。使用opt运行Pass的基本命令为：

```
opt -load pass.so -passname sourcefile
```

## 为Pass设置断点

1. 由于Pass通过opt动态加载，因此首先使用gdb加载opt：

   ``` shell
   gdb opt
   ```

   首先需要确保LLVM编译时开启了编译选项，如果没有开启，使用gdb加载opt时会出现：

   ```
   Reading symbols from ./opt...
   (No debugging symbols found in ./opt)
   ```

   此时需要从源码重新编译LLVM，并将新编编译出的bin目录添加到环境变量中。

   而正确开启了编译选项的opt在gdb中的输出为：

   ![image-20230329155838622.png](https://s2.loli.net/2023/03/29/jgxJkPWR89UYB3F.png)

2. 为了给我们想要debug的Pass设置断点，我们首先需要找到加载Pass后Pass执行之前的代码设置断点，以得到给Pass设置断点的机会。

   LLVM文档中给出的断点可以设置为

   ```
   break llvm::PassManager::run
   ```

   stackoverflow中给出的断点可以设置为

   ```
   break llvm::Pass::preparePassManager
   ```

   经过测试，在LLVM 10.0中无法命中llvm::PassManager::run这一断点，而llvm::Pass::preparePassManager可以多次命中。

3. 加载Pass：

   ```
   run -load pass.so -passname xxxx.bc
   ```

   程序将会暂停在llvm::Pass::preparePassManager这一断点处，之后可以直接为Pass添加断点，并将preparePassManager这一断点移除，进入正常的debug流程。

## 遇到的问题：

1. 在重新编译LLVM后，使用opt加载Pass报错：

   ```
   : CommandLine Error: Option 'help-list' registered more than once!
   LLVM ERROR: inconsistency in registered CommandLine options
   ```

   推测是由于Debug版本的clang和Release版本的clang编译出的Pass符号不同，导致在使用opt加载时命令行参数出现冲突，通过重新设置环境变量、重新编译Pass后解决

2. 使用gdb调试，加载opt并设置断点后，通过run -load加载Pass出错：

   ```
   Error opening 'passname.so': /path/to/passname.so: undefined symbol: _ZN4llvm24DisableABIBreakingChecksE
     -load request ignored.
   ```

   通过c++filt查看该符号，得到：

   ```
   $ c++filt _ZN4llvm24DisableABIBreakingChecksE
   llvm::DisableABIBreakingChecks
   ```

   Google得知，在llvm/Config/abi-breaking.h中EnableABIBreakingChecks和DisableABIBreakingChecks的定义如下：

   ```C++
   namespace llvm {
   #if LLVM_ENABLE_ABI_BREAKING_CHECKS
   extern int EnableABIBreakingChecks;
   LLVM_HIDDEN_VISIBILITY
   __attribute__((weak)) int *VerifyEnableABIBreakingChecks =
       &EnableABIBreakingChecks;
   #else
   extern int DisableABIBreakingChecks;
   LLVM_HIDDEN_VISIBILITY
   __attribute__((weak)) int *VerifyDisableABIBreakingChecks =
       &DisableABIBreakingChecks;
   #endif
   }
   ```

   从上述代码可以得到，通过设置编译选项，将LLVM_ENABLE_ABI_BREAKING_CHECKS设置为false就会产生对于DisableABIBreakingChecks这一符号的定义。

   根据LLVM文档中对**LLVM_ABI_BREAKING_CHECKS**的说明:

   Used to decide if LLVM should be built with ABI breaking checks or not. Allowed values are WITH_ASSERTS (default), FORCE_ON and FORCE_OFF. WITH_ASSERTS turns on ABI breaking checks in an assertion enabled build. FORCE_ON (FORCE_OFF) turns them on (off) irrespective of whether normal (NDEBUG-based) assertions are enabled or not. A version of LLVM built with ABI breaking checks is not ABI compatible with a version built without it.

   因此只要在编译时将LLVM_ABI_BREAKING_CHECKS设置为FORCE_OFF即可关闭ABI breaking check。在cmake中添加`-DLLVM_ABI_BREAKING_CHECKS=FORCE_OFF`重新编译Pass，即可成功在gdb中运行。

## Reference

1. [Writing an LLVM Pass — LLVM 17.0.0git documentation](https://llvm.org/docs/WritingAnLLVMPass.html#introduction-what-is-a-pass)
2. [Debugging an llvm pass with gdb - Stack Overflow](https://stackoverflow.com/questions/2226112/debugging-an-llvm-pass-with-gdb)
3. [Building LLVM with CMake — LLVM 17.0.0git documentation](https://llvm.org/docs/CMake.html)
4. [Using LLVM headers gives a compilation error : learncpp (reddit.com)](https://www.reddit.com/r/learncpp/comments/gbmlj5/using_llvm_headers_gives_a_compilation_error/)

