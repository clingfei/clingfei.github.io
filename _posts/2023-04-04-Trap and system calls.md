---
declare: true
title: xv6 trap and system call notes
categories: [operating system, xv6]
tags:
- operating system
- xv6
---

---

trap包括三种情况：

- 系统调用，用户程序执行ecall指令切换到内核态执行对应的系统调用处理程序

- 异常，用户或内核执行某些非法指令，例如除零或访问无效的虚拟地址

- 设备中断

## 与trap有关的控制寄存器
- stvec: 保存trap处理程序的地址
- sepc: trap发生时，pc将会切换到trap处理程序的首地址，而sepc用于保存被中断的程序计数器的值。在sret返回时，将sepc寄存器的值加载到pc中
- scause：保存trap发生的原因
- sscratch：指向每个进程的trapframe的地址，在陷入处理程序起始时使用，同时用于保存a0寄存器
- sstatus：SIE位表示当前开启了设备中断，如果SIE位为0，RISC-V将会推迟设备中断的处理，直到内核将SIE置位。SPP位表示trap来自于用户模式还是supervisor模式，并在返回时决定跳转到user还是supervisor。
上述寄存器仅能在supervisor模式下访问。

## RISC-V硬件处理trap
1. 如果trap是由设备中断引起的，并且当前sstatus 的SIE位为0，则无需处理，等待内核再次将SIE设置为1时再处理。
2. SIE位清0，关闭设备中断
3. 将pc保存到sepc寄存器
4. 保存当前模式到sstatus的SPP位
5. 将trap发生原因保存到scause寄存器
6. 将模式切换到supervisor
7. 将stvec寄存器的值加载到pc，即切换到trap处理程序执行
8. 按照pc所指的位置执行trap处理程序
在RISC-V中，CPU仅保存了并切换了PC的值，而并没有切换到内核页表和内核栈，这一部分工作是由操作系统内核实现的，这样做是为了为操作系统保留足够的灵活性，因为某些操作系统可能在处理trap时并不需要切换页表，从而能够提高性能。

## trap from user space
上面提到的stvec寄存器中实际上保存的是uservec的入口地址。在处理陷入时，首先执行uservec，然后执行usertrap。在从陷入返回时，首先执行usertrapret，然后执行userret。

由于RISC-V硬件在处理trap时，并不会切换页表，因此从用户页表到内核页表的切换是由uservec来完成的，同时为了uservec在进程切换到内核页表后能够继续执行，因此uservec在用户页表和内核页表中应该具有相同的映射。

实际上uservec保存在trampoline页中，从下面两张图可以看出，trampoline页在用户页表和内核页表中，都是占据了MAXVA-PAGESIZE~MAXVA这一块虚拟地址空间，因此具有相同的虚拟地址，并且会映射到相同的物理地址。

![Pasted image 20230404164748.png](https://s2.loli.net/2023/04/05/vTzV2FAf5J8KrkP.png)
![Pasted image 20230404164818.png](https://s2.loli.net/2023/04/05/uPdhIvMBOYsjpKz.png)

在uservec开始执行时，RISC-V CPU的32个寄存器中保存着被中断的代码的值，uservec需要保存这些寄存器的值，以保证在trap返回时能够继续执行。但是，无论是设置satp以加载内核页表还是保存寄存器现场，uservec都需要使用寄存器，从而就会修改上面说的某些寄存器的值。为了解决这个问题，uservec利用了RISC-V的sscratch寄存器和csrrw指令，将a0寄存器与sscratch寄存器的值进行交换，即将a0寄存器的值保存到sscratch寄存器中，而此时a0寄存器中保存了当前进程的trapframe的地址。随后，uservec将所有的寄存器（包括目前存在于sscratch中的a0寄存器）的值保存在trapframe页中。（*此时进程页表尚未切换到内核态，因此寄存器保存在用户地址空间*）

同时trapframe中还保存有当前进程内核栈的指针，当前CPU的hartid（*[硬件线程号](https://stackoverflow.com/questions/42676827/risc-v-spec-references-the-word-hart-what-does-hart-mean)*），usertrap的地址以及内核页表的地址。uservec从trapframe中取出这些值，将内核页表加载到satp寄存器中，跳转到usertrap执行。

usertrap首先判断陷入的类型，并调用对应的处理函数，最后调用usertrapret返回。首先设置stvec寄存器为kernelvec的地址，从而使kernelvec处理来自内核态的trap。然后保存sepc寄存器的值（*此时sepc中保存的是在陷入时由硬件自动保存的被中断的用户程序的PC*），因为usertrap中可能会出现导致sepc中的值被覆盖的进程切换，随后根据陷入类型调用不同的处理函数：syscall/devintr/kill。（*特殊情况：对于系统调用，在陷入时pc指向ecall这条指令，因此需要对trapframe中的pc值+4，使返回时执行ecall的后一条指令，而不是再次陷入内核*）。

在处理完陷入后，首先调用usertrapret。usertrap首先恢复RISC-V的控制寄存器以供下次陷入使用。包括使stvec指向uservec（*该寄存器在usertrap中设置为指向kernelvec*），设置uservec中使用的保存在trapframe中的值（*包括当前进程内核栈的指针、CPU的hartid、usertrap的地址和内核页表的地址等*），将usertrap中保存的pc的值恢复到sepc。最后调用userret，与uservec类似，由于该部分代码需要设置satp寄存器以实现内核页表到用户页表的切换，因此同样保存在trampoline中。

usertrapret在调用userret时传递了两个参数，其中a0寄存器中保存了进程的用户页表的地址，a1寄存器中保存了TRAPFRAME的地址。userret将satp寄存器设置为用户页表的地址，从trapframe中拷贝a0的值到sscratch寄存器，之后userret从trapframe中恢复其他保存的寄存器的值，并交换a0和sscratch两个寄存器，以使sscratch重新指向进程的trapframe。此时完成了对于中断现场的恢复，最后执行sret恢复到用户态继续执行之前被中断的代码。

## traps from kernel space
xv6在执行内核代码时，stvec寄存器指向的是kernelvec。与uservec不同，kernelvec不需要进行用户页表到内核页表的切换和用户栈到内核栈的切换，只需要保存当前所有寄存器的值到内核栈。

保存现场后，kernelvec跳转到kerneltrap，用于处理两种类型的陷入：设备中断和异常。如果陷入的原因是时钟中断，并且此时正在运行的是一个进程对应的内核线程而不是scheduler，kerneltrap将会调用yield以切换线程执行。

kerneltrap返回时，恢复保存的sepc和sstatue寄存器的值，然后返回到kernelvec。kernelvec从栈中恢复保存的上下文。

当CPU从用户空间进入到内核态时，需要将stvec设置成kernelvec，而在进入内核态到设置为kernelvec之间，需要关闭中断，否则一旦出现中断，根据stvec将会调用uservec，而此时系统已经处于内核态，从而会引起整个系统的混乱。在设置stvec为kernelvec之后，再重新开启中断。

## Reference
[xv6: a simple, Unix-like teaching operating system (mit.edu)](https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf)