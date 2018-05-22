##2.1ARM体系结构概述

### 2.1.1 ARM简介
 
ARM公司是一家知识产权（IP）供应商，它不制造芯片且不向终端用户出售芯片，而是向合作伙伴提供设计方案，再由合作伙伴生产各种各样的芯片。ARM公司通过这种双赢的方式迅速成为了全球性RISC（Reduced Instruction Set Computer）微处理器标准的缔造者。

ARM架构是ARM公司面向市场设计的第一款低成本RISC微处理器，具有耗电少功能强、16位/32位双指令集和合作伙伴众多等众多特点。并在现实中应用广泛，比如手机、PDA、MP3/MP4等便携式消费产品中。

### 2.1.2 RISC结构特性

RISC（Reduced Instruction Set Computer）中文是精简指令集计算机，是在CISC（复杂指令计算机）的基础上优先选取使用频率最高的简单指令，避免复杂指令；并将指令长度固定，减少指令格式和寻址方式；不用或少用微码控制已达到精简计算机结构，提高运算速度的目的。

ARM内核采用精简指令集计算机（RISC）体系结构，因此ARM具备了非常典型的RISC结构特性：
	
	（1）具有大量的通用寄存器；
    
	（2）通过装载/保存（load-store）结构使用独立的load和store指令完成数据在寄存器和外部存储器之间的传送，处理器只处理寄存器中的数据，从而可以避免多次访问存储器；
    
	（3）寻址方式非常简单，所有装载/保存的地址都只由寄存器内容和指令域决定；
    
	（4）使用统一和固定长度的指令格式。

ARM体系结构不仅仅具备RISC结构特性，还提供：

	（1）每一条数据处理指令都可以同时包含算术逻辑单元（ALU）的运算和移位处理，以实现对ALU和移位器的最大利用；
    
	（2）使用地址自动增加和自动减少的寻址方式优化程序中的循环处理；
    
	（3）load/store指令可以批量传输数据，从而实现了最大数据吞吐量；
    
	（4）大多数ARM指令是可“条件执行”的，也就是说只有当某个特定条件满足时指令才会被执行。通过使用条件执行，可以减少指令的数目，从而改善程序的执行效率和提高代码密度。


 这些在基本RISC结构上增强的特性使ARM处理器在高性能、低代码规模、低功耗和小的硅片尺寸方面取得良好的平衡。

### 2.1.3 ARM处理器常用系列

ARM体系结构从最初开发到现在有了很大的改进，并仍在完善和发展。ARM 微处理器目前包括下面几个系列：

	ARM7系列　ARM9系列　ARM9E系列　ARM10E系列
	SecurCore系列　Intel的StrongARM ARM11系列 Intel的Xscale
	
	其中，ARM7、ARM9、ARM9E和ARM10为4个通用处理器系列，每一个系列提供一套相对独特的性能来满足不同应用领域的需求。SecurCore系列专门为安全要求较高的应用而设计。

	Axxia 4500通信处理器基于采用28纳米工艺的ARM 4核Cortex-A15处理器，并搭载ARM全新CoreLink CCN-504高速缓存一致性互连技术，实现安全低功耗和最佳性能。

	ARM公司在经典处理器ARM11以后的产品改用Cortex命名，并分成A、R和M三类，旨在为各种不同的市场提供服务。

## 2.2基本编程模型

### 2.2.1 ARM处理器工作模式

ARM微处理器支持7种工作模式：用户模式、系统模式、快中断模式（Fast Interrupt Request）、一般中断模式（IRQ）、管理模式（Supervisor，SVC）、中止（abort）、中止（abort）。其中，除用户模式之外的其余6种称为特权模式（Privileged Modes）；即具有如下权利：

	1.MRS（把状态寄存器的内容放到通用寄存器中）；

	2.MSR（把通用寄存器的内容放到状态寄存器中）。

而在特权模式中，除系统模式之外的其余5种又称为异常模式（Exception Modes）。

### 2.2.2 ARM处理器工作状态

ARM微处理器的工作状态一般有两种，并可在两种状态之间切换：

ARM状态：此时处理器执行32位的字对齐的ARM指令；

Thumb状态：此时处理器执行16位的、半字对齐的Thumb指令。

### 2.2.3 ARM处理器寄存器组织

ARM指令中有37个物理寄存器，有31个通用寄存器和6个状态寄存器。但是这些寄存器不能被同时访问，具体哪些寄存器是可编程访问的，取决微处理器的工作状态及具体的运行模式。其中通用寄存器R14～R0、程序计数器PC、一个或两个状态寄存器都是可访问的。

1、arm状态下寄存器。

通用寄存器包括R0～R15，可以分为三类：

    （1）未分组寄存器R0～R7：在所有的运行模式下，未分组寄存器都指向同一个物理>寄存器，他们未被系统用作特殊的用途

    （2）分组寄存器R8～R14：对于分组寄存器，他们每一次所访问的物理寄存器与处理器当前的运行模式有关。

    （3）程序计数器PC(R15)：寄存器R15用作程序计数器（PC）

2、Thumb状态下的寄存器。

Thumb状态下的寄存器集是ARM状态下寄存器集的一个子集，程序可以直接访问8个通用寄存器（R7～R0）、程序计数器（PC）、堆栈指针（SP）、链接寄存器（LR）和CPSR。同时，在每一种特权模式下都有一组SP、LR和SPSR。

在Thumb状态下，高位寄存器R8～R15并不是标准寄存器集的一部分，但可使用汇编语言程序受限制地访问这些寄存器，将其用作快速的暂存器。使用带特殊变量的MOV指令，数据可以在低位寄存器和高位寄存器之间进行传送；高位寄存器的值可以使用CMP和ADD指令进行比较或加上低位寄存器中的值。

3、ARM状态和Thumb状态之间寄存器的关系。

    （1）Thumb状态R0～R7与ARM状态R0～R7相同；

    （2）Thumb状态CPSR和SPSR与ARM状态CPSR和SPSR相同；

    （3）Thumb状态SP映射到ARM状态R13；

    （4）Thumb状态LR映射到ARM状态R14；

    （5）Thumb状态PC映射到ARM状态PC（R15）。

**各种处理器模式下的寄存器**

用户模式 | 系统模式 | 特权模式 | 中止模式 | 未定义指令模式 | 外部中断模式|快速中断模式
--------| --------| ------- |  ------- | ------------- | -------| ------- 
R0 | R0 | R0 | R0 | R0 | R0 | R0 
R1 | R1 | R1 | R1 | R1 | R1 | R1 
R2 | R2 | R2 | R2 | R2 | R2 | R2 
R3 | R3 | R3 | R3 | R3 | R3 | R3 
R4 | R4 | R4 | R4 | R4 | R4 | R4 
R5 | R5 | R5 | R5 | R5 | R5 | R5 
R6 | R6 | R6 | R6 | R6 | R6 | R6 
R7 | R7 | R7 | R7 | R7 | R7 | R7 
R8 | R8 | R8 | R8 | R8 | R8 | R8_fiq
R9 | R9 | R9 | R9 | R9 | R9 | R9_fiq
R10| R10| R10| R10| R10| R10| R10_fiq
R11| R11| R11| R11| R11| R11| R11_fiq
R12| R12| R12| R12| R12| R12| R12_fiq
R13| R13|R13_svc|R13_abt|R13_und|R13_inq|R13_fiq
R14| R14|R14_svc|R14_abt|R14_und|R14_inq|R14_fiq
PC | PC | PC | PC | PC | PC | PC 
CPSR|CPSR|CPSR  SPSR_svc|CPSR  SPSR_abt|CPSR  SPSR_und|CPSR    SPSR_inq|CPSR  SPSR_fiq


### 2.2.4 ARM异常处理

当系统运行时，异常可能会随时发生，为保证在ARM处理器发生异常时不至于处于未知状态，在应用程序的设计中，首先要进行异常处理，采用的方式是在异常向量表中的特定位置放置一条跳转指令，跳转到异常处理程序，当ARM处理器发生异常时，程序计数器PC会被强制设置为对应的异常向量，从而跳转到异常处理程序，当异常处理完成以后，返回到主程序继续执行。
 