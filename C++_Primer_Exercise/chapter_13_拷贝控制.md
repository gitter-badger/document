### chapter 13 拷贝控制 ###
--------------------------
1. 拷贝构造函数是什么？什么时候使用它？
	定义：如果一个构造函数的第一个参数是自身类型的引用，且任何额外的参数都有默认值，则此构造函数是拷贝构造函数。
	
	当我们使用拷贝初始化的时候，将会使用拷贝构造函数。比如：
	
		string s = dots; // dots is a string object
	简单来说，当我们`=`号去初始化一个类对象的时候，将会发生拷贝构造函数的调用。
	
	在以下几种情况下也会发生拷贝构造函数的调用：
	- **将一个对象作为实参传递给一个非引用类型的形参**
	- **从一个返回类型为非引用类型的函数返回一个对象**
	- 用花括号列表初始化一个数组的元素或一个聚合类中的成员（这个我还没仔细琢磨过）
2. 解释为什么下面的声明是非法的：
	
		Sales_data::Sales_data(Sales_data rhs);
	拷贝控制函数的参数**必须是指向自身的const引用**，也就是说，如下声明才是合法的：
		
		Sales_data::Sales_data(const Sales_data& rhs);
	这里要解释两点：		
	a. **为什么其参数要使用指向自身的引用？**
	
	   试想一下，如果我们有这么一个函数：
	
			void foo(Sales_data item);		
	当我们向这个函数传递参数的参数的时候，实际就发生拷贝构造的过程，见**习题1**所答。为了调用拷贝控制函数，我们必须拷贝它的参数，但为了拷贝实参，我们又必须调用拷贝构造函数，如此无限循环。调用就无法成功。
	b. **为什么需要添加const修饰？**
	   
	   还是刚刚这个函数。如果我们这么去调用它：
	   		
	   		foo(Sales_data());
	   那么这个通过`Sales_data`默认构造函数将会传递给一个不具名的tmp对象（**这是const对象！**），也就是说编译器会发生如下情况：
	   		
	   		const Sales_data tmp(Sales_data());
	   		Sales_data item = tmp;  // 这是调用copy-construtor
	   	为了能成功调用拷贝构造函数，在这种情况下，我们必须使用指向自身类型的const引用。
3. 当我们拷贝一个`StrBlob`时，会发生什么？拷贝一个`StrBlobPtr`呢？

	由于我们没有对这两个类定义拷贝构造函数，编译器会自动帮我们生成合成的拷贝构造函数。
	一般情况下，合成的拷贝构造函数会将其参数的成员逐个拷贝到正在创建的对象中。编译器从给定对象中依次将每个非static成员拷贝到正在创建的对象中。
	
	对于类类型，会使用其拷贝构造函数来拷贝；
	
	对于内置对象，直接拷贝。
	
	`StrBlob`只有一个`shared_ptr`类型的数据成员，`shared_ptr`是一个类类型，所以会调用其拷贝构造函数进行初始化；
	
	`StrBlobPtr`有两个数据成员，`weak_ptr`类型和`size_t`类型的数据成员。其中一个是类类型，一个是内置对象，所以同上所述。

4. 假定`Point`是一个类类型，它有一个public的拷贝构造函数，指出下面程序片段中哪些地方使用了拷贝构造函数：
	
		1. Point global;
		2. Point foo_bar(Point arg)
		3. {
		4. 	 Point local = arg, *heap = new Point(global);
		5.	 *heap = local;
		6.	 Point pa[4] = { local, *heap };
		7.	 return *heap;
		8. }
		
	这个程序片段已经包含了**习题1**所论述的内容。
	
	a. `L2`非引用参数的拷贝；
	b. `L4`对local对象进行了拷贝初始化；
	c. `L5`其实是拷贝赋值
	d. `L6`采用花括号初始化进行拷贝初始化；
	e. `L7`函数返回非引用参数，需要调用拷贝构造函数；
5. 给定下面的类框架，编写一个拷贝构造函数，拷贝所有成员。你的构造函数应该动态分配一个新的`string`，并将对象拷贝到`ps`指向的位置，而不是`ps`本身的位置。

		class HasPtr {
		public:
		    HasPtr(const std::string &s = std::string()) :
		        ps(new std::string()), i(0) { }
		private:
		    std::string *ps;
		    int i;
		};
		
	解决这个问题的关键要意识到：`ps`是一个指针对象。如果我们采用合成构造函数，只是拷贝指针的值而不重新分配动态内存，那么当原始对象被析构之后，这个采用拷贝初始化的对象将指向一个被释放掉的内存区域，引用它将导致未定义行为（Segmentation fault on Linux）。
	
	代码如下所示（增加了`HasPtr`的析构函数和一个`print`调试函数）：
	
		class HasPtr {
		public:
		    HasPtr(const std::string &s = std::string()) :
		        ps(new std::string()), i(0) { }
		    
		    HasPtr(const HasPtr& rhs) : ps(new std::string(*(rhs.ps))), i(rhs.i) { }
		    
		    void print() const 
		    {
		    	cout << *ps << endl;
		    	cout << i << endl;
		    }
		    
		private:
		    std::string *ps;
		    int i;
		};
		
		
