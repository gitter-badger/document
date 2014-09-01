### C++ Primer chpater 7小结 ###
--------------------------------
C++ Primer第五版的第七章是对类这个概念一个轮廓式的介绍，里面有一些重要且稍显凌乱的知识点，这里总结如下。

1. 默认构造参数

	默认构造函数（default constructor）就是在没有显式提供初始化式时调用的构造函数。它由不带参数的构造函数，**或者为所有的形参提供默认实参的构造函数定义**。如果定义某个类的变量时没有提供初始化式就会使用默认构造函数。
	如下所示：
	
		class Test {
		public:
			Test() { }
			Test(int _a = 1) : a(_a) { }
			int a;
		};
		
		......
		Test a;
		
	这样编译器是会报错的。因为我们定义了两个默认构造函数。
	
	**如果一个构造函数为所有参数都提供了默认实参，则它实际上也定义了默认构造函数**。

2. 如果类中包含一个其他类类型的成员且这个成员的类型没有默认构造函数，那么编译器将无法进行初始化。对于这样的类来说，我们必须进行初始化。

	**如果成员是const，引用，或者属于某种未提供默认构造函数的类类型，我们必须通过构造函数初始值列表为这些成员提供初始值**。这里要理解的是：赋值和初始化是不同的！


3. **类可以允许其他类或者函数访问它的非公有成员，方法是令其他类或者函数成为它的友元**。

	友元声明只能出现在类定义的内部，但是在类内出现的具体位置不限。**友元不是类的成员也不受它所在区域访问控制级别的约束**。在Google C++编程规范中，是这么建议程序员使用友元的：

	>**我们允许合理使用地使用友元类和友元函数**。
	
	>通常友元应该定义在同一文件内, 避免代码读者跑到其它文件查找使用该私有成员的类。
	
	>经常用到友元的一个地方是将`FooBuilder`声明为`Foo`的友元, 以便`FooBuilder`正确构造`Foo`的内部状态, 而无 需将该状态暴露出来。某些情况下，将一个单元测试类声明成待测类的友元会很方便。友元扩大了 (但没有打破) 类的封装边界. 某些情况下，相对于将类成员声明为public，使用友元是更好的选择,尤其是如果你只允许另一个类访问该类的私有成员时。当然，大多数类都只应该通过其提供的公有成员进行互操作。

4. 除了定义数据和函数成员外，类还可以自定义某种类型在类中的别名。**由类定义的类型名字和其他成员一样存在访问限制，可以是public或者private中的一种**。

		class Screen {
		public:
			typedef std::string::size_type pos;
		private:
			pos cursor = 0;
			pos height = 0, width = 0;
			std::string contents;
		};
	这样就能更好地隐藏代码的实现细节。在C++11中，还可以这样使用：
	
		class Screen {
		public:
			......
			using pos = std::string::size_type;
			......
		};
5. 可变数据成员（mutable）

	有时（但并不频繁）会发生这样一种情况，我们希望能修改类的某个数据成员，即使是在一个const成员函数内。可以通过在变量的声明中加入`mutable`关键字做到这一点。
	
	**一个可变数据成员（mutable data member）永远不会是const，即使它是const对象的成员**。
	
		class Screen {
		public:
			void some_member() const;
		private:
			mutable size_t access_ctr;   // 即使在一个const对象内也能被修改
		};
		
		void Screen::some_member() const
		{
			++access_ctr;                // 保存一个计数值，用于记录成员函数被调用的次数
		}
		
6. 隐式的类类型转换（说实话，我真不明白C++为什么要有这么古怪的语法点）

	**如果构造函数只接受一个实参，则它实际上定义了转换为此类型的隐式转换机制**。有时我们把这种构造函数称作**转换构造函数（converting constructor）**。
	
	比如：
	
		class Sales_data {
		public:
		......
			Sales_data &combine(const Sales_data&);
			Sales_data (std::istream& );
		......
		private:
			std::string bookNo;
		......
		};
		
	当我们这么使用代码的时候：
		
		string null_book = "9-999-99999-9";
		item.combine(null_book);
		item.combine(cin);
		
	在这段代码中，首先是创建一个`Sales_data`的临时对象，然后用`null_book`去初始化，由于**combine的参数是一个常量引用（这是必须的）**，所以把这个临时对象传递给该常量引用。
	
	可以再看一段代码：
	
		......
		class Test {
		public:
			Test() { cout << "Test()" << endl; }
			Test(string str) : mStr(str) { cout << "Test(string str)" << endl; }
			void print(const Test& test)    // 你可以去掉const然后编译试试看
			{
				cout << test.mStr << endl;
			}
			string mStr;
		};
		
		int main()
		{
			string id("01234");
			Test test;
			test.print(id);   // 发生隐式转换
			return 0;
		}
		
	**在要求隐式转换的上下文中，我们可以通过将构造函数声明为`explicit`加以阻止**。
	
		......
		class Sales_data {
		public:
		......
			explicit Sales_data(const std::string &s) : bookNo(s) { }
			explicit Sales_data (std::istream& );
		......
		};
	
	这样，之前的那种行为就是非法的。
		
	**关键字`explicit`只对一个实参的构造函数有效，需要多个实参的构造函数不能用于隐式转换，所以也无需使用该关键字**。
	
	**只能在类内声明构造函数时使用`explicit`关键字，在类外部定义时不应重复**。
	
	**尽管编译器不会将`explicit`的构造函数用于隐式转换过程，但是我们可以通过强制转换来使用这样的构造函数**。
	
		item.combine(Sales_data(null_book));
		item.combine(static_cast<Sales_data>)

7. **有的时候类需要它的一些成员与类本身直接相关，而不是与类的各个对象保持关联**。我们通过在成员的声明之前加上`static`使得与其类关联在一起。
		
		class Account {
		public:
			void calculate() { amount += amount * interestRate; }
			static double rate() { return interestRate; }
			static void rate(double);
		private:
			std::string owner;
			double amount;
			static double interestRate;
			static double initRate();
		};
		
	**类的静态成员存在于任何对象之外，对象之中不包含任何与静态数据成员相关的数据**。因此，上面的代码只存在一个`interestRate`对象且被所有Account对象共享。
	
	**静态成员函数也不与任何对象绑定在一起，它们不包含this指针**。作为结果，静态成员不能声明成const（因为没有this指针），而且我们也不能在static函数体内使用this指针。
	
	**当在类的外部定义静态成员时，不能重复static关键字。该关键字只出现在类内部的声明语句中**。
	
	**因为静态数据成员不属于类的任何一个对象，所以它们并不是在创建类的对象时被定义的。我们应该在类的外部定义和初始化每个静态成员，且一个静态数据成员只能定义一次**。
	
	**静态成员和普通成员的另一个区别是我们可以使用静态成员作为默认实参**。

8. 聚合类（aggregate class）

