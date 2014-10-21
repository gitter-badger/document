### 关于C++ RVO的几个实验 ###
---------------------------

今天在重新温习C++拷贝控制的过程的时候，发现根据书中（《C++ Primer 5th》）关于拷贝控制的说法所写的测试代码，其运行过程竟然跟理论是一定差距的。后面Google了一下StackOverflow和Wikipedia，发现了`RVO`这个编译优化手段，从而进一步了解了这个不为人知的技术细节。

首先先解释一下什么是**`RVO`**。根据**wikipedia**的解释，`RVO`全称就是`Return Value Optimization`：

> Return value optimization, or simply RVO, is a compiler optimization technique that involves eliminating the temporary object created to hold a function's return value. In C++, it is particularly notable for being allowed to change the observable behaviour of the resulting program.

简而言之，`RVO`就是用来一种用来消除为存放函数返回值所创建的临时对象的编译器优化技术。

让我们先用一小段代码来直观了解一下`RVO`（**我使用的是g++ 4.7.3**）:

```c++
	#include <iostream>

	using namespace std;

	class Foo {
	public:
    	Foo() { cout << "Foo()" << endl; }
    	Foo(const Foo& f) { cout << "Foo(const Foo& f)" << endl; }
    	~Foo() { }
	};

	Foo bar()           // return value copy will call copy constructor in theory
	{
    	cout << "bar()" << endl;
    	return Foo();   // it will call Foo default constructor
	}

	int main()
	{
    	Foo f = bar();
    	return 0;
	}
```
这么一小段代码，根据教科书的说法，我们**理论上**应该可以观察到：

`bar()`内部会call一次Foo的默认构造函数

`bar()`内部函数返回值会call一次拷贝构造函数

创建`f`的时候会以`bar()`的返回值为复制对象call一次拷贝构造函数

实际代码运行结果是：

	nick@GGUbuntu:c++11_move$ ./test_RVO.o 
	bar()
	Foo()

**没有拷贝构造函数的调用**！这就是`RVO`的作用。通过函数返回值来直接创建对象。

如果我们不想使用`RVO`，可以在g++编译的时候加入`-fno-elide-construtors`。这时的运行结果是：

	nick@GGUbuntu:c++11_move$ ./a.out 
	bar()
	Foo()
	Foo(const Foo& f)
	Foo(const Foo& f)
跟我们理论设想是一致的。
