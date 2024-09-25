# Lab0.5 启动GDB验证启动流程

## 一.实验目的

实验0.5主要讲解最小可执行内核和启动流程。我们的内核主要在 Qemu 模拟器上运行，它可以模拟一台 64 位 RISC-V 计算机。为了让我们的内核能够正确对接到 Qemu 模拟器上，需要**了解 Qemu 模拟器的启动流程**，还需要一些程序内存布局和编译流程（特别是链接）相关知识。

## 二.实验过程


*为了熟悉使用qemu和gdb进行调试工作,使用gdb调试QEMU模拟的RISC-V计算机加电开始运行到执行应用程序的第一条指令（即跳转到0x80200000）这个阶段的执行过程，说明RISC-V硬件加电后的几条指令在哪里？完成了哪些功能？要求在报告中简要写出练习过程和回答。*

### 1.复位

（1）.首先进入到riscv64-ucore-labcodes下的lab0中，同时打开两个shell后使用命令，结合gdb和qemu源码级调试ucore：

```bash
make debug
make gdb
```
打开makefile里的源文件，找到debug的target,发现其定义如下：
```json
 $(UCOREIMG) $(SWAPIMG) $(SFSIMG)
	$(V)$(QEMU) \
		-machine virt \
		-nographic \
		-bios default \
		-device loader,file=$(UCOREIMG),addr=0x80200000\
		-s -S   
```
这段代码首先通过宏定义$(UCOREIMG) $(SWAPIMG) $(SFSIMG)的函数进行目标文件的构建，然后使用qemu语句进行qemu启动加载内核，同时等待GDB连接以便进行调试。关键解析如下：

- `-device loader,file=$(UCOREIMG),addr=0x80200000:`: 告诉 QEMU 使用一个设备加载器（loader），加载 $(UCOREIMG) 文件，并将其映射到虚拟内存地址 0x80200000。

- `-s`: 告诉 QEMU 在启动时打开标准输入。

- `-S`: 告诉 QEMU 在启动时暂停执行，等待用户通过GDB连接。

gdb的target如下：
```json
 gdb:
	riscv64-unknown-elf-gdb \
    -ex 'file bin/kernel' \
    -ex 'set arch riscv:rv64' \
    -ex 'target remote localhost:1234'
```

显然，这个 gdb 目标的作用是启动GDB，加载指定的RISC-V架构的可执行文件，并连接到一个远程的调试目标。
- `-ex 'file bin/kernel'`: 告诉GDB加载 bin/kernel 文件作为调试的程序

- `-ex 'set arch riscv:rv64'`: 设置GDB的架构为RISC-V 64位

- `-ex 'target remote localhost:1234'`: 指示GDB连接到运行在本机的1234端口上的远程目标，即我们启动的debug命令

在我们进行make dbg后，得到如下结果：
![这是图片](1.jpg "Magic Gardens")

在RISC-V体系结构中，系统加电自检 后，处理器启动流程首先涉及将程序计数器（PC）初始化到一个预设的复位向量地址。对于采用QEMU模拟的RISC-V处理器而言，该**复位向量被配置为 0x1000。**
在QEMU的RISC-V模拟环境中，复位机制将处理器状态恢复至初始设定，包括但不限于寄存器清零、内存状态重置以及中断禁用等。复位地址 0x1000 是模拟环境中用于引导系统启动的起始代码位置。



（2）.进入gdb中通过命令`x/10i $pc`查看0x1000处的10条汇编指令。

```bash
(gdb) x/10i $pc
=> 0x1000:	auipc	t0,0x0
   0x1004:	addi	a1,t0,32
   0x1008:	csrr	a0,mhartid
   0x100c:	ld	t0,24(t0)
   0x1010:	jr	t0
   0x1014:	unimp
   0x1016:	unimp
   0x1018:	unimp
   0x101a:	0x8000
   0x101c:	unimp
```

接下来对代码进行逐行的解析：

- `auipc t0, 0x0`: 将当前 PC 加上立即数（0），结果存入寄存器 t0。

- `addi a1, t0, 32`: 将寄存器 t0 的值加上立即数 32，结果存入寄存器 a1。

- `csrr a0, mhartid`: 读取 mhartid CSR（Machine Hart ID）的值到寄存器 a0。

- `ld t0, 24(t0)`: 从寄存器 t0 偏移 24 个字节处加载一个双字（64位）到寄存器 t0。

- `jr t0`: 跳转到寄存器 t0 指示的地址，**此时t0寄存器指示的地址就是0x80000000。**

- `unimp`: 未实现指令。

- `0x8000`: 直接的立即数。

这些便是RISC-V硬件加电后的几条指令的一部分,用于在 RISC-V 架构中**进行程序控制和加载还有Bootloader的启动**，例如设置寄存器的值，跳转到指定地址，以及处理特殊情况。其中 `unimp` 表示未实现的指令。

接下来我们通过`si`命令逐行调试，最后通过`jr t0`跳转到物理地址 0x80000000 对应的指令处即启动Bootloader，并进入第二阶段。



### 2.Bootloader即OpeSBI启动

Bootloader的主要任务是**初始化系统硬件、配置内存，并最终加载操作系统内核至内存中。** 在Bootloader完成其任务后，控制权转交给操作系统内核，从而继续系统启动流程。
进入到`0x80000000`后，以10条指令为代表简单分析这部分的指令和所完成的功能：

```bash
(gdb) si
0x0000000080000000 in ?? ()
(gdb) x/10i $pc
=> 0x80000000:	csrr	a6,mhartid
   0x80000004:	bgtz	a6,0x80000108
   0x80000008:	auipc	t0,0x0
   0x8000000c:	addi	t0,t0,1032
   0x80000010:	auipc	t1,0x0
   0x80000014:	addi	t1,t1,-16
   0x80000018:	sd	t1,0(t0)
   0x8000001c:	auipc	t0,0x0
   0x80000020:	addi	t0,t0,1020
   0x80000024:	ld	t0,0(t0)
```

这些指令主要功能包括设置启动代码的入口地址、配置处理器寄存器以及获取CPU相关信息。这些步骤是确保RISC-V处理器能够**正确初始化并准备加载操作系统或其他应用程序**的关键环节。
之后为了正确地和上一阶段的 OpenSBI 对接，我们需要保证**内核的第一条指令位于物理地址 0x80200000 处**，因为这里的代码是**地址相关的**，这个地址是由处理器，即Qemu指定的。为此，我们需要将内核镜像预先加载到 Qemu 物理内存以地址 0x80200000 开头的区域上。

### 3.内核镜像启动

通过命令`break *0x80200000`在预先已经知道会被加载的内核镜像位置打上断点，并运行到对应位置。之后查看此处的10条指令。

```bash
(gdb) break *0x80200000
Breakpoint 1 at 0x80200000: file kern/init/entry.S, line 7.
(gdb) continue
Continuing.

Breakpoint 1, kern_entry () at kern/init/entry.S:7
7	    la sp, bootstacktop
(gdb) x/10i $pc
=> 0x80200000 <kern_entry>:		auipc	sp,0x3
   0x80200004 <kern_entry+4>:	mv	sp,sp
   0x80200008 <kern_entry+8>:	j	0x8020000c <kern_init>
   0x8020000c <kern_init>:		auipc	a0,0x3
   0x80200010 <kern_init+4>:	addi	a0,a0,-4
   0x80200014 <kern_init+8>:	auipc	a2,0x3
   0x80200018 <kern_init+12>:	addi	a2,a2,-12
   0x8020001c <kern_init+16>:	addi	sp,sp,-16
   0x8020001e <kern_init+18>:	li	a1,0
   0x80200020 <kern_init+20>:	sub	a2,a2,a0
```

对代码前三行进行逐行解析：

- `auipc sp, 0x3`: 将当前指令的地址的高 20 位加上一个偏移量（0x3），并将结果存储在栈指针寄存器 sp 中。

- `mv sp, sp`: 将 sp 寄存器的值复制给 sp 寄存器 

- `j 0x8020000c`: 无条件跳转到地址 `0x8020000c` 处执行。

   以上三段代码等价于kern_entry中的：
	
	```assembly
	la sp, bootstacktop
	tail kern_init
	```
	
 

这段代码实际上是已经执行应用程序的第一条指令（**即跳转到0x80200000即kern_entry**）后的指令，并不属于加电后的执行阶段了。主要**用于内核的初始化**等。但通常用于初始化寄存器、栈帧以及准备参数以调用其他函数，包括从kern_entry跳转到kern_init。

 

## 三.知识点总结
### 1.项目组成
```
── Makefile  GNU make编译脚本
├── kern
│   ├── debug
│   │   ├── assert.h
│   │   ├── kdebug.c
│   │   ├── kdebug.h
│   │   ├── kmonitor.c
│   │   ├── kmonitor.h
│   │   ├── panic.c
│   │   └── stab.h
│   ├── driver
│   │   ├── clock.c
│   │   ├── clock.h
│   │   ├── console.c  简单封装OpenSBI字符读写接口，向上提供给输入输出库
│   │   ├── console.h
│   │   ├── intr.c
│   │   ├── intr.h
│   │   ├── kbdreg.h
│   │   ├── picirq.c
│   │   └── picirq.h
│   ├── init
│   │   ├── entry.S  OpenSBI启动之后跳转处，进行内核栈的分配
│   │   └── init.c  内核真正的入口点，从entry.S跳转过来完成其他初始化工作
│   ├── libs
│   │   ├── readline.c
│   │   └── stdio.c
│   ├── mm
│   │   ├── memlayout.h
│   │   ├── mmu.h
│   │   ├── pmm.c
│   │   └── pmm.h
│   └── trap
│       ├── trap.c
│       ├── trap.h
│       └── trapentry.S
├── libs
│   ├── defs.h  定义了一些常用的类型和宏
│   ├── elf.h
│   ├── error.h  定义了一些内核错误类型的宏
│   ├── printfmt.c
│   ├── riscv.h 以宏的方式定义了riscv指令集的寄存器和指令
│   ├── sbi.c  封装OpenSBI接口为函数
│   ├── sbi.h
│   ├── stdarg.h
│   ├── stdio.h  实现了一套标准输入输出
│   ├── string.c
│   └── string.h  一些对字符数组进行操作的函数
└── tools
    ├── function.mk  定义Makefile中使用的一些函数
    ├── kernel.ld 链接脚本，告诉链接器如何将目标文件的section组合为可执行文件
```

### 2.内核执行流步骤： 

`加电 -> OpenSBI启动 -> 跳转到 0x80200000 (kern/init/entry.S）->进入kern_init()函数（kern/init/init.c) ->调用cprintf()输出一行信息->结束`

在QEMU模拟的这款riscv处理器中，将**复位向量地址初始化为0x1000**，再将PC初始化为该复位地址，因此处理器将从此处开始执行复位代码，复位代码主要是将计算机系统的各个组件（包括处理器、内存、设备等）置于初始状态，并且会启动Bootloader。

操作系统作为一个程序，必须加载到内存里才能执行，必然有一个其他程序完成这个工作，而这个程序就是bootloader.：他负责boot(开机)，还负责load(加载OS到内存里)。等完成后，其把CPU的控制权交给操作系统。

在QEMU模拟的riscv计算机里，我们使用QEMU自带的bootloader: OpenSBI固件，那么在 Qemu 开始执行任何指令之前，首先两个文件将被加载到 Qemu 的物理内存中：即作为 bootloader 的 **OpenSBI.bin 被加载到物理内存以物理地址 0x80000000 开头的区域**上，同时**内核镜像 os.bin 被加载到以物理地址 0x80200000 开头的区域上**




### 3.bin和elf
两者都是都是用于存储程序的二进制格式
- `elf`: 复杂，包含一个文件头(ELF header), 包含冗余的调试信息，指定程序每个section的内存布局，需要解析program header才能知道各段(section)的信息。

- `bin`: 简单地在文件头之后解释自己应该被加载到什么起始位置

实际上，可以认为**bin文件会把elf文件指定的每段的内存布局都映射到一块线性的数据里**，这块线性的数据（或者说程序）加载到内存里就符合elf文件之前指定的布局。所以，我们只需要得到内存布局合适的elf文件，然后把它转化成bin文件（这一步通过objcopy实现），然后加载到QEMU里运行（QEMU自带的OpenSBI会干这个活）。

### 4.内核栈

​内核栈是操作系统内核在**执行任务时用于存储临时数据**，如局部变量、函数参数和返回地址等的内存区域，它采用**栈的数据结构，遵循后进先出（LIFO）的原则**。在系统启动的引导过程中，bootloader 作为操作系统内核的启动程序，其职责是初始化内核栈，并将操作系统加载到内存中，准备启动。内核栈在引导过程中极为关键，因为它提供了一个必要的存储空间，用于在内核运行时保存和管理临时数据。一旦操作系统内核启动，它将利用这个初始化的内核栈来执行包括处理中断、任务调度和系统调用等在内的多种操作，从而确保内核的顺畅运行。简而言之，**内核栈是操作系统内核的核心组件之一**，而 bootloader 的任务是设置内核栈并引导内核开始工作。

 
### 5.地址相关代码与地址无关代码 
地址无关代码（Position-Independent Code, PIC）和地址相关代码（Position-Dependent Code, PDC）是两种不同的代码类型，其主要区别在于代码中是否包含硬编码的内存地址。

**地址无关代码 (PIC)**:

- **含义**: 这种代码类型不依赖于固定的内存地址，可以在不同的内存位置加载和运行而不需要修改其本身代码。

- **特点**:
  - 使用**相对寻址或寄存器间接寻址**，避免了硬编码的内存地址。
  - 可以被加载到任意内存位置，而无需重定位。
  - 通常用于**共享库 或可执行文件**，使得共享库可以在内存中的不同位置加载，同时可以减少不同程序的内存占用。

**地址相关代码 (PDC)**:

- **含义**: 这种代码类型包含了硬编码的内存地址，依赖于特定的内存布局。如果将其加载到不同的内存地址，可能会导致错误或崩溃。

- **特点**:
  - 包含**直接使用内存地址的指令或数据**。
  - 在加载到不同的内存位置时，可能需要对其进行重定位，修改内部的地址值。
  - 通常用于**独立的可执行文件**，对于每个程序，其被加载到内存的特定位置。
 
