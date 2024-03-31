---
declare: true
title: 使用qemu进行Linux内核调试
categories: [Linux,debug,gdb]
tags:
- Linux
- debug
- gdb
---

---



>  一篇创建于2023.9.11，完成于2024.3.31，拖了大半年的文章

## 调试前的准备

#### 编译qemu

在ubuntu中可以通过apt安装qemu-system，也可以从源码自行编译。在编译qemu-8.0时，可能遇到`kvm Parameter 'type' expects a netdev backend type`，原因是因为在编译时没有enable-slirp，而enable_slirp需要安装相应的依赖，完整的命令如下：

```shell
sudo apt-get install libslirp
sudo apt-get install libglib2.0-dev
cd qemu-8.0
./configure -enable-slirp
make
```

#### 编译内核

想必大家都会，略过不提

#### 编译磁盘镜像

qemu可以使用多种格式的磁盘镜像，在qemu的文档中给出了一些创建文件系统的方式：[Disk Images — QEMU documentation (qemu-project.gitlab.io)](https://qemu-project.gitlab.io/qemu/system/images.html)

个人比较喜欢通过两种方式创建磁盘镜像：

1. 利用syzkaller的create-image.sh：[syzkaller/tools/create-image.sh at master · google/syzkaller (github.com)](https://github.com/google/syzkaller/blob/master/tools/create-image.sh)
2. 编译busybox：[Qemu 模拟环境 - CTF Wiki (ctf-wiki.org)](https://ctf-wiki.org/pwn/linux/kernel-mode/environment/qemu-emulate/#_8)

前者的好处是创建的磁盘镜像比较完整，比较方便安装各种工具、与宿主机进行各种交互，而后者的好处是比较轻量，只有一个init进程，可以避免多余进程的干扰。可以根据自己需求自行选择。

#### 启动内核

常用的启动命令：

```shell
sudo qemu-system-aarch64 -M virt -machine gic-version=3 \
-enable-kvm -cpu host -nographic -smp 8 -drive \
file=/path/to/jammy.img,format=raw \
-kernel /path/to/arch/arm64/boot/Image  \
-append "nokaslr console=ttyAMA0 root=/dev/vda oops=panic panic_on_warn=1 panic=-1 ftrace_dump_on_oops=orig_cpu debug earlyprintk=serial net.ifnames=0 slub_debug=UZ" -m 8G -net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10024-:22 -net nic -S -gdb tcp::4321 -no-reboot
```

对其中各个参数的解释：

1. enable-kvm可以使虚拟机获得*接近于*真机的性能，并且如果使用-cpu host使用宿主机cpu的硬件特性，则必须开启kvm。开启kvm后可能对调试过程产生干扰，即在使用gdb单步调试时常常陷入中断，目前存在几种办法来避免该问题：
   1. 在单步的下一条指令处打断点，直接跳到下一条指令，忽略中间的中断处理
   2. 在要调试的代码前后关闭中断
   3. 关闭kvm，并在qemu-system-aarch64 --machine help列出的qemu所支持的cpu中选择合适的替换-cpu host
2. -append后面是内核的启动参数，具体可以参阅内核文档https://docs.kernel.org/admin-guide/kernel-parameters.html
3. -net后面是对qemu的网络设置，为了使该设置生效，在编译内核时需要将网卡驱动编译进内核，即将CONFIG_E100、CONFIG_E1000、CONFIG_E1000E设置为y
4. -gdb表示可以通过gdb连接该端口实现远程调试，-S表示需要等待gdb连接后内核才能正常启动
5. -no-reboot表示内核关机后自动重启
6. 其他需要的参数可以自行参阅文档，按需使用

## 调试内核

#### 调试内核的启动过程

在调试正常启动后的内核时使用的gdb命令与调试普通的用户态进程并没有什么太大的区别，然而在调试内核的启动过程时，发现内核中head.S打不上断点，究其原因在于内核启动时在__enable_mmu前尚未开启mmu，运行时使用的.text段、.init.text段和.rodata等段的位置与开启mmu和分页后使用的地址不一致，也就导致gdb插入stub的地址与内核实际执行的指令的地址不一致，因此需要通过add-symbol-file手动指定内核中各个段的位置，以使gdb能够正确添加断点。

首先使用qemu启动内核，在gdb中连接虚拟机：

```
(gdb) target remote:4321
Remote debugging using:4321
0x0000000040000000 in ??()
```

我们查看此时的前32条指令，得到：

![image-20230911215712787.png](https://s2.loli.net/2024/03/31/DodLiQalTUxEeVr.png)

前几条指令是qemu为了引导内核启动加入的代码， 首先将0x40200000加载到x4寄存器中，然后通过br x4跳转到0x40200000位置。查看该位置的代码，得到：

![image-20230911220129145.png](https://s2.loli.net/2024/03/31/EnQYp6vqcuaT84O.png)

对应地，查看head.S的代码，有：

```assembly
_head:
	/*
	 * DO NOT MODIFY. Image header expected by Linux boot-loaders.
	 */
#ifdef CONFIG_EFI
	/*
	 * This add instruction has no meaningful effect except that
	 * its opcode forms the magic "MZ" signature required by UEFI.
	 */
	add	x13, x18, #0x16
	b	primary_entry
#else
	b	primary_entry			// branch to kernel start, magic
	.long	0				// reserved
#endif
	.quad	0				// Image load offset from start of RAM, little-endian
	le64sym	_kernel_size_le			// Effective size of kernel image, little-endian
	le64sym	_kernel_flags_le		// Informative flags, little-endian
	.quad	0				// reserved
	.quad	0				// reserved
	.quad	0				// reserved
	.ascii	ARM64_IMAGE_MAGIC		// Magic number
#ifdef CONFIG_EFI
	.long	pe_header - _head		// Offset to the PE header.
```

二者刚好对应，因此，内核在qemu上的加载基地址为0x40200000。

我们通过readelf查看vmlinux的段偏移：

![image-20230911220705601.png](https://s2.loli.net/2024/03/31/8oxWkKAXRf1zOdH.png)

![image-20230911220806932.png](https://s2.loli.net/2024/03/31/zR1MwfWk5InhxUJ.png)

可以得到.text相对于.head.text的偏移为0x10000,.rodata相对于.head.text的段偏移为0xe70000，.init.text相对于.head.text的偏移为0x15d0000，因此可以得到加载时内核各个段在内存中的地址，据此通过add-symbol-file对MMU开启前的section进行符号表的修改：

```shell
add-symbol-file vmlinux -s .head.text 0x40200000 -s .text 0x40210000 -s .rodata 0x41170000 -s .init.text 0x417d0000
```

#### 调试动态插入内核的kernel module

由于动态插入的kernel module在编译时并没有编入vmlinux，并且在kernel module插入内核时其内存地址是动态分配的，因此gdb无法通过读取vmlinux找到相应的符号，也无法静态地确定kernel module在内存中的位置从而也就无法在相应的函数上打断点。

在kernel module插入内核后，内核会为每个kernel module创建一个kobject，并在sysfs伪文件系统下创建相应的文件用于导出其数据和属性到用户空间，为用户提供对这些数据和属性的访问支持。以dm_mirror为例，在/sys/module/dm_mirror/sections中，存在如下文件：

![image-20240331163125159.png](https://s2.loli.net/2024/03/31/CuvkqKydI6gUFPD.png)

其中.text、.data、.rodata等分别代表着dm_mirror中相应的段在内核中的地址：

![image-20240331163251096.png](https://s2.loli.net/2024/03/31/nJpOBdX8VEy4UWo.png)

在gdb中，同样使用add-symbol-file来指定ko段在内核中的地址，使gdb能够得到各个符号的位置和相应的描述，即可成功插入断点，实现对kernel module的调试：

```shell
add-symbol-file /path/to/dm_mirror.ko -s .text address
```