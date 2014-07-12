### C++ const 限定符小结 ###
--------------------------
### 4. 顶层 const 和 底层 const ###

顶层const（top-level const）和底层const（low-level const）这两个概念估计是C++ Primer首创的。我翻阅过其他C++书籍，几乎都没有类似的说法。不过，C++ Primer的作者是C++标准和编译器的开发者，自然会采用另外一种角度（编译器开发者而非语言使用者）去看待这个语法现象。

> **---> 顶层 const 表示对象本身是个常量，底层 const 表示所指对象是个常量**

就指针本身来说，顶层const表示指针本身是个常量，底层const表示指针所指的对象是个常量。

	int i = 0;
	int *const p1 = &i;        // 不能改变p1的值，对于p1对象来说，这是一个顶层const
	const int ci = 42;         // 不能改变ci的值，对于ci对象来说，这是一个顶层const
	const int *p2 = &ci;       // 允许改变p2，但是不能改变p2所指向的值，这是一个顶层const
	const int *const p3 = p2;  // 靠右是顶层const，靠左是底层const
	const int &r = ci;         // 不能改变r绑定的值，这是一个底层const
	
	
> **--> 对于拷入拷出操作，顶层const不受什么影响**

	int i = 0;
	const int ci = 42;
	i = ci;                     // 拷贝ci的值，ci是一个顶层const，对此操作无影响
	const int *p2 = &ci;
	const int *const p3 = p2;   // p2和p3指向的对象类型相同，p3顶层const的部分不影响
	
> **---> 对于拷入拷出操作，底层const对于拷入拷出对象必须具有相同的底层const资格，或者两个对象的数据类型必须能够转换**

	int *p = p3;                // error，p3包含底层const的定义，而p没有
	p2 = p3;                    // correct，p2和p3都是底层const
	p2 = &i;                    // correct，int*能转换成const int*
	int &r = ci;                // error，二者不具有相同资格的底层const
	const int &r2 = i;          // correct，可以转换



	
