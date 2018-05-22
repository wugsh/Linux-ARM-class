#QEMU 调试ARM Linux内核
##1. 配置交叉编译环境
##2. 配置内核
    make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm  vexpress_defconfig
    make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm  menuconfig 

确保编译的内核包含调试信息：
![](https://i.imgur.com/VSHgEIv.jpg)

##3. 编译内核

    make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm

##4. 启动内核

    qemu-system-arm -M vexpress-a9 -smp 4 -m 1024M -kernel \
 	arch/arm/boot/zImage -append "rdinit=/linuxrc console=ttyAMA0 loglevel=8" \
	-dtb arch/arm/boot/dts/vexpress-v2p-ca9.dtb -S -s



> -S: 表示QEMU虚拟机会冻结CPU，直到远程的GDB输入相应控制命令。
> 
> -s：表示在1234端口接受GDB的调试链接。
    
##启动调试终端
在另一个超级终端中启动ARM GDB。

	cd linux-4.*
    arm-linux-gnueabi-gdb --tui vmlinux
    	(gdb) target remote localhost:1234
    	(gdb) b start_kernel
    	(gdb) c
GDB开始接管ARM-Linux 内核运行，并且到断点中暂停，这时可使用GDB命令来调试内核。