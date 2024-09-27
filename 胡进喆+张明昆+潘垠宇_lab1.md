<h1 align = "center">操作系统实验报告</h1>

<h3 align = "center">实验名称：中断与中断处理流程    实验地点：实验楼A204</h3>

<h4 align = "center">组号：94      小组成员：胡进喆  张明昆  潘垠宇</h4>

## 一、实验目的

实验1 主要讲解的是中断处理机制。操作系统是计算机系统的监管者，必须能对计算机系统状态的突发变化做出反应，这些系统状态可能是程序执行出现异常，或者是突发的外设请求。当计算机系统遇到突发情况时，不得不停止当前的正常工作，应急响应一下，这是需要操作系统来接管，并跳转到对应处理函数进行处理，处理结束后再回到原来的地方继续执行指令。这个过程就是中断处理过程。

## 二、实验过程

### 1.练习1

`kern/init/entry.S`是`OpenSBI`启动时最先执行的一段汇编代码，在该段代码中，完成了对于内核栈的分配，然后跳转到真正的内核初始化函数进行执行。下面我们对题目中要解析的两句汇编指令进行解析。

1.`la sp, bootstacktop`

**la(load address)用于将一个标签的地址加载到寄存器中**。sp 是栈指针寄存器，负责指向当前栈的顶部。bootstacktop 是一个标签，表示引导栈（boot stack）的顶部地址。

这条指令**将 bootstacktop 的地址加载到栈指针寄存器 sp 中**，即将 sp 设置为引导栈的顶部地址。

在内核启动初期，必须设置一个有效的栈空间供内核使用。通过将 sp 设置为 bootstacktop，确保内核在后续的操作中有一个已知且有效的栈空间。

2.`tail kern_init`

tail 是一种跳转指令，类似于 jmp，但通常用于函数调用的尾部优化（尾调用优化），以减少栈的使用和提高效率。**kern_init 是内核初始化函数的标签，负责完成内核的进一步初始化工作。**

这条指令将控制权转移到 kern_init 函数，同时不会返回到 kern_entry，即内核初始化流程从 kern_entry 直接跳转到 kern_init。

### 2.练习2

我们实现时钟中断的处理函数代码如下：

```C
        case IRQ_S_TIMER:
            // "All bits besides SSIP and USIP in the sip register are
            // read-only." -- privileged spec1.9.1, 4.1.4, p59
            // In fact, Call sbi_set_timer will clear STIP, or you can clear it
            // directly.
            // cprintf("Supervisor timer interrupt\n");
             /* LAB1 EXERCISE2   YOUR CODE :  */
            /*(1)设置下次时钟中断- clock_set_next_event()
             *(2)计数器（ticks）加一
             *(3)当计数器加到100的时候，我们会输出一个`100ticks`表示我们触发了100次时钟中断，同时打印次数（num）加一
            * (4)判断打印次数，当打印次数为10时，调用<sbi.h>中的关机函数关机
            */
            clock_set_next_event();
            ++ticks;
            if(ticks % TICK_NUM == 0){
                print_ticks();
                ++num;
            }
            if(num == 10){
                sbi_shutdown();
            }
            break;
```

每次触发时钟中断后，我们通过调用`clock_set_next_event()`函数来设置下一个时钟中断，然后将中断次数计数器`ticks`加1，然后判断该次数是否为100的整数倍，若是的话则调用函数打印`100 ticks`，同时把打印次数计数器`PRINT_NUM`也加1。当`PRINT_NUM`的值达到10时，调用`shutdown`函数进行关机。

下面我们简要说明一下时钟中断的处理流程。

最早产生的时钟中断事件是在`kern/init/init.c`文件中产生的，在该文件中有如下代码：

```C
int kern_init(void) {
    extern char edata[], end[];
    memset(edata, 0, end - edata);

    cons_init();  // init the console

    const char *message = "(THU.CST) os is loading ...\n";
    cprintf("%s\n\n", message);

    print_kerninfo();

    // grade_backtrace();

    idt_init();  // init interrupt descriptor table

    // rdtime in mbare mode crashes
    clock_init();  // init clock interrupt

    intr_enable();  // enable irq interrupt
    
    while (1)
        ;
}
```

其中的clock_init()产生了**第一次**时钟中断，捕捉到该中断后操作系统通过查找`stvec`，调用`trap()`函数，该对中断进行识别分类，然后进入上述的时钟中断处理函数中进行处理。

下面我们探索一下clock_init()的具体实现。
```C
void clock_init(void) {
    // enable timer interrupt in sie
    set_csr(sie, MIP_STIP);
    // divided by 500 when using Spike(2MHz)
    // divided by 100 when using QEMU(10MHz)
    // timebase = sbi_timebase() / 500;
    clock_set_next_event();

    // initialize time counter 'ticks' to zero
    ticks = 0;

    cprintf("++ setup timer interrupts\n");
}
```
上述代码是`clock_init()`函数的定义，可以看到该函数通过调用`clock_set_next_event()`函数来初始化第一次时钟中断。

clock_set_next_event()的定义又是什么呢，我们继续深入观察一下。

```C
void clock_set_next_event(void) { sbi_set_timer(get_cycles() + timebase); }
```

可以看到该函数是把**当前的时刻+固定时间间隔**作为下一次产生时钟中断的时间。

而get_cycles()又是如何获取当前时刻的呢。

```C
static inline uint64_t get_cycles(void) {
#if __riscv_xlen == 64
    uint64_t n;
    __asm__ __volatile__("rdtime %0" : "=r"(n));
    return n;
#else
    uint32_t lo, hi, tmp;
    __asm__ __volatile__(
        "1:\n"
        "rdtimeh %0\n"
        "rdtime %1\n"
        "rdtimeh %2\n"
        "bne %0, %2, 1b"
        : "=&r"(hi), "=&r"(lo), "=&r"(tmp));
    return ((uint64_t)hi << 32) | lo;
#endif
}
```
可以看到上述代码里的`get_cycles()`函数是计算执行到当前指令时距CPU开始运转时已经过去了多少周期。
### 3.扩展练习Challenge1

#### 3.1 ucore 中处理中断异常的流程

1. 当计算机运行过程中发生中断或异常，硬件会向处理器发出信号，通知操作系统需要进行处理。；

2. 处理器读取 `stvec`（Supervisor Trap Vector）寄存器的值，以确定中断处理程序的入口地址。stvec具有两种模式。若`stvec`寄存器最低2位是00，则说明其高位保存的是唯一的中断处理程序的地址；如果是01，说明其高位保存的是中断向量表的地址，操作系统通过不同的异常原因来索引中断向量表以获取处理程序的地址。
   
3. 我们将`stvec`的值设置成为中断入口点`__alltraps`的地址。进入中断入口点后，操作系统通过`SAVE_ALL`汇编宏来保存上下文，然后进入`trap()`函数进行处理。
   
4. 在 `trap()` 函数内部，调用 `trap_dispatch()` 函数，根据捕捉到的中断或异常类型进行分发处理。通过检查 `tf->cause`（陷阱原因寄存器）的最高位，区分中断（Interrupt）和异常（Exception）。中断则调用 `interrupt_handler()` 进行处理。异常则调用 `exception_handler()` 进行处理。
   
5. 处理完成后，会执行`__trapret`部分的代码。通过`RESTORE_ALL`汇编宏恢复各个寄存器的值，然后通过`sret`指令把`sepc`的值赋值给`pc`，使处理器继续执行被中断或异常打断的指令之后的代码。
```S
__alltraps:   
    SAVE_ALL
    move  a0, sp  #栈帧的指针传递给 a0 寄存器，用于在异常处理函数中访问保存的寄存器
    jal trap
    # sp should be the same as before "jal trap"

    .globl __trapret
__trapret:
    RESTORE_ALL
    # return from supervisor call
    sret   #从异常返回
```
#### 3.2 一些问题

1. `mov a0，sp` 的目的是什么？

   **答：** 该指令是把**保存上下文的栈顶指针寄存器**赋值给a0寄存器，a0寄存器是参数寄存器，这样就可以把当前的中断帧作为参数传递给中断处理程序，从而实现对中断的处理。

2. `SAVE_ALL`中寄存器保存在栈中的位置是什么确定的。

   **答：** 我们通过栈顶寄存器sp来索引各个寄存器保存的位置。
   在保存上下文之前我们首先通过指令`addi sp, sp, -36 * REGBYTES`，在栈中开辟出了保存上下文的内存区域，然后我们通过栈顶指针sp来把对应的寄存器保存在栈中。

3. 对于任何中断，`__alltraps` 中都需要保存所有寄存器吗？请说明理由。

    **答：** 保存所有寄存器是确保中断处理后，被中断的程序能够正确恢复执行的关键。
    统一的保存策略简化了内核的设计，降低了出错的可能性，提高了系统的可靠性和稳定性。


### 4.扩增练习Challenge2

1. `csrw sscratch, sp；csrrw s0, sscratch, x0` 实现了什么操作，目的是什么？

   **答：**
   `csrw sscratch, sp`这条指令将当前的栈指针寄存器 sp 的值写入到控制状态寄存器 sscratch 中。

   `csrrw s0, sscratch, x0`这条指令将 sscratch 的原值读取到 s0 中，同时 sscratch 被写入值 0。实现sscratch的清零，以便后续标识中断前程序处于S态。

   将 sscratch 设置为 0，可以用于判断异常或中断是否发生在内核态。如果在处理异常期间再次发生异常，通过检查 sscratch 是否为 0，可以识别递归异常，避免重复保存上下文或引发其他问题。

2. `SAVE_ALL`里面保存了stval,scause 这些csr，而在`RESTORE_ALL`里面却不还原它们？那这样store的意义何在呢？

   **答：**
    在 SAVE_ALL 中，我们将异常相关的控制状态寄存器（CSR）如 stval（sbadaddr）、scause、sstatus、sepc 等保存到栈中。这样做的目的是让操作系统内核的异常处理函数能够访问这些信息，以确定异常的类型、原因和发生的地址，从而进行相应的处理。

    stval 和 scause ，记录了异常发生时的地址和原因码。异常处理完成后，**这些寄存器的值对于恢复正常执行并不重要，不需要还原。** 恢复执行时，只需确保程序计数器 sepc 和状态寄存器 sstatus 被正确还原即可。

   

### 5.扩展练习Challenge3

我们编程完善的代码如下：

```C
	case CAUSE_ILLEGAL_INSTRUCTION:
		// 非法指令异常处理
		/* LAB1 CHALLENGE3   YOUR CODE : */
	   /*(1)输出指令异常类型（ Illegal instruction）
		*(2)输出异常指令地址
		*(3)更新 tf->epc寄存器
	   */
		cprintf("Exception type:Illegal instruction \n");
		cprintf("Illegal instruction exception at 0x%016llx\n", tf->epc);//采用0x%016llx格式化字符串，用于打印16位十六进制数，这个位置是异常指令的地址,以tf->epc作为参数。
		//%016llx中的%表示格式化指示符的开始，0表示空位补零，16表示总宽度为 16 个字符，llx表示以长长整型十六进制数形式输出。
		tf->epc += 4;//指令长度都为4个字节
		break;
	case CAUSE_BREAKPOINT:
		//断点异常处理
		/* LAB1 CHALLLENGE3   YOUR CODE : */
		/*(1)输出指令异常类型（ breakpoint）
		 *(2)输出异常指令地址
		 *(3)更新 tf->epc寄存器
		*/
		cprintf("Exception type: breakpoint \n");
		cprintf("ebreak caught at 0x%016llx\n", tf->epc);
		tf->epc += 2;//ebreak指令长度为2个字节，为了4字节对齐
		break;
```

1. 非法指令异常处理结束后，`tf->epc`的值加了4，这是因为导致该异常的指令`mret`的长度为4字节，因此我们通过该操作便能够成功在程序顺序执行过程中跳过非法指令。
2. 断点异常处理结束后，`tf->epc`的值加了2，这是由于导致该异常的指令`ebreak`的长度是2字节，为了保存指令的四字节对其，我们便只加2。

在上述处理结束后，我们通过`RETORE_ALL`和`sret`指令，把更新后的sepc的值赋值给pc，使得程序跳过非法指令继续执行。

## 三、实验中的知识点

- ### 1. **中断处理机制**

  中断是计算机系统响应外部或内部事件的方式，它允许CPU在处理当前任务时，能够中断其执行并转向处理其他更紧急的任务。中断处理机制是操作系统的一个关键组成部分，其基本流程如下：

  - **中断源**：中断可以由硬件（如定时器、外设等）或软件（系统调用等）触发。
  - **中断请求**：当发生中断时，CPU会停止当前的执行，并保存当前上下文（程序计数器、寄存器等状态）。
  - **中断向量**：CPU通过查找中断向量表（IVT）或`stvec`寄存器来确定中断处理程序的地址。
  - **中断处理**：操作系统执行相应的中断处理程序，完成对中断的响应和处理。
  - **恢复上下文**：处理完成后，操作系统恢复之前的上下文，继续执行被中断的程序。

  ### 2. **操作系统的初始化过程**

  操作系统的初始化过程是加载和准备操作系统的运行环境。这个过程通常包括以下几个步骤：

  - **内存清零**：如在`kern_init()`函数中，使用`memset()`清除未初始化的全局变量，确保内存中的数据是已知的。
  - **设备初始化**：如`cons_init()`用于初始化控制台设备，准备输入输出环境。
  - **中断描述符表初始化**：通过`idt_init()`函数初始化中断描述符表（IDT），以处理各类中断。
  - **时钟中断初始化**：通过`clock_init()`函数设置时钟中断，以便系统可以定期响应和处理时钟事件。

  ### 3. **时钟中断**

  时钟中断是操作系统维护时间和调度任务的重要机制。它通常被用来：

  - **定期执行任务**：例如，操作系统通过时钟中断来触发定时任务或轮询外设。
  - **维护系统时间**：操作系统记录系统启动后的时间，并处理与时间相关的系统调用。
  - **实现调度**：时钟中断可以使操作系统实现时间片轮转调度，确保多个进程能够公平地共享CPU资源。

  ### 4. **中断向量表（IVT）与`stvec`寄存器**

  - **中断向量表**：是一个存储中断处理程序地址的表。每个中断都有一个唯一的入口点。
  - **`stvec`寄存器**：用于存储中断处理程序的入口地址或中断向量表的起始地址。通过读取`stvec`，处理器可以知道在发生中断时应该跳转到哪个处理程序。

  ### 5. **上下文切换**

  上下文切换是操作系统在多个任务（进程或线程）之间切换执行状态的过程。它涉及到：

  - **保存上下文**：在中断发生时，操作系统保存当前任务的状态。
  - **恢复上下文**：在切换回某个任务时，操作系统恢复该任务之前的状态。
  - **性能影响**：上下文切换会消耗CPU时间和资源，因此要尽量减少频繁的切换。

  ### 6. **定时器与周期性任务**

  在实验中，通过时钟中断来设置定时器，实现周期性任务的调度。以下是相关概念：

  - **定时器中断**：定时器以固定频率触发中断，使操作系统能够执行预定的任务。
  - **周期性任务**：使用定时器来定期执行特定功能，如更新系统状态或轮询设备状态。
