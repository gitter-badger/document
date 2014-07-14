### 根据const参数进行函数重载 ###
------------------------------

顶层const不影响传入函数的对象。**一个拥有顶层const的形参无法和另一个没有顶层const的形参区分开来**。

	Record lookup(Phone);
	Record lookup(const Phone);     // 重复声明了Record lookup(Phone)
	
	Record lookup(Phone *);
	Record lookup(Phone* const);    // 重复声明了Record lookup(Phone *)
	
显然，上面的形参都是用了顶层const，所以不能进行重载。

另一方面，如果形参是**顶层const**（**也就是某种类型的指针或引用**），那么则可以通过区分**其指向的是常量对象还是非常量对象**来实现函数重载。

	Record lookup(Account&)；
	Record lookup(const Account&);  // 新函数，作用于常量引用 
	
	Record lookup(Account*);
	Record lookup(const Account*);  // 新函数，作用于指向常量的指针
	
编译器可以通过实参**是否是常量来推断应该调用哪个函数**。因为const不能转换成其他类型（当然，强转党另说），所以我们只能把const对象或指向const的指针传递给const形参。相反地，因为非常量可以转换成const，所以上面的4个函数都能作用于非常量对象或者指向非常量对象的指针。

**当我们传递一个非常量或者指向非常量对象的指针时，编译器会优先选用非常量版本的函数**。

看看一下测试代码：

	......
	void foo(int *cp)
	{
		cout << "foo(int *cp)" << endl;
	}
	
	void foo(const int *cp)
	{
		cout << "foo(const int *cp)" << endl;
	}
	
	int main()
	{
		int i = 123;
		const int j = 456;
		foo(&i);                  // 调用foo(int *cp)，如上所说，优先调用非常量版本
		foo(&j);                  // 调用foo(const int *cp) 
		foo((const int *)&i);     // 调用foo(const int *cp)，采用了强转
	}
	......
	

同样道理，**通过区分成员函数是否是const，我们可以对其进行重载**。如下所示：

	......
	class Test
	{
	public:
		void foo() const
		{
			cout << "foo() const" << endl;
		}
		void foo()
		{
			cout << "foo()" << endl;
		}
	};
	
	int main()
	{
		Test test_no_const;
		const Test test_const;
		test_no_const.foo();       // 调用foo()的非const版本
		test_const.foo();          // 调用foo()的const版本
		return 0;
	}
	......
