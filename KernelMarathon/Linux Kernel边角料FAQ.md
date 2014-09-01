### Linux Kernel边角料FAQ ###
----------------------------

1.什么是`asmlinkage`？

在`2.6.22`的代码中：

	/* code path: linux-2.6.22/include/asm-i386/linkage.h */
	#define asmlinkage CPP_ASMLINKAGE __attribute__((regparm(0)))

其中`CPP_ASMLINKAGE`在以下文件中定义：

	/* code path: linux-2.6.22/include/linux/linkage.h */
	#ifdef __cplusplus
	#define CPP_ASMLINKAGE extern "C"
	#else
	#define CPP_ASMLINKAGE
	#endif

可见，`regparm`又是GCC的attribute用法，如下：

>regparm (number)
>
>On the Intel 386, the regparm attribute causes the compiler to pass arguments number one to number if they are of integral type in registers EAX, EDX, and ECX instead of on the stack. Functions that take a variable number of arguments continue to be passed all of their arguments on the stack.
>Beware that on some ELF systems this attribute is unsuitable for global functions in shared libraries with lazy binding (which is the default). Lazy binding sends the first call via resolving code in the loader, which might assume EAX, EDX and ECX can be clobbered, as per the standard calling conventions. Solaris 8 is affected by this. Systems with the GNU C Library version 2.1 or higher and FreeBSD are believed to be safe since the loaders there save EAX, EDX and ECX. (Lazy binding can be disabled with the linker or the loader if desired, to avoid the problem.) 

说明所有参数通过stack来传递。

2.什么是`Statements and Declarations in Expressions`？

这又是一个GNU C Extension，将表达式做了扩展。这种表达式看起来很像宏函数，但是可以在内部定义局部变量，使用循环或switch等语句，使之更为强大。其形式为：

		({ int y = foo (); int z;
        if (y > 0) z = y;
        else z = - y;
        z; })

必须用括号包起来，然后内部用花括号闭合起来，里面可以使用语句，最后一个表达式的值作为整个表达式的值。如果最后一个语句是个声明，则整个表达式为`void`。

下面举个简单的例子来说明：

	#include <stdio.h>
	#define max(a, b) \
	    ({ typeof(a) _a = a, _b = b; _a > _b ? _a : _b; })
	#define _max(a, b) ((a) > (b) ? (a) : (b))
	#define ret(x) \
	    ({ int a = 1; x; })
	int main(void)
	{
	    int i = 4, j = 4;
	    printf("%d\n", max(max(++i,j), 3));		// 输出 5
	    printf("%d\n", _max(_max(++i,j), 3));   // 输出 9
	    printf("%d\n", ret(i));
	    return 0;
	}

可以看到，用这种扩展形式写出来的`max`可以避免一些side effect。由上可以看出，以前的宏函数由于有side effect，最后输出会是9。而扩展形式的`max`则能输出5。`ret(x)`可以看到其表达式的值以最后一个表达式的值为准。

3.什么是`typeof`？

可以看如下示例：
	
	int a;
	typeof(a) c;
	
这时候c就是int类型。`typeof`其实就是可以把当前变量的类型取出来。

4.什么是`likely`和`unlikely`？

这两个宏其实是利用GCC 的built-in函数来实现指令优化。

	
	/* linux-2.6.22/include/linux/compiler.h */

	/*
	 * Generic compiler-dependent macros required for kernel
	 * build go below this comment. Actual compiler/compiler version
	 * specific implementations come from the above header files
	 */
	
	#define likely(x)	__builtin_expect(!!(x), 1)
	#define unlikely(x)	__builtin_expect(!!(x), 0)

`__builtin_expect`是GCC的一个内置函数，其作用是：

> Built-in Function: long __builtin_expect (long exp, long c)
> 
> expect exp == c

所以`likely(x)`的意思就是：x的值有很大可能是1，`unlikely(x)`就是说x很大可能是0。还有一个疑问：为什么要在宏里面用`!!(x)`呢？因为`likely(x)`和`unlikely(x)`希望x只是0或1，如果x是非0值，那么取反一次即是0，再取反便是1了。这样便将一个非零值转化为1了。

采用`__builtin_expect`最大的好处是有助于CPU在流水线处理时分支预测，将最有可能被执行到二进制程序提前
载入cache，提高cache的命中率。

5.什么是`container_of`宏？

这个可是个大名鼎鼎的宏。让我们先看一看它的实现：

	/* linux-2.6.22/include/linux/kernel.h */
	
	/**
	 * container_of - cast a member of a structure out to the containing structure
 	 * @ptr:	the pointer to the member.
 	 * @type:	the type of the container struct this is embedded in.
 	 * @member:	the name of the member within the struct.
 	 *
 	 */
	#define container_of(ptr, type, member) ({			\
	const typeof( ((type *)0)->member ) *__mptr = (ptr);	\
	(type *)( (char *)__mptr - offsetof(type,member) );})
	
其中，`offsetof`的定义如下所示：
	
	#undef offsetof
	#ifdef __compiler_offsetof
	#define offsetof(TYPE,MEMBER) __compiler_offsetof(TYPE,MEMBER)
	#else
	#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
	#endif
	#endif /* __KERNEL__ */
	
这个宏的作用便是： **从某个结构体的成员指针来获取该结构体的起始地址，从而来访问其他成员**。它到底是怎么工作的呢？

从形式上来看，它是`2`中说过的GCC中扩展的表达式。这个宏最巧妙的地方就是**使用了0地址来获取offset而无需访问该地址**。如何理解？
可以看以下的测试代码：

	#include <stdio.h>
	
	struct task_list {
		unsigned long pid;
		unsigned long nice;
		unsigned short time;
		char *name;
	};
	
	int main(void)
	{
		printf("task_list's pid offset:  %d\n", (size_t)&((struct task_list *)0)->pid);
		printf("task_list's nice offset: %d\n", (size_t)&((struct task_list *)0)->nice);
		printf("task_list's time offset: %d\n", (size_t)&((struct task_list *)0)->time);
		printf("task_list's name offset: %d\n", (size_t)&((struct task_list *)0)->name);
		return 0;
	}

运行结果是：

	task_list's pid offset:  0
	task_list's nice offset: 4
	task_list's time offset: 8
	task_list's name offset: 12

通过这种方式我们就轻易获取到了每个成员变量在结构体中的offset。

我们再回过头来看`container_of`这个宏。

`step_1--->` 先将某成员变量的指针的值赋给`__mptr`；
`step_2--->` 将`__mptr`减去该成员变量在结构体内的offet，从而获取起始地址；

这便是`container_of`的工作原理，其中`offset_of`才是其中最其巧妙之处。


6.什么是`ALIGN`宏？

这个宏的定义如下所示：

	/* linux-2.6.22/include/linux/kernel.h */
	
	#define ALIGN(x,a)		__ALIGN_MASK(x,(typeof(x))(a)-1)
	#define __ALIGN_MASK(x,mask)	(((x)+(mask))&~(mask))
	
先不说其实现细节，先看看怎么去用。看以下的测试代码：

	#include <stdio.h>   

	#define ALIGN(x,a)		__ALIGN_MASK(x,(typeof(x))(a)-1)
	#define __ALIGN_MASK(x,mask)	(((x)+(mask))&~(mask))

	int main(void)
	{
    	printf("origin_value = %d, aligned_value = %d\n", 111, ALIGN(111, 4));   // 2^2 = 4
    	printf("origin_value = %d, aligned_value = %d\n", 123, ALIGN(123, 8));   // 2^3 = 8
    	printf("origin_value = %d, aligned_value = %d\n", 134, ALIGN(134, 16));  // 2^4 = 16
    	printf("origin_value = %d, aligned_value = %d\n", 123, ALIGN(123, 13));  // 13 is not power of 2
    	return 0; 
	}

运行结果：

	origin_value = 111, aligned_value = 112
	origin_value = 123, aligned_value = 128
	origin_value = 134, aligned_value = 144
	origin_value = 123, aligned_value = 131
可见，`ALIGN(x,a)`中，`a`必须是2的某次幂，这样，该宏就可以把`x`对齐到该次幂上。

7.关于`ERR_PTR`，`PTR_ERR`和`IS_ERR`的使用

这几个宏的设置是为了简化错误处理对于返回值的处理。例如，kernel对于每项操作的错误都会设定一个错误码，关于错误码的设定，`2.6.35`中的文件包含关系是：

	//---> include/linux/errno.h
	#ifndef _LINUX_ERRNO_H
	#define _LINUX_ERRNO_H
	
	#include <asm/errno.h>
	#ifdef __KERNEL__		// 这说明以下错误码只能由kernel使用，module不能使用
	......
	#define ERESTARTSYS     512
	......
	#endif
	#endif
	
然后再包含：
	
	//---> include/asm-generic/errno.h
	#ifndef _ASM_GENERIC_ERRNO_H
	#define _ASM_GENERIC_ERRNO_H
	
	#include <asm-generic/errno-bash.h>
	#define EDEADLK         35      /* Resource deadlock would occur */
	.....
	#endif
	
最后再包含：
	
	//---> include/asm-generic/errno-base.h
	// 以下包含了最常用的错误码
	#ifndef _ASM_GENERIC_ERRNO_BASE_H
	#define _ASM_GENERIC_ERRNO_BASE_H
	#define EPERM            1      /* Operation not permitted */
	#define ENOENT           2      /* No such file or directory */
	......
	#endif
	
好，以上就是kernel中对于错误码的设定。现在让我们来设想一种场景，当我们进入mount系统调用的时候，会调用到这么一个函数，其api如下所示：

	struct vfsmount *
	vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data);
	
不难看到，其返回值是某种数据结构的指针。假如，在这个函数调用的处理流程发生了错误而结束了整个执行流程，我们该返回什么呢？这时候便需要使用到`ERR_PTR`，这个宏的作用就是将某种error code转换成pointer，还是拿这个api来举例，假如传入的`type`是空，我们需要返回`ENODEV`的指针转换值：

	struct vfsmount *
	vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
	{
		......		
		if (!type) 	// type为空
			return ERR_PTR(-ENODEV); // 返回ENODEV的指针转换值，注意是负值
		......
	}
	
当我们想知道某个api返回值是不是一个错误码，也很简单，使用`IS_ERR`就可以。如果该指针是由某一个错误码转换而来，它就返回1，反之返回0。如果你想将指针转换成错误码，那就是使用`PTR_ERR`，注意，转换回来后也是负值。在kernel中，返回的错误码都是`-Exxxx`的一个负数。

kernel规定错误码最多是4095个，见如下代码：
	
	//---> include/linux/err.h
	......
	#define MAX_ERRNO	4095 	// 注意一个细节，(unsigned long)-4095 = 0xFFFF-F001
								// 那么每个错误码都必须大于-MAX_ERRNO
	#define IS_ERR_VALUE(x)  	unlikely((x) >= (unsigned long)-MAX_ERRNO)
								// 为什么错误码需要使用负数，这是因为使用负数，其错误指针是落在kernel地址空间
	......
	static inline void *__must_check ERR_PTR(long error)
	{
		return (void *)error;	// 只是简单地做一次强转
	}
	
	static inline long __must_check PTR_ERR(const void *ptr)
	{
		return (long)ptr;
	}

	static inline long __must_check IS_ERR(const *ptr)
	{
		return IS_ERR_VALUE((unsigned long)ptr);
	}
	
	......

8.Linux Kernel Procee的状态

	
- `TASK_RUNNING`：意味着进程处于可运行状态。这并不意味着已经实际分配了CPU。进程可能会一直等到调度器选中它。改状态确保进程可以立即运行而无需等待外部事件。

- `TASK_INTERRUPTIBLE`：是针对等待某事件或其他资源的睡眠进程设置的。在内核发送信号给该进程表明事件已经发生时，进程状态变为`TASK_RUNNIG`，它只要调度器选中该进程即可恢复执行。

- `TASK_UNINTERRUPTIBLE`：用于因内核指示而停用的睡眠进程。它们不能由外部信号唤醒，只能由内核亲自唤醒。

- `TASK_STOPPED`：表示进程特意停止运行，例如，由调试器暂停。

- `TASK_TRACED`： 表示不是进程状态，用于从停止的进程中，将当前被调试的那些(使用ptrace机制)与常规的进程区分开。
