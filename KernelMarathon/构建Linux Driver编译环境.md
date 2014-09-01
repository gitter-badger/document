#### 如何构建 Linux driver 编译环境 ####
-------------------------------------
想在原有的linux系统中添加自己的driver模块(`*.ko`)，就必须先构建build ko文件的编译环境。尽管ko文件只是linux的一个模块，但是它在编译过程中也会用到linux kernel的一些宏定义和头文件。所以，我们必须在编译环境中构建一个linux kernel的源码树。可参考以下两种方式：

1. download linux kernel；

	我下载了一包`linux-3.9`的源代码，编译一遍，这就是一包源码树了。
	
2. using `usr/src/*`；

	如果你是使用`ubuntu`操作系统，可以直接使用`/usr/src/$(uname -r)`。或者使用`/lib/modules/$(shell uname -r)/build`，其实`build`就是我上面说的那个目录的符号链接。
	
通过以上两种方式构建好本地源码树后，`Makefile`可以参考如下写法：

	obj-m := hello_driver.o
	
	KERNELDIR := /lib/modules/$(shell uname -r)/build
	#KERNELDIR := /home/nick/Hacking/kernel/linux-3.9

	PWD := $(shell pwd)

	all: 
		make -C $(KERNELDIR) M=$(PWD) modules
