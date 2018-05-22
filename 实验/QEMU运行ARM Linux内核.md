#QEMU运行ARM Linux内核
##1. 准备工具
###1.1下载代码包：
>**linux-4.0内核：https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.0.tar.gz**
>
>**busybox工具包：https://busybox.net/downloads/busybox-1.24.0.tar.bz2** 

###1.2配置交叉编译工具
>**wget https://releases.linaro.org/components/toolchain/binaries/4.9-2017.01/arm-linux-gnueabi/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabi.tar.xz .**
>
>**export PATH=$PATH:/home/wgs/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabi/bin**


##2. 编译最小的文件系统
###2.1 配置busybox的menuconfig，配置成静态编译
	make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm menuconfig
![](https://i.imgur.com/BHl79mj.jpg)
###2.2 编译busybox
	make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm install 
###2.3 将\_install ，目录复制到linux-4.x 目录下，进入\_install目录，创建目录：
	mkdir etc dev mnt 
	mkdir -p etc/init.d/

###2.4 在_install/etc/init.d/目录下创建rcS文件(启动脚本)
	mkdir -p /proc
	mkdir -p /tmp
	mkdir -p /sys
	mkdir -p /mnt
	/bin/mount -a   //自动挂载文件系统
	mkdir -p /dev/pts
	mount -t devpts devpts /dev/pts
	echo /sbin/mdev /proc/sys/kernel/hotplug
	mdev -s

**增加可执行权限 chmod +x rcS**
###2.5 在_install/etc 目录创建fstab文件
 /etc/fstab是用来存放文件系统的静态信息的文件。当系统启动的时候，系统会自动地从这个文件读取信息，并且会自动将此文件中指定的文件系统挂载到指定的目录。

	proc /proc proc defaults 0 0
	tmpfs /tmp tmpfs defaults 0 0
	sysfs /sys sysfs defaults 0 0
	tmpfs /dev tmpfs defaults 0 0
	debugfs /sys/kernel/debug debugfs defaults 0 0

###2.6 在_install/etc 目录创建inittab文件
当内核初始化后，就会启动第一个进程 init，init进程会进行一系列的系统初始化工作，init是根据什么来进行初始化的？

init 会读取/etc/inittab文件，执行里面的内容来进行初始化工作，这个文件是一定的格式。

	::sysinit:/etc/init.d/rcS
	::respawn:-/bin/sh
	::askfirst:-/bin/sh
	::ctrlaltdel:/bin/umount -a -r
###2.7 在_install/dev 目录创建设备节点，需要root权限
在Linux中，所有设备都以文件的形式存放在/dev目录下，都是通过文件的方式进行访问，设备节点是Linux内核对设备的抽象，一个设备节点就是一个文件。

	sudo mknod console c 5 1
	sudo mknod null c 1 3
##3. 编译内核
	make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm  vexpress_defconfig
	make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm  menuconfig
**配置initramfs， 在initramfs source file中填入\_install,并把Default kernel command string 清空。**
![](https://i.imgur.com/DaTyvhm.png)
**配置memory split 为“3G/1G user/kernle split”, 并打开高端内存**![](https://i.imgur.com/zWM9pCL.jpg)

	make bzImage -j4 CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm 
	make dtbs CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm 

##4. 运行QEMU来模拟4核Cortex-A9的Versatile Express开发平台
>**qemu-system-arm -M vexpress-a9 -smp 4 -m 1024M -kernel  arch/arm/boot/zImage -append "rdinit=/linuxrc console=ttyAMA0 loglevel=8" -dtb arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic**

