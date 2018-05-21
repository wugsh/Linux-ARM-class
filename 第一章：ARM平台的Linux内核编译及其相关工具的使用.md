# 第一章： ARM 平台的 Linux 内核编译及其相关工具的使用 #

# Bootloader：uboot #
## 1.1. Bootloader ##
### 1.1.1. Bootloader的引入 ###
系统上电之后，需要一段程序来进行初始化：关闭WATCHDOG、改变系统时钟、初始化存储控制器、将更多的代码复制到内存中等等。如果它能将操作系统内核(无论从本地，比如Flash；还是从远端，比如通过网络)复制到内存中运行，就称这段程序为Bootloader。

简单地说，Bootloader就是这么一小段程序，它在系统上电时开始执行，初始化硬件设备、准备好软件环境，最后调用操作系统内核。

可以增强Bootloader的功能，比如增加网络功能、从PC上通过串口或网络下载文件、 
烧写文件、将Flash上压缩的文件解压后再运行等──这就是一个功能更为强大的Bootloader，也称为Monitor。实际上，在最终产品中用户 
并不需要这些功能，它们只是为了方便开发。

Bootloader的实现严重依赖于具体硬件，在嵌入式系统中硬件配置千差万别，即使是相 
同的CPU，它的外设(比如Flash)也可能不同，所以不可能有一个Bootloader支持所有的CPU、所有的电路板。即使是支持CPU架构比较多 
的U-Boot，也不是一拿来就可以使用的(除非里面的配置刚好与你的板子相同)，需要进行一些移植。
### 1.1.2. Bootloader的概念 ###
在一个嵌入式Linux系统中，从软件的角度通常可以分为4个层次：

（1）引导加载程序，包括固化在固件(firmware)中的 boot 代码(可选)和Bootloader两大部分。

有些CPU在运行Bootloader之前先运行一段固化的程序(固件，firmware)，比如x86结构的CPU就是先运行BIOS中的固件，然后才运行硬盘第一个分区(MBR)中的Bootloader。

在大多嵌入式系统中并没有固件，Bootloader是上电后执行的第一个程序。

（2）Linux内核。

特定于嵌入式板子的定制内核以及内核的启动参数。内核的启动参数可以是内核默认的，或是由Bootloader传递给它的。

（3）文件系统。

包括根文件系统和建立于Flash内存设备之上的文件系统。里面包含了Linux系统能够运行所必需的应用程序、库等，比如可以给用户提供操作Linux的控制界面的shell程序，动态连接的程序运行时需要的glibc或uClibc库，等等。

（4）用户应用程序。

特定于用户的应用程序，它们也存储在文件系统中。有时在用户应用程序和内核层之间可能还会包括一个嵌入式图形用户界面。常用的嵌入式 GUI 有：Qtopia 和 MiniGUI 等。
### 1.1.3. Bootloader的启动方式 ###
CPU上电后，会从某个地址开始执行。比如MIPS结构的CPU会从0xBFC00000取 
第一条指令，而ARM结构的CPU则从地址0x0000000开始。嵌入式单板中，需要把存储器件ROM或Flash等映射到这个地 
址，Bootloader就存放在这个地址开始处，这样一上电就可以执行。

在开发时，通常需要使用各种命令操作Bootloader，一般通过串口来连接PC和开发 
板，可以在串口上输入各种命令、观察运行结果等。这也只是对开发人员才有意义，用户使用产品时是不用接串口来控制Bootloader的。从这个观点来 
看，Bootloader可以分为两种操作模式(Operation Mode)：

（1）启动加载(Boot loading)模式。

上电后，Bootloader从板子上的某个固态存储设备上将操作系统加载到RAM中运行，整个过程并没有用户的介入。产品发布时，Bootloader工作在这种模式下。

（2）下载(Downloading)模式。

在这种模式下，开发人员可以使用各种命令，通过串口连接或网络连接等通信手段从主机(Host)下载文件(比如内核映像、文件系统映像)，将它们直接放在内存运行或是烧入Flash类固态存储设备中。

板子与主机间传输文件时，可以使用串口的xmodem/ymodem/zmodem协议，它们使用简单，只是速度比较慢；还可以使用网络通过tftp、nfs协议来传输，这时，主机上要开启tftp、nfs服务；还有其他方法，比如USB等。

像Blob或U-Boot等这样功能强大的Bootloader通常同时支持这两种工作模 
式，而且允许用户在这两种工作模式之间进行切换。比如，U-Boot在启动时处于正常的启动加载模式，但是它会延时若干秒(这可以设置)等待终端用户按下 
任意键而将U-Boot切换到下载模式。如果在指定时间内没有用户按键，则U-Boot继续启动Linux内核。
### 1.1.4. Bootloader的启动过程 ###
Bootloader的启动过程启动过程可以分为单阶段(Single 
Stage)、多阶段(Multi-Stage)两种。通常多阶段的Bootloader能提供更为复杂的功能，以及更好的可移植性。从固态存储设备上启 
动的Bootloader大多都是 2 阶段的启动过程。这从前面的硬件实验可以很好地理解这点：第一阶段使用汇编来实现，它完成一些依赖于 CPU 
体系结构的初始化，并调用第二阶段的代码。第二阶段则通常使用C语言来实现，这样可以实现更复杂的功能，而且代码会有更好的可读性和可移植性。

（1）Bootloader第一阶段的功能。

硬件设备初始化。

为加载Bootloader的第二阶段代码准备RAM空间。

拷贝Bootloader的第二阶段代码到 RAM 空间中。

设置堆栈。

跳转到第二阶段代码的C入口点。

在第一阶段进行的硬件初始化一般包括：关闭WATCHDOG、关中断、设置CPU的速度和时钟频率、RAM初始化等。这些并不都是必需的，比如S3C2410/S3C2440的开发板所使用的U-Boot中，就将CPU的速度和时钟频率的设置放在第二阶段。

甚至，将第二阶段的代码复制到RAM空间中也不是必需的，对于NOR Flash等存储设备，完全可以在上面直接执行代码，只不过这相比在RAM中执行效率大为降低。

（2）Bootloader第二阶段的功能。

初始化本阶段要使用到的硬件设备。

检测系统内存映射(memory map)。

将内核映像和根文件系统映像从Flash上读到RAM空间中。

为内核设置启动参数。

调用内核。

为了方便开发，至少要初始化一个串口以便程序员与Bootloader进行交互。

所谓检测内存映射，就是确定板上使用了多少内存，它们的地址空间是什么。由于嵌入式开发中，Bootloader多是针对某类板子进行编写，所以可以根据板子的情况直接设置，不需要考虑可以适用于各类情况的复杂算法。

Flash上的内核映像有可能是经过压缩的，在读到RAM之后，还需要进行解压。当然，对于有自解压功能的内核，不需要Bootloader来解压。

将根文件系统映像复制到RAM中，这不是必需的。这取决于是什么类型的根文件系统，以及内核访问它的方法。
## 1.2. U-Boot ##
### 1.2.1. 简介 ###
U-Boot，全称为Universal Boot 
Loader，即通用Bootloader，是遵循GPL条款的开放源代码项目。其前身是由德国DENX软件工程中心的Wolfgang 
Denk基于8xxROM的源码创建的PPCBOOT工程。后来整理代码结构使得非常容易增加其他类型的开发板、其他架构的CPU(原来只支持 
PowerPC)；增加更多的功能，比如启动Linux、下载S-Record格式的文件、通过网络启动、通过PCMCIA/CompactFLash 
/ATA disk/SCSI等方式启动。增加ARM架构CPU及其他更多CPU的支持后，改名为U-Boot。

它的名字“通用”有两层含义：可以引导多种操作系统、支持多种架构的CPU。它支持如下操作 
系统：Linux、NetBSD、 
VxWorks、QNX、RTEMS、ARTOS、LynxOS等，支持如下架构的CPU：PowerPC、MIPS、x86、ARM、NIOS、 
XScale等。

可以从http://sourceforge.net/projects/u-boot获得U-Boot的最新版本，如果使用过程中碰到问题或是发现Bug，可以通过邮件列表网站http://lists.sourceforge.net/lists/listinfo/u-boot-users/获得帮助。最新的更新代码地址http://www.denx.de/wiki/U-Boot/WebHome
### 1.2.2. U-Boot的目录结构 ###
![](http://i.imgur.com/Up1MvWu.png)
### 1.2.3. U-Boot的启动过程 ###
1. 在Flash中运行汇编程序，将Flash中的启动代码部分复制到SDRAM中，同时创造环境准备运行C程序
2. 在SDRAM中执行，对硬件进行初始化
3. 设置内核参数的标记列表，复制镜像文件，进入内核的入口函数

### 1.2.4. U-Boot的配置编译过程 ###
（1）确定开发板名称BOARD_NAME。

（2）创建到平台/开发板相关的头文件的链接。

（3）创建顶层Makefile包含的文件include/config.mk。

（4）创建开发板相关的头文件include/config.h。

总结一下：配置命令“make 
smdk2410_config”，实际的作用就是执行“./mkconfig smdk2410 arm arm920t smdk2410 
NULL s3c24x0”命令。假设执行“./mkconfig $1 $2 $3 $4 $5 $6”命令。

（1）开发板名称BOARD_NAME等于$1；

（2）创建到平台/开发板相关的头文件的链接：

    ln -s asm-$2 asm
    
    ln -s arch-$6 asm-$2/arch
    
    ln -s proc-armv asm-$2/proc　 如果$2不是arm的话，此行没有

(3) 创建顶层Makefile包含的文件include/config.mk。

    ARCH = $2
    
    CPU = $3
    
    BOARD = $4

`VENDOR = $5` 　$5为空，或者是NULL的话，此行没有

`SOC = $6　`　 $6为空，或者是NULL的话，此行没有

（4）创建开发板相关的头文件include/config.h。

    /* Automatically generated - do not edit */
    
    #include <configs/$1.h>"
配置完后，执行“make all”即可编译。

### 1.2.5. U-Boot的移植 ###
1. U-Boot移植参考版
2. U-Boot烧写地址
3. CPU寄存器参数设置
4. 串口调试
5. 与启动Flash相关的寄存器BR0、OR0的参数设置
6. CPLD电路
7. SDRAM驱动
8. 补充功能的添加

### 1.2.6. U-Boot的基本命令 ###
（1）帮助命令help。

运行help命令可以看到U-Boot中所有命令的作用，如果要查看某个命令的使用方法，运行“help 命令名”，比如“help bootm”。

可以使用“?”来代替“help”，比如直接输入“?”、“? bootm”。

（2）下载命令。

U-Boot支持串口下载、网络下载，相关命令有：loadb、loads、loadx、loady和tftpboot、nfs。

前几个串口下载命令使用方法相似，以loadx命令为例，它的用法为“loadx [ 
off ] [ baud 
]”。中括号“[]”表示里面的参数可以省略，off表示文件下载后存放的内存地址，baud表示使用的波特率。如果baud参数省略，则使用当前的波特 
率；如果off参数省略，存放的地址为配置文件中定义的宏CFG_LOAD_ADDR。

tftpboot命令使用TFTP协议从服务器下载文件，服务器的IP地址为环境变量 
serverip。用法为“tftpboot [loadAddress] 
[bootfilename]”，loadAddress表示文件下载后存放的内存地址，bootfilename表示要下载的文件的名称。如果 
loadAddress省略，存放的地址为配置文件中定义的宏CFG_LOAD_ADDR；如果bootfilename省略，则使用单板的IP地址构造 
一个文件名，比如单板IP为192.168.1.17，则缺省的文件名为C0A80711.img。

nfs命令使用NFS协议下载文件，用法为“nfs [loadAddress] 
[host ip 
addr:bootfilename]”。loadAddress、bootfilename的意义与tftpboot命令一样，host ip 
addr表示服务器的IP地址，默认为环境变量serverip。

下载文件成功后，U-Boot会自动创建或更新环境变量filesize，它表示下载的文件的长度，可以在后续命令中使用“$(filesize)”来引用它。

（3）内存操作命令。

常用的命令有：查看内存命令md、修改内存命令md、填充内存命令mw、拷贝命令cp。这些 
命令都可以带上后缀“.b”、“.w”或“.l”，表示以字节、字(2个字节)、双字(4个字节)为单位进行操作。比如“cp.l 30000000 
31000000 2”将从开始地址0x30000000处，拷贝2个双字到开始地址为0x31000000的地方。

md命令用法为“md[.b, .w, .l] address [count]”，表示以字节、字或双字(默认为双字)为单位，显示从地址address开始的内存数据，显示的数据个数为count。

mm命令用法为“mm[.b, .w, .l] address”，表示以字节、字或双字(默认为双字)为单位，从地址address开始修改内存数据。执行mm命令后，输入新数据后回车，地址会自动增加，Ctrl+C退出。

mw命令用法为“mw[.b, .w, .l] address value [count]”，表示以字节、字或双字(默认为双字)为单位，往开始地址为address的内存中填充count个数据，数据值为value。

cp命令用法为“cp[.b, .w, .l] source target count”，表示以字节、字或双字(默认为双字)为单位，从源地址source的内存拷贝count个数据到目的地址的内存。

（4）NOR Flash操作命令。

常用的命令有查看Flash信息的flinfo命令、加/解写保护命令protect、擦除 
命令erase。由于NOR Flash的接口与一般内存相似，所以一些内存命令可以在NOR Flash上使用，比如读NOR 
Flash时可以使用md、cp命令，写NOR Flash时可以使用cp命令(cp根据地址分辨出是NOR Flash，从而调用NOR 
Flash驱动完成写操作)。

直接运行“flinfo”即可看到NOR Flash的信息，有NOR Flash的型号、容量、各扇区的开始地址、是否只读等信息。比如对于本书基于的开发板，flinfo命令的结果如下：

Bank # 1: AMD: 1x Amd29LV800BB (8Mbit)

Size: 1 MB in 19 Sectors

Sector Start Addresses:

00000000 (RO) 00004000 (RO) 00006000 (RO) 00008000 (RO) 00010000 (RO)

00020000 (RO) 00030000 00040000 00050000 00060000

00070000 00080000 00090000 000A0000 000B0000

000C0000 000D0000 000E0000 000F0000 (RO)

其中的RO表示该扇区处于写保护状态，只读。

对于只读的扇区，在擦除、烧写它之前，要先解除写保护。最简单的命令为“protect off all”，解除所有NOR Flash的写保护。

erase命令常用的格式为“erase start 
end”──擦除的地址范围为start至end、“erase start +len”──擦除的地址范围为start至(start + len 
– 1)，“erase all”──表示擦除所有NOR Flash。

注意：其中的地址范围，刚好是一个扇区的开始地址到另一个(或同一个)扇区的结束地址。比如要擦除Amd29LV800BB的前5个扇区，执行的命令为“erase 0 0x2ffff”，而非“erase 0 0x30000”。

（5）NAND Flash操作命令。

NAND Flash操作命令只有一个：nand，它根据不同的参数进行不同操作，比如擦除、读取、烧写等。

“nand info”查看NAND Flash信息。

“nand erase [clean] [off size]”擦除NAND 
Flash。加上“clean”时，表示在每个块的第一个扇区的OOB区加写入清除标记；off、size表示要擦除的开始偏移地址和长度，如果省略 
off和size，表示要擦除整个NAND Flash。

“nand read[.jffs2] addr off size”从NAND Flash偏移地址off处读出size个字节的数据，存放到开始地址为addr的内存中。是否加后缀“.jffs”的差别只是读操作时的ECC较验方法不同。

“nand write[.jffs2] addr off size”把开始地址为addr的内存中的size个字节数据，写到NAND Flash的偏移地址off处。是否加后缀“.jffs”的差别只是写操作时的ECC较验方法不同。

“nand read.yaffs addr off size”从NAND Flash偏移地址off处读出size个字节的数据(包括OOB区域），存放到开始地址为addr的内存中。

“nand write.yaffs addr off size”把开始地址为addr的内存中的size个字节数据(其中有要写入OOB区域的数据），写到NAND Flash的偏移地址off处。

“nand dump off”，将NAND Flash偏移地址off的一个扇区的数据打印出来，包括OOB数据。

（6）环境变量命令。

“printenv”命令打印全部环境变量，“printenv name1 name2 ...”打印名字为name1、name2、……”的环境变量。

“setenv name value”设置名字为name的环境变量的值为value。

“setenv name”删除名字为name的环境变量。

上面的设置、删除操作只是在内存中进行，“saveenv”将更改后的所有环境变量写入NOR Flash中。

（7）启动命令。

不带参数的“boot”、“bootm”命令都是执行环境变量bootcmd所指定的命令。

“bootm [addr [arg 
...]]”命令启动存放在地址addr处的U-Boot格式的映像文件(使用U-Boot目录tools下的mkimage工具制作得到)，[arg 
...]表示参数。如果addr参数省略，映像文件所在地址为配置文件中定义的宏CFG_LOAD_ADDR。

“go addr [arg ...]”与bootm命令类似，启动存放在地址addr处的二进制文件， [arg ...]表示参数。

“nboot [[[loadAddr] dev] offset]”命令将NAND 
Flash设备dev上偏移地址off处的映像文件复制到内存loadAddr处，然后，如果环境变量autostart的值为“yes”，就启动这个映 
像。如果loadAddr参数省略，存放地址为配置文件中定义的宏CFG_LOAD_ADDR；如果dev参数省略，则它的取值为环境变量 
bootdevice的值；如果offset参数省略，则默认为0。

# Builtroot使用教程 #
## 1.3. 获取Builtroot ##
从buildroot官网(http://buildroot.uclibc.org/download.html)获取buildroot源码包，buildroot基本上三个月更新一次，这里实际下载的源码包是buildroot-2015.02.tar.gz
## 1.4. 配置Builtroot ##
进入源码包目录

    tar -xvf buildroot-2015.02.tar.gz
    cd buildroot-2015.02
开始配置

    make menuconfig
配置界面如下:

![](http://i.imgur.com/z10WBzH.png)

### 1.4.1. 进入target options ###
将Target Architecture配置为ARM(littlt endian)，将Target Architecture Variant配置为cortex-A9，将Target ABI配置为EABI，将ARM instruction set配置为ARM，再退回上一界面

![](http://i.imgur.com/jVOZIqg.png)

### 1.4.2. 进入toolchain ###
将Toolchaintype配置为Externaltoolchain，然后在Toolchain中选择交叉编译工具的版本，如ARM 2013.11，在Toolchain origin中选择Toolchain to be downloaded andinstalled，后面编译时，buildroot将会自动下载对应的工具链并自动安装。选中Enable MMUsupport，退回上一界面 

![](http://i.imgur.com/QhjV7oX.png)

### 1.4.3. 进入System configuration  ###
在system hostname一栏中输入开发板的名称，如metal box，在system banner中可输入欢迎语，如welcome to metal world。在Init system中选择BusyBox，在/dev management中选择Dynamic using mdev，即使用mdev动态加载设备节点的方式，然后在Path to thepermission tables中选择设备节点的配置表，这里我们一定要选择system/device_table_dev.txt，否则后面在dev目录下将不会生成各种设备节点。当然我们也可以手动的配置该文件，添加必要的节点或删除不需要的节点。Root password为配置进入Linux控制台终端后的密码，为空则登录时不需要密码，默认登录用户名为root。选中Run agetty(login prompt)after boot。 

![](http://i.imgur.com/65DJoIc.png)

再进入下面的getty options选项： 
将TTY port配置为ttySAC3，将baudrate配置为115200，对应开发板的打印串口。 

![](http://i.imgur.com/ABv1Uqw.png)

再返回上一界面，将Root filesystem overlay directories设置为board/metalboard/exynos4412/rootfs-overlay，这里表示该路径下的所有文件将会无条件覆盖buildroot默认的相关路径文件。配置这一步的同时，我们一并将开发板光盘中的相关文件拷贝到buildroot对应的board目录。返回上一界面。 

![](http://i.imgur.com/AVN2o3q.png)

### 1.4.4. 进入Filesystem images  ###

选中ext2/3/4root filesystem，然后在ext2/3/4variant中选择ext4，选中tar the root filesystem，最后保存当前的配置并退出，配置完成。大家也可以根据自己的实际需要进行配置。 

![](http://i.imgur.com/w9Mw36R.png)

## 1.5. 编译buildroot ##
只需在buildroot的根目录下执行make指令即可编译整个buildroot。第一次编译可能会弹出一些错误，这基本上是没有安装一些第三方工具造成的。按照提示安装即可，有问题问度娘。 
开始编译的时候，buildroot会自动下载所需要的相关源码包，自动编译安装。
### 1.5.1. 下载的源码包在buildroot根目录的dl目录下 ###

![](http://i.imgur.com/aXcvNf3.png)

### 1.5.2. 编译出来的各种文件会放在buildroot目录下面的output目录 ###

![](http://i.imgur.com/UVuYbJv.png)

###  1.5.3. 需要烧写的最终的映像文件在output/images目录下 ###

![](http://i.imgur.com/r0pN5pU.png)

###  1.5.4. output/target目录下为对应未打包的文件系统，在调试时可借助于该目录下的文件分析原因 ###

![](http://i.imgur.com/kz9euhC.png)

# Busybox的使用教程 #

## 1.6. Busybox简介 ##

BusyBox 将许多具有共性的小版本的UNIX工具结合到一个单一的可执行文件,主要应用于嵌入式linux系统，是一个开源的“万能工具”，下面简单介绍其使用方法。

### 1.6.1. 下载busybox和linux kernel的源码 ###

busybox的源码地址: http://www.linuxidc.com/Linux/2011-08/40704.htm

linux kernel的源码地址: http://www.kernel.org/pub/linux/kernel/v2.6/

我选择的busybox版本是: busybox-1.16.0.tar.bz2

linux kernel的版本是: linux-2.6.28.tar.bz2



### 1.6.2 配置编译linux内核 ###

将下载下来的内核源代码压缩包拷贝到/usr/src目录下，然后进入到这个目录将其解压,命令如下:

    # tar xjf linux-2.6.28.tar.bz2

然后创建一个目录，用来保存编译内核产生的目标文件

    # pwd
    
    /usr/src
    
    # mkdir linux-2.6.28-obj

执行完上述命令，/usr/src目录会有如下图所示:

![](http://i.imgur.com/r2RV6XL.png)

linux-2.6.28-obj现在是一个空目录，编译内核时会将目标文件输出保存到这个目录下。

linux-2.6.28 是刚才linux-2.6.28.tar.bz2文件解压出来的目录。

然后我们开始编译linux内核,输入如下所示的命令:

`# cd /usr/src/linux-2.6.28`  (进入到内核源码树目录)

`# make O=/usr/src/linux-2.6.28-obj menuconfig` (配置内核)

配置内核时，里面的选项有很多,如果不确定的话就将所有选项都编译进内核，当然最好能针对性的配置内核，这样产生出的内核镜像不至于太大。还有一点就是配置时一定要将选定的选项编译进内核，而不要编译成模块。同时，为了支持initrd内存盘文件系统，有两个选项是必须的。

一个是General Setup –> Initial RAM filesystem and RAM disk support

另一个是 Device Drivers –> Block Devices –> RAM block device support

这个选项的子选项保持默认就可以了，如下图所示:

![](http://i.imgur.com/haRClSf.png)

然后退出配置界面，在退出时会提示你是否保存刚才的配置，选择yes就可以了(因为我们在配置时指明了O=/usr/src/linux-2.6.28-obj 目录，所以配置文件会保存到这个目录下，文件名为.config)

接下来我们开始编译内核:

`# make O=/usr/src/linux-2.6.28-obj`  (生成内核镜像和模块)

通常，我们编译内核是为了更新内核，但这里我们只是为了编译出一个内核镜像，所以就不调用make install命令来安装内核了。

内核编译完成，将编译好的内核镜像拷贝到主目录下，以供后面使用。

`# cp /usr/src/linux-2.6.28-obj/arch/x86/boot/bzImage` ~ (拷贝内核镜像到root用户的主目录下)

- 编译busybox

`# tar xf busybox-1.16.0.tar.bz2` (解压busybox压缩包)

`# cd busybox-1.16.0` (进入到解压后的busybox源码目录)

`# make menuconfig` (配置busybox)

注意配置时，一定要选择静态链接选项，该选项位于:

Busybox Settings –> Build Options –> Build Busybox as a static binary

接下来安装busybox：

`# make install `(busybox默认安装到了其源码树目录的名字为_install的目录中)

`# cd _install` (进入安装了busybox的目录)

- 在busybox中添加配置文件并生成initrd镜像

`# mkdir proc sys etc dev` (创建四个空目录，linux内核需要)

   ` # cd dev`

`# mknod console c 5 1` (创建一个控制台字符设备文件)

`# mknod null c 1 3` (创建一个0设备文件)

    # cd ..

    # cd etc

`# vim fstab `(输出如下图内容)

![](http://i.imgur.com/ZAdVVDI.png)

    # mkdir init.d

`# vim init.d/rcS` (输出如下内容)

![](http://i.imgur.com/NyS8cPo.png)


`# chmod +x init.d/rcS` (给rcS文件加上可执行权限)

`# vim inittab` (输入如下内容)

![](http://i.imgur.com/EEl1OEL.png)

    # cd ..

`# pwd` (打印当前目录)

/root/busybox-1.16.0/_install

此时表明处在busybox安装文件的根目录下

`# rm linuxrc`(删除linuxrc链接文件)

然后新创建一个指向busybox文件的链接文件，如下图所示：

![](http://i.imgur.com/zqjyUT2.png)
    
    # find . | cpio --quiet -H newc -o | gzip -9 -n > ../initrd.gz

    # cd ..

`# cp initrd.gz` ~ (将其拷贝到主目录)

得到两个镜像文件：

bzImage : linux内核镜像文件

initrd.gz : 内存盘根文件系统镜像文件

### 1.6.3 grub引导和启动 ###

将生成的这两个文件拷贝到了/boot下，在grub提示符下输入如下图所示的三个命令：

![](http://i.imgur.com/SIGlPpQ.png)


启动后如下图所示：

![](http://i.imgur.com/lVng3nN.png)

# 交叉编译工具 #
## 1.7. 交叉编译工具简介 ##
在一种计算机环境中运行的编译程序，能编译出在另外一种环境下运行的代码，这个编译过程就叫交叉编译。简单地说，就是在一个平台上编译生成另一个平台上的可执行代码或程序。

交叉编译工具链是一个由编译器、连接器和解释器组成的综合开发环境，交叉编译工具链主要由binutils、gcc和glibc三个部分组成。有时出于减小 libc 库大小的考虑，也可以用别的 c 库来代替 glibc，例如 uClibc、dietlibc 和 newlib。
### 1.7.1. 为什么要使用交叉编译工具 ###
ARM上可以运行操作系统，所以用户完全可以将ARM当做计算机来使用，理论上也可以在ARM上使用本地的编译器来编译程序.但是，编译器在编译程序时，会产生大量的中间文件，这会占用很大的内存和磁盘空间，且对CPU处理速度要求较高，比如S3C2440A内存、磁盘空间只有几十到100多兆，CPU只有400-500MHz，完全达不到编译程序的要求.所以，在进行ARM-linux嵌入式开发时必须在PC机(x86结构)上编译出能够运行在ARM上的程序，然后再将程序下载到ARM中来运行.这就用到了交叉编译器.![](http://i.imgur.com/5uo131e.png)
### 1.7.2. 体系结构与操作系统 ###
(1)常见的体系结构有ARM结构、x86结构等。

(2)常见的操作系统有linux,windows等。

(3)同一个体系结构可以运行不同操作系统，如x86上可以运行Linux、Windows等，在ARM上可以运行Linux、WinCE。

(4)同一个操作系统可以在不同的体系结构上运行，比如Linux可以运行在x86上，也可以运行在ARM上。

(5)同样的程序不可能运行在多个平台上，比如Windows下应用程序不能在Linux下运行.如果一个应用程序想在另一个平台上运行，必须使用针对该平台的编译器，来重新编译该应用程序的二进制代码。
### 1.7.3. 命名规则 ###
交叉编译工具的命名规则为：arch [-vendor] [-os] [-(gnu)eabi]

- arch - 体系架构，如ARM，MIPS
- vendor - 工具链提供商
- os - 目标操作系统
- eabi - 嵌入式应用二进制接口（Embedded Application Binary Interface）

根据对操作系统的支持与否，ARM GCC可分为支持和不支持操作系统。
### 1.7.4. 实例 ###
1.arm-none-eabi-gcc

用于编译 ARM 架构的裸机系统（包括 ARM Linux 的 boot、kernel，不适用编译 Linux 应用 Application），一般适合 ARM7、Cortex-M 和 Cortex-R 内核的芯片使用，所以不支持那些跟操作系统关系密切的函数，比如fork(2)，他使用的是 newlib 这个专用于嵌入式系统的C库。

2.arm-none-linux-gnueabi-gcc

主要用于基于ARM架构的Linux系统，可用于编译 ARM 架构的 u-boot、Linux内核、linux应用等。arm-none-linux-gnueabi基于GCC，使用Glibc库，经过 Codesourcery 公司优化过推出的编译器。arm-none-linux-gnueabi-xxx 交叉编译工具的浮点运算非常优秀。一般ARM9、ARM11、Cortex-A 内核，带有 Linux 操作系统的会用到。

3.arm-eabi-gcc

Android ARM 编译器。

4.armcc

ARM 公司推出的编译工具，功能和 arm-none-eabi 类似，可以编译裸机程序（u-boot、kernel），但是不能编译 Linux 应用程序。armcc一般和ARM开发工具一起，Keil MDK、ADS、RVDS和DS-5中的编译器都是armcc。

5.arm-linux-gnueabi-gcc 和 arm-linux-gnueabihf-gcc

softfp： armel架构（对应的编译器为 arm-linux-gnueabi-gcc ）采用的默认值，用fpu计算，但是传参数用普通寄存器传，这样中断的时候，只需要保存普通寄存器，中断负荷小，但是参数需要转换成浮点的再计算。

hard： armhf架构（对应的编译器 arm-linux-gnueabihf-gcc ）采用的默认值，用fpu计算，传参数也用fpu中的浮点寄存器传，省去了转换，性能最好，但是中断负荷高。

6.arm-none-uclinuxeabi-gcc 和 arm-none-symbianelf-gcc

arm-none-uclinuxeabi 用于uCLinux。

arm-none-symbianelf 用于symbian。



