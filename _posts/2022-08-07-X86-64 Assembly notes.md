---
declare: true
title: x86-64 Assembly notes
categories: [x86-64, Assembly]
tags:
- X86-64 Assembly 
---

---

## Registers

用于保存前六个参数和返回值的寄存器是callee-owned，即被调用者所有。被调用者可以任意修改和读取这些寄存器的值。如果调用者需要继续使用`%rax`中的值，那么需要在安全的位置保存一份副本（因为无法保证被调用者不会修改`%rax`中的值，其他也同理。）如果被调用者希望使用caller-owned寄存器，必须首先保存寄存器的值，并在函数结束前将其恢复。

| Register                      | 用途                       |
| ----------------------------- | -------------------------- |
| %rax                          | 函数返回值， callee-owned  |
| %rdi                          | 1st argument, callee-owned |
| %rsi                          | 2nd argument, called-owned |
| %rdx                          | 3rd argument, called-owned |
| %rcx                          | 4th argument, callee-owned |
| %r8                           | 5th argument, callee-owned |
| %r9                           | 6th argument, callee-owned |
| %r10/%r11                     | 临时使用， callee-owned    |
| %rsp                          | 栈指针，caller-owned       |
| %rbx/%rbp/%r12/%r13/%r14/%r15 | 局部变量，caller-owned     |
| %rip                          | 指针寄存器                 |
| %eflags                       | 状态/条件位寄存器          |

`%rbp`是x86_64的栈基底指针，`%rsp`是x86_64的栈顶指针

## Addressing modes

寻址方式主要包括直接寻址和间接寻址等。以`mov`指令为例：

```assembly
mov src, dst
movl $1, 0x604892  #直接寻址，目标操作数为目的地址
movl $1, (%rax)	   #寄存器间接寻址，目标操作数存储在寄存器%rax中
movl $1, -24(%rbp) #目标地址 = (%rbp) - 24
movl $1, 8(%rsp, %rdi, 4) # 目标地址 = %rsp+8+%rdi*4
movl $1, (%rax, %rcx, 8) # 目标地址 = %rsp + %rcx * 8
movl $1, 0x8(, %rdx, 4)  # 目标地址 = 0x8 + %rdx * 4
movl $1, 0x4(%rax, %rcx) # 目标地址 = 0x4 + %rax + %rcx
```

## Common instructions

指令的后缀(`b`, `w`, `l`, `q`)指明了操作数的位宽，如果通过操作数可以推断出位宽则可以省略后缀。例如`%eax`必须是4字节。某些指令可能有不止一个后缀，例如`movzbl`表示将1字节的源操作数移动到4字节的目标地址。

在目标操作数是子寄存器时，通常情况下只有子寄存器中的特定字节会被写入，但32bit指令会将目标寄存器的高位置零。

#### Mov and lea

`mov`将源操作数中的数据拷贝到目的操作数中。源操作数可以是立即数、寄存器或内存地址，目的操作数可以时寄存器或内存地址。后缀b,w,l,q代表了拷贝的数据位宽。

`lea`指令的源操作数是一个内存地址，并将计算后的源操作数的地址拷贝到目的操作数中。**lea的作用是计算地址，而没有移动地址中存放的数据**，

`movs`和`movz`用于从位宽较小的寄存器拷贝到更长的寄存器中，指明寄存器高位填充的方式，其中`movs`进行符号扩展，即将最高有效位复制到寄存器的高位，`movz`进行零扩展，即将高位直接置零。(`mov`在使用32bit的寄存器作为源操作数时默认进行零扩展)

`cltq`指令是特殊的`movs`指令，源操作数是`%eax` ,目的操作数是`%rax`，用于在`%rax`上进行符号扩展。

#### Arithmetic and bitwise operations

二元运算指令的第二个操作数既是源操作数又是目标操作数。一元运算指令的操作数既是目的地址又是源地址。

#### Branches and other use of condition codes

`%eflags`作为状态寄存器，其中 保存了一系列用于条件判断的标志位。ZF为0标志位，SF作为符号标志位，OF为溢出标志位，CF为进位位。用法:

```assembly
cmpl op2, op1    # computes result = op1 - op2, discards result, sets condition codes
test op2, op1    # computes result = op1 & op2, discards result, sets condition codes

jmp target    # unconditional jump
je  target    # jump equal, synonym jz jump zero (ZF=1)
jne target    # jump not equal, synonym jnz jump non zero (ZF=0)
js  target    # jump signed (SF=1)
jns target    # jump not signed (SF=0)
jg  target    # jump greater than, synonym jnle jump not less or equal (ZF=0 and SF=OF)
jge target    # jump greater or equal, synonym jnl jump not less (SF=OF)
jl  target    # jump less than, synonym jnge jump not greater or equal (SF!=OF)
jle target    # jump less or equal, synonym jng jump not greater (ZF=1 or SF!=OF)
ja  target    # jump above, synonym jnbe jump not below or equal (CF=0 and ZF=0)
jae target    # jump above or equal (CF=0)
jb  target    # jump below, synonym jnae jump not above or equal (CF=1)
jbe target    # jump below or equal (CF=1 or ZF=1)
```

#### setx and movx

`setx`将目标寄存器根据条件x设置为0或1，其目标操作数只能为单字节的子寄存器，例如`%rax`的低字节`%al`。`cmovx`根据条件x选择是否执行mov指令，源操作数和目的操作数都只能是寄存器。其中x是条件变量的占位符。

```assembly
sete dst           # set dst to 0 or 1 based on zero/equal condition
setge dst          # set dst to 0 or 1 based on greater/equal condition
cmovns src, dst    # proceed with mov if ns condition holds
cmovle src, dst    # proceed with mov if le condition holds
```

#### Function Call Stack

`push`和`pop`指令用于操作栈中元素，

函数调用者在执行被调用的函数前，首先需要设置寄存器，将传递的参数写入到寄存器`%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9`中

## Reference

【1】[CS107 Guide to x86-64 (stanford.edu)](https://web.stanford.edu/class/cs107/guide/x86-64.html)

【2】[Guide to x86 Assembly (yale.edu)](https://flint.cs.yale.edu/cs421/papers/x86-asm/asm.html)