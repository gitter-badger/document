### C++ const 限定符小结 ###
--------------------------
在Google C++编程规范中有一条Tips：

> 我们强烈建议你在任何可能的情况下都要使用 const.

但是在C++庞杂的语法点中，const拥有着相当复杂的变化，以至于一些C++程序员在使用了好几年C++中都不能完全了解这些变化。下面就C++ Primer中提及的const知识点做一个小小的总结。


### 1. const 限定符简介 ###
我们希望定义这样一种变量：**它的值不能被改变**。为了满足这一要求，可以用关键字`const`对变量类型加以限定。

> **---> const对象必须初始化**

因为const对象一旦创建后其值就不能再改变，所以const对象必须初始化。

	const int i = get_size();	 // correct
	const int j = 42;			 // correct
	const int k;				 // error
	
> **---> 利用一个对象去初始化另外一个对象，则它们是不是const都无关紧要**

与非const类型所能参与的操作相比，const类型的对象能完成其中大部分，但也不是所有的操作都适合，**主要的限制就是只能在const类型的对象上执行不改变其内容的操作**。在不改变const对象的操作中还有一种是**初始化**：

	int i = 42;
	const int ci = i;			// correct
	int j = ci;					// correct
	
> **---> 默认状态下，const对象仅在文件内有效**

默认情况下，const对象被设定为仅在文件内有效。当多个文件中出现了同名的const变量时，**其实等同于在不同文件中分别设定了独立的变量**。比如以下这个例子：

	// main.cpp
	......
	const int ci = 123;
	
	int main()
	{
		foo();
		cout << "[main.cpp] ci = " << ci << endl;
		return 0;
	}
	......

而在另外一个foo.c中则定义了一个同名const变量：
	
	// foo.cpp
	......
	const int ci = 456;
	void foo()
	{
		cout << "[foo.cpp] ci = " << ci << endl;
	}
	......
然后编译看看这个test case你就会发现ci的输出值是不同的且编译期间不会报错。

**但是**，某些时候，我们确实有必要在不同文件中共享同一个const变量，该怎么办呢？

**解决办法是：**不管声明还是定义都添加`extern`关键字。

	// file_1.cc定义和初始化了一个常量，该常量能被其他文件访问
	extern const int bufSize = fcn();
	
	// file_1.h
	extern const int bufSize; // 与file_1.cc中定义的bufSize是同一个
