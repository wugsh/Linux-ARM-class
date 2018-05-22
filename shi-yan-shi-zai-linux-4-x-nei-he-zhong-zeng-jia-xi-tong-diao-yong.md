#在Linux 4.x内核中增加系统调用
##1. 下载内核源码

	wget https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.4.tar.gz .
##2. 增加系统调用号
    在系统调用入口表中增加一项：
    vim arch/x86/entry/syscalls/syscall_64.tbl
![](https://i.imgur.com/00pEe3d.png)
##3. 声明系统调用函数
    vim include/linux/syscalls.h
    在末端加入系统调用函数的声明
![](https://i.imgur.com/E1BfKWp.png)
##4. 实现系统调用函数
    vim kernel/sys.c
    在文件末端加入对应的实现函数：
![](https://i.imgur.com/N1c9jmM.png)

##5. 编译安装内核
    编译安装内核用到以下命令
    make x86_64_defconfig
    make -j8  #-j后面的数字表示多线程编译的个数
    sudo make modules_install
    sudo make install
    sudo reboot
    重启后通过uname -r发现当前系统内核已被更新
##6. 测试
###6.1 编写测试用例：

    #include<unistd.h>  
    int main() {  
    	for(int i = 0; i < 5; i++) {  
    		syscall(326, i);  
    	}  
    	return 0;  
    }  
###6.2 打印结果
![](https://i.imgur.com/7hv8ZKD.png)