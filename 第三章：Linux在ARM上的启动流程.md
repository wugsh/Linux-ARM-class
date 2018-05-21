#第三章 Linux在ARM上的启动流程

嵌入式Linux系统加电后，经过约十几秒的时间，Linux内核就处于工作状
态了，并会执行很多应用程序，具体由系统初始化脚本指定。这期间的大部分动作都是由系统配置管理的，并处于嵌入式开发者的控制之下。

##3.1合成内核镜像

系统加电后，嵌入式系统中的引导加载程序首先获得处理器控制权。当它完成一些底层的硬件初始化后，会将控制权转交给Linux内核。

内核的构建过程中，不管采用哪种架构构建时都会生成一些通用文件，其中之一就是vmlinux的ELF二进制文件。这个二进制文件就是单体内核（monolithic kernel）本身，我们也称它为内核主体。

内核执行的第一行代码可以在一个名为head.S（或类似的名字）的汇编语言源文件中找到。有些架构和引导加载程序可以直接引导vmlinux镜像（将其格式从ELF转换成二进制之后）。对于其他架构和引导加载程序的组合，可能还需要一些额外的功能来建立合适的上下文，并提供必要的工具以加载和引导内核。

代码清单3-1显示了内核构建过程中的最后一些详细步骤，此次构建采用的硬件平台基于ADIEngineering公司的Coyote参考平台，这个平台包含英特尔的IXP425网络处理器。

	代码清单3-1 最后的内核构建步骤：ARM/IXP425
	$ make ARCH=arm CROSS_COMPILE=xscale_be- zImage
	...
	LD		vmlinux
	SYSMAP	System.map
	SYSMAP	.tmp_System.map
	OBJCOPY	arch/arm/boot/Image
	Kernel:	arch/arm/boot/Image is ready
	AS		arch/arm/boot/compressed/head.o
	GZIP	arch/arm/boot/compressed/piggy.gz
	AS		arch/arm/boot/compressed/piggy.o
	CC		arch/arm/boot/compressed/misc.o
	AS		arch/arm/boot/compressed/head-xscale.o
	AS		arch/arm/boot/compressed/big-endian.o
	LD		arch/arm/boot/compressed/vmlinux
	OBJCOPY	arch/arm/boot/zImage
	Kernel:	arch/arm/boot/zImage is ready

第三行中，构建系统链接生成了vmlinux镜像（内核主体）。之后，构建系统处理了很多其他对象模块。包括head.o、piggy.o以及与具体架构相关的head-xscale.o等。一般来说，这些对象模块都是和具体架构相关的（这个例子是ARM/IXP425），并且包含了一些底层函数，用于在特定架构上引导内核。

通过示意图帮助理解镜像的具体结构，图3-1显示了镜像的组成成员，以及在内核构建过程中它们如何改变形态，直至最终生成了一个可引导的内核镜像。
![图3-1](http://i.imgur.com/Qwew5xo.png)
###3.1.1 Image对象

Image对象是由vmlinux对象生成的。去除ELF文件vmlinux中的冗余段（标记和注释），并去掉所有可能存在的调试符号，就是Image了。下面的命令用于该用途：
	
	xscale_be-objcopy -O binary -R .note -R .note.gnu.build-id -R .comment -S  vmlinux arch/arm/boot/Image

总而言之，Image只不过是将内核主体从ELF转换成二进制形式，并去除了调试信息和前面提到的.note*和.comment段。

###3.1.2 与具体架构相关的对象

按照构建次序，接下来会编译很多小模块，包括几个由汇编语言源文件编译的对象（head.o、head-xscale.o等），它们完成底层具体和架构和处理器相关的一些任务。特别需要注意的是创建piggy.o对象的流程。首先使用gzip命令对Image文件（二进制内核镜像）进行压缩：
	
	cat Image | gzip -f -9 > piggy.gz

这个命令创建了一个名为piggy.gz的新文件，它只不过是二进制内核镜像Image的压缩版。接下来汇编器汇编名为piggy.S的汇编语言文件，而这个文件包含了一个对压缩文件piggy.gz的引用。从本质上说，二进制内核镜像以负载的形式依附在了一个底层的启动加载程序之上，采用汇编语言编写的。启动加载程序先初始化处理器和必需的内存区域，然后解压二进制内核镜像（piggy.gz）,并将解压后的内核镜像（Image）加载到系统内存的合适位置，最后将控制权转交给它。代码清单3-2显示了汇编语言源文件.../arch/arm/boot/compressed/piggy.S的完整内容。
	
	代码清单3-2 汇编文件piggy.S
	./section	.piggydata,#alloc
	./globl		input_data
	 input_data:
		.incbin	  "arch/arm/boot/compressed/piggy.gz"
		.globl	  input_data_end
	input_data_end:

这个汇编语言源文件虽然简短，但其中包含了不易被发现的复杂内容。汇编器汇编这个文件并生成一个ELF格式的镜像piggy.o，该镜像包含一个名为.piggydata的段，这个文件的作用是将压缩后的二进制内核镜像（piggy.gz）放到这个段中，成为其内容。该文件通过汇编器的预处理指令.incbin将piggy.gz包含进来，.incbin类似于C语言中的#include文件指令，只不过它包含的是二进制数据。总之，改汇编语言文件的作用是将压缩的二进制内核镜像（piggy.gz）放进另一个镜像中（piggy.o）中。

###3.1.3 启动加载程序

不要将它与引导加载程序混淆，很多架构都使用启动加载程序将Linux内核镜像加载到内存中。

引导加载程序和启动加载程序之间的区别也很简单：当硬件单板加电时，引导加载程序获得其控制权，根本不依赖于内核。相反，启动加载程序的主要作用是作为裸机引导加载程序和Linux内核之间的粘合剂。启动加载程序负责提供合适的上下文让内核运行于其中，并且执行必要的步骤以解压和重新部署内核二进制镜像。

下图3-2清晰地解释了这一概念，启动加载程序和内核镜像拼接在一起，用于加载。

![图3-2](http://i.imgur.com/rOlEvWi.png)



###3.1.4 引导信息

在PC自身的BIOS消息之后，我们会看到很多由Linux输出的控制台消息，表示它正在初始化各个内核子系统，引导信息中的大部分都与架构和机器类型无关。在早期的引导消息中，内核版本字符串和内核命令行是我们感兴趣的内容。

##3.2 初始化时的控制流

引导加载程序是一种底层软件，存储在系统的非易失性内存（闪存或ROM）中。当系统加电时它立刻获得控制权。它通常体积很小，包含一些简单的函数，主要用于底层的初始化、操作系统镜像的加载和系统诊断。最后，引导加载程序还包含一些处理逻辑，用于将控制权转交给另一个程序，一般是操作系统，比如Linux。

当系统第一次加电时，这个引导加载程序开始执行，然后会加载操作系统。当引导加载程序部署并加载了操作系统镜像之后，就将控制权转交给那个镜像。

下图3-3显示了ARM引导时的控制流。当启动加载程序完成它的工作后，将控制权转交给内核主体的head.o，之后再转到文件main.c中的函数start_kernel()。

![图3-3](http://i.imgur.com/izbV7su.png)
###3.2.1 内核入口：head.o

head.o这个模块是由汇编语言源文件head.S生成的，它的具体路径是.../arch/<ARCH>/kernel/head.S，其中<ARCH>替换成具体的架构名称。

head.o模块完成与架构和CPU相关的初始化，为内核主体的执行做好准备。与CPU相关的初始化工作尽可能地做到了在同系列处理器中通用。head.o还要执行下列底层任务：

1)检查处理器和架构的有效性；

2)创建初始的页表（page table）表项；

3)启用处理器的内存管理单元（MMU）；

4)运行错误检测并报告；

5)跳转到内核主体的起始位置，也就是文件main.c中的函数start_kernel()。

这些复杂的功能是由汇编语言实现的，并且涉及虚拟内存的硬件细节，相关讨论超出了本文的范围，但有几个点还是需要特别注意一下。

当启动加载程序将控制权第一次转交给内核的head.o时，处理器运行于我们过去常说的实地址模式（real mode）。处理器的程序计数器或其他类似寄存器中包含的地址称为逻辑地址（logical address），而处理器的内存地址引脚上的电信号地址值称为物理地址（physical address）。在实地址模式下，这两个值是相同的。为了启用内存地址转换，需要先初始化相关的寄存器和内核数据结构，当这些初始化完成后，就会开启处理器的MMU。当MMU的功能开启时，物理地址被替换成了逻辑地址。这就是调试器不能像调试普通代码一样单步调试这段代码的原因。

第二点需要注意的是，在内核引导过程的早期阶段，地址的映射范围是有限的。很多开发者都曾尝试修改head.o以适应特定平台，但因为这个限制的存在而犯错。假设你有一个硬件设备，而你需要在系统引导的早期阶段加载一个固件（firmware）。一种解决方案是将必需的固件静态编译到内核镜像中，然后使用一个指针引用它，并将它下载到你的设备中。然而，由于是在内核引导的早期阶段，其地址映射存在限制，很有可能固件镜像所处的位置超过这个范围。当代码执行时，它会产生一个页面错误（page fault），因为这时你想访问一个内存区域，但处理器内部还没有建立起对这块区域的有效映射。更糟糕的是，在早期阶段，页面错误处理程序还没有安装到位，所以最终结果会是莫名其妙的系统崩溃。在系统引导的早期阶段，不会有任何错误信息能够帮助你找到问题所在。

明智的做法是考虑尽可能推迟所有定制硬件的初始化工作，直到内核完成引导之后。用这种方式，你可以使用设备驱动程序模型来访问定制硬件，而不是去修改更复杂的汇编语言启动代码。如果你必须修改汇编语言早期启动代码，你需要在开发时间、费用和复杂度等方面付出更高的代价。硬件工程师和软件工程师应该在硬件开发的早期讨论这些现实问题，此时往往硬件设计的一个小的改动就能够节省大量的软件开发时间。

在虚拟内存环境中，开发人员会面临一些约束，认识到这点很重要，几乎所有主流的32位或更高位微处理器都包含了内存管理的硬件单元，用于实现虚拟内存架构。虚拟内存机有一个显著的优点，它能够帮助不同团队的开发人员编写大型复杂的应用程序，同时保护其他软件模块和内核本身，使它们不受编程错误的影响。

###3.2.2 内核启动：main.c

内核自身的head.o模块完成的最后一个任务就是将控制权转交给一个由C语言编写的，负责内核启动的源文件。
每个架构的head.o模块在转交控制权时会使用不同的方法，汇编语言的语法也不同，但它们的代码结构是类似的。对于ARM架构，如下所示

	b		start_kernel
对于Power架构，大致如下：

	lis		r4,start_kernel@h
	ori		r4,r4,start_kernel@l
	lis		r3,MSR_KERNEL@h
	ori		r3,r3,MSR_KERNEL@l
	mtspr	SRRO,r4
	mtspr	SRR1,r3
	rfi

这两个例子的结果是一样的。控制权从内核的第一个对象模块（head.o）转交至c语言函数start_kernel（），这个函数位于文件.../init/main.c中，内核从这里开始了新的生命旅程。任何想深入理解linux内核的人都应该仔细研究main.c文件，研究它是由哪些部分组成的，以及这些成员是如何初始化或实例化的。汇编语言之后的大部分Linux内核启动工作都是由main.c来完成的，从初始化第一个内核线程开始，直到挂载根文件系统并执行最初的用户空间linux应用程序。

函数start_kernel（）目前是main.c中最大的一个函数。大多数Linux内核初始化工作都是在这个函数中完成的。

###3.2.3 架构设置

.../init/main.c中的函数start_ kernel（）在其执行的开始阶段会调用setup_ arch()，而这个函数是在文件.../arch/arm/kernel/setup.c中定义的。该函数接受一个参数———一个指向内核命令行的指针：
	
	
	setup_arch(&command_line);

该语句调用一个与具体架构相关的设置函数，linux内核支持的每种主要架构都提供了这个函数，它负责完成那些对某种架构通用的初始化工作。函数setup_ arch（）会调用其他识别具体CPU的函数，并提供一种机制，用于调用高层高层特定CPU的初始化函数。setup_ processor（）是其中一个例子，由setup_ arch（）直接调用，位于文件.../arch/arm/kernel/setup.c中。这个函数会验证CPU的ID和版本，并调用特定的CPU初始化函数，同时会在系统引导时向控制台打印几行相关信息。

架构设置函数最后的工作中有一项是完成那些依赖于机器类型的初始化，不同架构采用的机制有所不同。对于ARM架构，可以在.../arch/arm/mach-*系列目录中找到与具体机器相关的初始化代码文件，具体的文件取决于机器类型。MIPS架构同样也包含很多目录，具体与它支持的硬件平台有关。对于Power架构，其platforms目录包含了特定机器的代码文件。

##3.3 内核命令行处理

在设置了架构之后，main.c开始执行一些通用的早期内核初始化工作，并显示内核命令行。

	Kernel command line:console=ttyS0,115200 root=/dev/nfs ip=dhcp

在这个简单的例子中，命令行中的console=ttyS0指示内核在串行端口设备ttyS0（一般是第一个串行端口）上打开一个控制台，波特率位115200bit/s。命令行中的ip=dhcp指示内核从一个DHCP服务器获取其初始IP地址，而root=/dev/nfs则是让内核通过NFS协议来挂载一个根文件系统。

Linux一般是由一个引导加载程序（或启动加载程序）启动的，它会向内核传递一系列参数，这些参数称为内核命令行。虽然你不会真正地使用shell命令行来启动内核，但很多引导加载程序都可以使用这种大家熟悉的形式向内核传递参数。在有些平台上，引导加载程序不能识别Linux，这时可以在编译的时候定义内核命令行，并硬编码到内核的二进制镜像中。在其他平台上（比如一个运行Debian的桌面PC），用户可以修改命令行的内容，而不需要重新编译内核。这时，bootsrap loader从一个配置文件生成内核命令行，并在系统引导时将它传递给内核。

内核命令行参数使用的基本语法规则很简单，有几种不同的形式，可以是单个单词，一个键/值对，或是key=value1,value2,...这种一个键和多个值的形式。这些参数是由其使用者具体进行处理的。命令行是全局可访问的，并且可以由很多模块处理。main.c中的函数start_kernel（）在调用函数setup_ arch（）时会传入内核命令行作为函数的参数，这也是唯一的参数。通过这个调用，与架构相关的参数和配置就被传递给了那些与架构和机器相关的代码。

设备驱动程序的编写者和内核开发人员都可以添加额外的内核命令行参数，以满足自身的具体需要。下面我们来看看这个机制。这里需要理解复杂的连接器脚本文件。

## __setup宏

我们希望在系统引导过程的早期阶段初始化控制台，以便我们可以在引导过程中将消息输出到这个目的设备上，初始化工作是由一个名为printk.o的内核对象完成的。控制台的初始化函数名为console_ setup（），这个函数只有内核命令行一个参数。

文件.../include/linux/init.h中定义了一个特殊的宏，用于将内核命令行字符串的一部分同某个函数关联起来，而这个函数会处理字符串的那个部分。你可以将__ setup宏看做是一个注册函数，即为内核命令行中的控制台相关参数注册的处理函数。实质上是指，当内核命令行中碰到console=字符串时，就调用__ setup宏的第二个参数所指定的函数，这里就调用函数console_ setup（）。但这个信息是如何传递给早期的设置代码的呢？这些代码在模块之外，也不了解控制台相关的函数。

具体的细节隐藏在一组宏当中，这些宏用于省去繁琐的语法规则，方便地将段属性添加到一部分目标代码中。这里的目标是建立一个静态列表，其中每一项包含一个字符串字面量及其关联的函数指针。这个列表由编译器生成，存放在一个单独命名的ELF段中，而该段是最终的ELF镜像vmlinux的一部分。理解这个技术很重要，内核中很多地方都是用这个技术来完成一些特殊的处理。

现在我们具体看一下__ setup宏的情况。代码清单3-3是头文件.../include/linux/init.h中的一部分代码，其中定义了__ setup系列的宏。
	
	代码清单3-3 init.h中__setup系列宏的定义
	...
	#define __setup_param(str, unique_id, fn, early)		\
		static const char __setup_str_##unique_id[] __initconst	\
			__aligned(1) = str;	\
		static struct obs_kernel_param __setup_##unique_id	\
			__used __section(.init.setup)			\
			__attribute__((aligned((sizeof(long)))))	\
			= { __setup_str__##unique_id, fn,early }

	#define __setup(str, fn)					\
		__setup_param(str, fn, fn, 0)
	...

 __ used宏指示编译器生成函数或者变量，即使是优化器不使用该函数或变量。__ attribute__((aligned))宏指示编译器将结构体对齐到一个特定的内存边界上---在这里是按sizeof(long)的长度对齐。

__ setup宏在展开后会生成一个obs_kernel_ param结构体，而这个结构体包含了一个用作标志的成员，结构体中这个标志名字是early，用于表示特定命令行参数是否在早期的引导过程中被处理过了。

##3.4 子系统的初始化

很多内核子系统的初始化工作都是由main.c中的代码完成的。有些是通过显式函数调用完成的，比如调用函数init_ timers（）和console_ init（），它们需要很早就被调用。其他子系统是通过一种非常类似__setup宏的技术来初始化的。简而言之，链接器首先构造一个函数指针的列表，其中的每个指针指向一个初始化的函数，然后在代码中使用一个简单的循环，依次执行这些函数。
	
	代码清单3-4 一个简单的初始化函数
	static init __init customize_machine(void)
	{
		/* 定制平台设备，或添加新设备 */
		if (init_machine)
			init_machine();
		return 0;
	}
	arch_initcall(customize_machine);

这段代码来自文件.../arch/arm/kernel/setup.c。该函数应用于对某个特定的硬件板卡进行一些定制的初始化。

##3.5 init线程

文件.../init/main.c中的代码负责赋予内核“生命”。函数start_ kernel（）执行基本的内核初始化，并显示调用一些早期初始化函数之后，就会生成第一个内核线程。这个线程就是成为init（）的内核线程，它的进程ID为1。init（）是所有用户空间Linux进程的父进程。系统引导到这个时间点时，有两个显著不同的线程正在运行：一个是函数start_kernel（）代表的线程，另一个就是init（）线程。前者在完成它的工作后会变成系统的空闲线程，而后者会变成init进程。

	代码清单3-5 内核init线程的创建	
	static noinline void __init_refok rest_init(void)
			__releases(kernel_lock)
	{
		int pid;

		rcu_scheduler_starting();
		kernel_thread(kernel_init, NULL，CLONE_FS | CLONE_SIGHAND);
		numa_default_policy();
		pid = kernel_thread(kthread, NULL, CLONE_FS | CLONE_FILES);
		kthreadd_task = find_task_by_pid_ns(pid,&init_pid_ns);
		unlock_kernel();

		/*
		 *启动时间空闲线程必须至少调用schedule()
		 *一次，以便系统运作
		 */
		init_idle_bootup_task(currrent);
		preempt_enable_no_resched();
		schedule();
		preempt_disable();

		/*当抢占禁用后调用cpu_idle
		cpu_idle();
	}

函数start_ kernel（）调用函数rest_ init（），内核的init进程是通过调用kernal_ process（）生成的，以函数kernel_ init作为其第一个参数。init会继续完成剩余的系统初始化，而执行函数start_ kernel（）的线程会在调用cpu_idle（）之后进入无限循环。

为什么要采用这样的结构？你也许已经发现了start_ kernel（）是个相当大的函数，而且是由__ init宏标记的。这意味着在内核初始化的最后阶段，它所占用的内存会被回收。而在回收内存之前，必须要退出这个函数和它所占的地址空间。解决的办法是让start__ kernel（）调用rest_ init（）。上面的代码显示了函数rest_ init（）的内容，这个函数相对来说小很多，它最终会变成系统的空闲进程。

###3.5.1 通过initcalls进行初始化

当执行函数kernel_ init（）的内核线程诞生后，它会最终调用do_ initcalls（）。*_ initcalls系列宏可以用于注册初始化函数，而函数do_ initcalls（）则负责调用其中大部分函数。

	代码清单3-6 通过initcalls进行初始化
		extern initcall_t __initcall_start[], __initcall_end[], __early_initcall_end[];

		static void __init do_initcalls(void)
		{
			initcall_t *fn;

			for (fn = __early_initcall_end; fn < __initcall_end; fn++)
				do_one_initcall(*fn);

			/*确保initcall序列中没有悬而未决的事情
			flush_scheduled_work();
		}

注意一下在初始化过程中有两处代码比较相似，一处是这里的do_ initcalls（），另一处是早前调用的do_pre_smp_ initcalls（）。这两个函数分别处理代码清单的不同部分，do_ pre_ smp_ initcalls（）处理从__ initcall_ start到__ early_ initcall_ end的部分，而do_ initcalls（）处理剩余的部分。

###3.5.2 initcall_debug

initcall_ debug是一个很有趣的内核命令行参数，它允许你观察启动过程中的函数调用。只需在启动内核时设置一下initcall_debug，就可以看到系统输出的相关诊断消息。

这里是一个输出信息的例子，启用调试语句后你会看到类似的信息：

	...
	calling uhci_hcd_init+0x0/0x100 @ 1
	uhci_hcd: USB Universal Host Controller Interface driver
	initcall uhci_hcd_init+0x0/0x100 returned 0 after 5639 usecs
	...

这里可以看到USB的通用主机控制器接口（Universal Host Controller Interface）驱动程序正在被调用。第一行通告了对函数uhci_ hcd_ init的调用，这是个设备驱动的初始化调用，来自USB驱动。通告之后，开始执行对这个函数的调用。第二行是有驱动程序本身打印的。第三行中的跟踪信息包含了函数的返回值以及函数调用的持续时间。

这是个查看内核初始化细节的好办法，特别是可以了解内核调用各个子系统和模块的顺序以及函数调用的持续时间。
如果你关心系统的启动时间，可以通过这种方法确定启动时间是在那里被消耗的。

###3.5.3 最后的引导步骤

在生成kernel_ init（）线程，并调用各个初始化函数之后，内核开始执行引导过程的最后一个步骤。包括释放初始化函数和数据所占用的内存，打开系统控制台设备，并启动第一个用户空间进程。代码清单3-7显示了内核的init进程执行的最后一些步骤，代码来自文件mian.c。

	代码清单3-7 最后的内核引导步骤
		static noinline int init_post(void)
			__releases(kernel_lock)
		{
		<... 为了简化，这里省略一些代码行 ...>
		...
		if (execute_command){
				run_init_process(execute_command);
				printk(KERN_WARNING "Failed to execute %s.	Attempting "
									"defaults...\n", execute_command);
		}

		run_init_process("/sbin/init");
		run_init_process("/etc/init");
		run_init_process("/bin/init");
		run_init_process("/bin/sh");

		panic("No init found.  Try passing init = option to kernel.");

		}

无论如何，run_ init_ process（）命令中至少有一个必须正确无误地执行。run_ init_ process（）函数在成功调用后不会返回。它用一个新的进程覆盖调用进程，实际上是用一个新的进程替换了当前进程。使用我们熟悉的execve（）系统调用来完成这个功能。最常见的系统配置一般会生成/sbin/init作为用户空间区地初始化进程。

	initcall_debug init=/sbin/myinit console=ttyS1,115200 root=/dev/hda1

这个内核命令行指示内核显示所有初始化函数的调用，配置初始地控制台设备为/dev/ttyS1，其数据速率为115Kbit/s，并执行一个定制的、名为myinit的用户空间初始化进程，这个程序位于根文件系统的/sbin目录中。它还指导内核从设备/dev/hda1挂载其根文件系统，这个设备是第一个IDE硬盘。一般来说，内核命令行中各个参数地先后次序无关紧要。


下面，我们将仔细研究内核初始化之后的系统初始化内容。
##3.6 根文件系统
尽管在没有文件系统的环境下Linux不存在问题，不过这么做不切实际，因为这样一来就无法充分利用Linux最重要的特性和价值，这就如同要将用户的整个系统应用塞进一个巨大的设备驱动程序或内核线程里。
根文件系统是指作为文件系统层次结构之基部而挂载的文件系统，简言之即“/”。实际上，根文件系统不过是挂载到文件系统层次结构基部的第一个文件系统。
###3.6.1 FHS
一些内核开发人员制定了一个标准，用来规定Unix文件系统的组织和部署。FHS（File System Hierarchy Standard，文件系统层次结构标准）为实现Linux各个发行版和应用程序之间的兼容性建立了最低基线。

许多Linux发行版的文件系统布局方式与FHS标准严格匹配。

###3.6.2 文件系统布局
出于空间的考虑，许多嵌入式系统开发人员会在一个可引导设备（例如Flash）上创建一个非常小的根文件系统，然后从另一个设备上挂载一个更大的文件系统，这里的设备可能是硬盘也可能是网络中的一个NFS服务器。

一个简单的Linux根文件系统可能包含如下顶层目录项：

    .
	|
	|--bin
	|--dev
	|--etc
	|--lib
	|--sbin
	|--usr
	|--var
	|--tmp

表3-1描述了上述各根目录项里最常见的内容。


通过斜杠（/）字符可以引用Linux文件系统层次结构的最顶层。例如，要列出根目录的内容，可以键入如下命令：

    $ ls /

输出结果类似如下：

    root@XXX：/# ls /
	bin dev etc home lib mnt opt proc root sbin tmp usr var
	root@XXX：/#

这个目录列表包含了用于其他用途的目录项，包括/mnt和/proc。注意，我们通过在前面加入“/”引用这些目录项，表示这些顶层目录的路径是从根目录开始的。

###3.6.3 最小文件系统

为了具体地说明一个根文件系统所必须的内容，我们创建一个最小的文件系统。这个例子是在一个使用XScale处理器的ADI Coyote评估板上实现的，代码清单3-8使用tree命令显示这个最小根文件系统的内容。

	代码清单3-8 一个最小的根文件系统的内容
    .
	|--bin
	|	|--busybox
	|	'--sh -> busybox
	|--dev
	|	'--console
	|--etc
	|	'--init.d
	|		'--rcS
	'--lib
		|--ld-2.3.2.so
		|--ld-linux.so.2 -> ld-2.3.2.so
		|--libc-2.3.2.so
		'--libc.so.6 -> libc-2.3.2.so

	5 directories, 8 files

该根文件系统配置使用了busybox，busybox是一个非常流行而且名符其实的嵌入式系统工具保。简言之，busybox是一个独立的二进制可执行文件，提供对大量常用Linux命令行实用工具的支持。

从/bin目标开始看，在其下面已经有了可执行的busybox文件和指向busybox的软链接（soft link）sh，你一会儿就会明白这样做的必要性。/dev目录下的文件是打开一个控制台进行输入/输出所需要的设备节点（device node）。/etc/init.d目录下的rcS文件是由busybox启动时处理的默认的初始化脚本文件，尽管该文件并不是必需的。使用rcS文件之后就不会出现busybox发出的警告信息，该警告信息只会在rcs文件缺失的时候才出现。

上述根文件系统必需的最后一个目录项及两个文件是两个库：GLIBC（libc-2.3.2.so）库和Linux动态加载器（ld-2.3.2.so）。GLIBC包括C标准库函数（如printf（））以及绝大多数应用程序所依赖的其他大量库函数；Linux动态加载器用于将二进制可执行文件加载到内存中，并且执行应用程序引用所需共享库函数的链接工作。这里包括的两个软连接是指向ld-2.3.2.so的ld-linux.so和指向libc-2.3.2.so的libc.so.6，这些链接使这些共享库不受版本影响并且具有向后兼容的特性，在所有Linux系统下都能看到这类链接。

这个简单的根文件系统实现了一个功能完整的系统。

##3.7 内核的最后引导过程

前面已经介绍了在系统引导最后阶段的内核动作，为方便起见，代码清单3-9列出了位于.../init/main.c的这最后一部分代码。
	
	代码清单3-9 最后的引导步骤
    ...
		if(execute_command){
				run_init_process(execute_command);
				printk(KERN_WARNING "Failed to execute %s.  Attempting "
				"defaults...\n",execute_command);
	}
	run_init_process("/sbin/init");
	run_init_process("/etc/init");
	run_init_process("/bin/init");
	run_init_process("/bin/sh");

	panic("No init found. Try passing init= option to kernel.");

在引导的最后阶段，内核会产生一个名为init的内核线程，上述代码正是该内核线程执行的最后一系列事件。

看一下上述代码，这意味着至少有一个run_init_process()函数调用必须成功执行，此外内核会试图按一定次序执行这四个程序。如果这四个程序没有一个执行成功，启动中的内核就会调用panic()函数并立即终止。需要记住的是，.../init/man.c中的这部分代码在引导阶段只执行一次，如果没有成功，内核除了通过调用panic()函数进行报错并终止引导过程之外，几乎不会再有任何动作。

###3.7.1 用户空间下第一个程序
在绝大部分Linux系统中，内核在启动过程中会执行/sbin/init，这也正是代码清单3-9中首先尝试执行/sbin/init的原因。实际上，这也造成了用户空间下第一个要运行的程序，可以回顾下面的引导次序；

1) 挂载根文件系统

2) 创建第一个用户空间下的程序，这里既是init。

在代码清单3-8所示的最小文件系统中，前三次创建用户空间程序的尝试都没有成功，因为在这个文件系统的任何地方都找不到一个名为init的可执行文件。回顾代码清单3-8，我们有一个名为sh的软链接，指向可执行文件busybox。现在你应该意识到这个软链接的作用了；它会使Linux将busybox作为初始化进程加以执行，同时也满足了在用户空间中的一个可执行shell的一般要求。

###3.7.2 解决依赖

简单地将一个init这样的可执行程序添加到用户文件系统并期望系统引导正常，还远远不够。对于添加到根文件系统的每一个程序，你还必须满足该程序的依赖。觉大部分程序有两种依赖：解析动态链接可执行程序的未定参考所需的文件；以及应用程序可能需要的外部配置文件或数据文件。对于前者，我们可以用一个工具找到相关文件；对于后者，只能通过大致了解出现问题的应用程序进行确认。

##3.8 init进程

除非你要做一些非常特殊的配置，否则永远不需要一个由用户定制的初始化程序，因为标准init进程的功能和使用非常灵活。init进程以及即将探讨的一系列启动脚本一起实现了通常所谓的System V Init，这一名源自最初使用这种模式的Unix System V。

###3.8.1 inittab
 
启动init进程会尝试着读取系统配置文件/etc/inittab，包含针对每个运行级及适用于所有运行级的指令。这个文件和init的行为在绝大多数Linux工作站的帮助手册里都有详细的记录，在一些关于Linux系统管理的书籍中也有详细描述。我们将重点着眼于开发人员如何为嵌入式系统配置inittab。
我们来看一个简单嵌入式系统的典型inittab文件。代码清单3-10是针对一个系统的inittab简单实例，该文件只支持一个运行级，实现系统的关机和重启。

	代码清单3-10 简单的inittab   
	# /etc/inittab

	# The default runlevel (2 in this example)
	id:2:initdefault:
	
	# This is the first process (actually a script) to be run.
	si::sysinit:/etc/rc.sysinit

	# Execute our shutdown script on entry to runlever 0
	10:0:wait:/etc/init.d/sys.shutdown

	# Execute our normal startup script on entering runlevel 2
	12:2:wait:/etc/init.d/runlvl2.startup

	# This line executes a reboot script (runlevel 6)
	16:6:wait:/etc/init.d/sys.reboot

	# This entry spawns a login shell on the console
	# Respawn means it will be restarted each time it is killed
	con:2:respawn:/bin/sh

这个非常简单的inittab脚本描述了3个不同的运行级，每个运行级都与一个脚本相关联，这些脚本必须是开发人员根据每个运行级所期望的动作而创建的。当init进程读取这个文件时，执行的第一个脚本是/etc/rc.sysinit，由标签sysinit表示。然后init进程进入运行级2，执行为运行级2定义的脚本，这个例子里即为脚本/etc/init.d/runlvl2.startup。正如代码清单3-10中的：wait：标签所示，init进程在该脚本执行完毕之前一直处于等待状态。在运行级2的脚本执行完毕后，init进程会在控制台中生成一个shell（通过符号链接/bin/sh)，如代码清单3-10最后一行所示。关键词respawn指示init进程一旦检测到shell退出便重新启动shell。代码清单3-11显示了启动期间的输出信息。

	代码清单3-11 启动消息示例
    ...
	VFS: Mounted root (nfs filesystem)
	Freeing init memory:304k
	INIT: vesion 2.78 booting
	This is rc.sysinit
	INIT: Entering runlevel:2
	This is runlvl2.startup

	#

这个例子里的启动脚本除了便于说明而打印自身被执行的信息外，不做其他任何事情。当然，在一个实际的系统中，这些脚本会启动若干功能和服务，完成一些有用的任务！就该例的这个简单配置文件而言，你可以在脚本/etc/init.d/runlvl2.startup里为特定的组件启动一些服务和应用程序，同时在关机或者重启脚本里执行逆操作，即终止这些应用程序、服务和设备。

##3.9 初始RAM磁盘

Linux内核提供了一些机制，能挂载一个早期的根文件系统来执行启动相关的系统初始化和配置任务；该机制即为初始RAM磁盘（简称initrd）。

###3.9.1 初始RAM磁盘的目的

初始RAM磁盘是一个功能完备的小型根文件系统，通常用来在系统引导过程结束之前加载一些特定的设备驱动模块。例如Red Hat和Fedora Core的Linux工作站发行版就使用了初始RAM磁盘，以便挂载真正的根文件系统之前加载EXT3文件系统相关的设备驱动。initrd一般用来加载访问真正的根文件系统所必需的设备驱动。

###3.9.2 构建initrd映象文件

构建一个合适的根文件系统映像是嵌入式系统开发所面临的挑战之一，而创建一个恰当的initrd映像则更具有挑战性，因为它要求更加简练、更加专业化。基于以上考虑，我们将在本节探讨initrd的要求以及initrd系统的内容。

使用tree命令来显示本章这个initrd映像实例中的内容，其内容如代码清单3-12所示。

	代码清单3-12 initrd示例内容
	|--bin
	|	|--busybox
	|	|--echo -> busybox
	|	|--mount -> busybox
	|	'--sh -> busybox
	|--dev
	|	|--console
	|	|--ram0
	|	'--ttySO
	|--etc
	|--linuxrc
	'--proc

	4 directories, 8 files

可以看到，其内容非常简练，解压后所有的文件加起来不到500KB。由于该initrd是基于busybox构建的，因此它具有很多功能。由于busybox是静态链接的，因此不依赖任何系统库。

##3.10 使用initramfs

initramfs是用来执行早期用户空间程序所采用的一种新机制，它与前面描述的initrd在概念上较为相似，其作用也很相似，即在挂载真正的根文件系统之前加载必要的设备驱动程序。然而，二者在某些方面还是有显著差别的。

从实用角度来看，initramfs更易使用。initramfs是一种cpio格式的档案文件，而initrd是一种gzipped格式的系统映像文件，二者这种简单的区别就使得initramfs更容易使用。当用户编译链接Linux内核映像时，initramfs会被集成到Linux内核源代码中并且被自动编译，此外对initramfs做出改变要比构建和加载一个新的initrd映像更容易一些。

##3.11 关机

有序关机操作在嵌入式系统的设计中曾一度被忽略，不正确的关机操作会影响到系统的启动时间，甚至会导致某些特定类型的文件系统崩溃。由于采用EXT2文件系统类型的系统在意外掉电后的重新启动过程中需要执行fsck（文件系统检查）命令，而执行该命令花费了太多的时间，这也成为了使用EXT2文件系统最多的抱怨。对于具有大容量磁盘的服务器来说，对几个大的EXT2分区进行证券的fsck操作可能要花费几个小时。

一个用于关机操作的脚本应该可以终止所有用户空间下的程序，最终关闭那些被进程打开的文件。如果init正在被使用中，那么执行init 0 命令会将系统挂起。通常来说，首先关机进程会向所有进程发送SIGTERM信号，通知他们系统正在执行关机操作。一段短暂的延时可以确保所有进程有机会执行自身的关闭操作。然后，向这些进程发送SIGKILL信号，最终彻底终止这些进程。
