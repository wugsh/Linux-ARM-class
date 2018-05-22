#GDB调试
##1. 安装GDB
    sudo apt-get install gdb
    
##2. 使用GDB
###2.1 开始调试，命令：r、n、p
第一个演示代码heoo.c

  	#include <stdio.h>
	int g_var = 0;

	static int _add(int a, int b) {
    	printf("_add callad, a:%d, b:%d\n", a, b);
    	return a+b;
	}

	int main(void) {
	    int n = 1;
	    
	    printf("one n=%d, g_var=%d\n", n, g_var);
	    ++n;
	    --n;
	    
	    g_var += 20;
	    g_var -= 10;
	    n = _add(1, g_var);
	    printf("two n=%d, g_var=%d\n", n, g_var);
	    
	    return 0;
	}

> 第一个命令 gdb heoo.out 表示 gdb加载 heoo.out 开始调试. 如果需要使用gdb调试的话编译的时候 gcc 需要加上 -g命令.
> 
> 其中 l 命令表示 查看加载源码内容. 下面将演示如何加断点.

![](https://i.imgur.com/MU8nzOy.png)

![](https://i.imgur.com/41nepqf.png)

> r 表示调试的程序开始运行.

![](https://i.imgur.com/BMBZDiK.png)


> p 命令表示 打印值. n表示过程调试, 到下一步. 不管子过程如何都不进入. 直接一次跳过.

![](https://i.imgur.com/ZcytiHt.png)

> 上面用的s 表示单步调试, 遇到子函数,会进入函数内部调试.

**总结一下 . l 查看源码 , b 加断点, r 开始运行调试, n 下一步, s下一步但是会进入子函数. p 输出数据.**

##2.2 gdb 其它常用命令用法 c q b info
首先看 用到的调试文件 houge.c

    #include <stdio.h>
    #include <stdlib.h>
    #include <time.h>
    
    /*
     * arr 只能是数组
     * 返回当前数组长度
     */
    #define LEN(arr) (sizeof(arr)/sizeof(*arr))
    
    // 简单数组打印函数
    static void _parrs(int a[], int len) {
    	int i = -1;
    	puts("当前数组内容值如下:");
    
    	while(++i < len) 
    		printf("%d ", a[i]);
    	putchar('\n');
    }
    
    // 简单包装宏, arr必须是数组
    #define PARRS(arr) \
    	_parrs(arr, LEN(arr))
    
    #define _INT_OLD (23)
    
    /*
     * 主函数,简单测试
     * 测试 core文件, 
     * 测试 宏调试
     * 测试 堆栈内存信息
     */
    int main(void) {
    	int i;
    	int a[_INT_OLD];
    	int* ptr = NULL;
    
    	// 来个随机数填充值吧
    	srand((unsigned)time(NULL));
   	 	for(i=0; i<LEN(a); ++i)
    		a[i] = rand()%222;
    
    	PARRS(a);
    
    //全员加double, 包含一个错误方便测试
    	for(i=1; i<=LEN(a); ++i)
    		a[i] <<= 1;
    	PARRS(a);
    
    // 为了错,强制错
    	*ptr = 0;
    
    	return 0;
	}
    
![](https://i.imgur.com/IvxRxIA.png)

> 这个图是前言的补充, c跳过直到下一个断点处, q表示程序退出.
> 
> 打开记录core功能，`ulimit -c 1024`
> 
> 在 houge.c 中我们开始调试. 一运行段错误, 出现了我们的 core.pid 文件.

![](https://i.imgur.com/YNeE2AG.png)

> 通过 gdb houge.out core.27047 开始调试. 马上定位出来了错误原因.

##2.3 调试 内存堆栈信息