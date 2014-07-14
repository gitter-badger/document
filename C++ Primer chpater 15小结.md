### C++ Primer chpater 15小结 ###
--------------------------------
在C++ Primer第五版中，第15章估计是全书最最最重要的一章了。因为在这一章节中，全面介绍了C++ OOP编程的基础与运用方式。所以，这一章不得不花一大块时间来慢慢消化和总结，现将部分知识点罗列一下，基本也是抄作业。

1. 在C++中，基类必须将它的两种成员函数区分开来：

	- 希望派生类进行覆盖的函数，也就是虚函数（virtual function）
	
	- 希望派生类直接继承而不需要改变的函数
	
	**如果基类把一个函数声明成虚函数，则该函数在派生类中隐式地也是虚函数**。
	
2. 派生类可以继承定义在基类中的成员，**但是派生类的成员函数不一定有权访问从基类继承而来的成员**。派生类可以访问公有成员，而不能访问私有成员。这时候，就诞生了一种需求：

	> 基类希望它的派生类有权访问该成员，同时禁止其他用户访问。
	
	这就是`protected`成员。如果不从继承的角度来看，它跟private的作用其实是一样的。如果从继承的角度来看，它就是提供了派生类访问基类成员的一种权限。

3. 如果派生类没有覆盖其基类的某个虚函数，派生类会直接继承其在基类中的版本。在C++11中，允许派生类显式地注明它使用某个成员函数覆盖了它继承的虚函数，即加上`override`，如下：

		class Bulk_quote : public Quote {
		public:
			Bulk_quote() = default;
			......
			double net_price(std::size_t) const override;
			......
		};
		
4.  **派生类必须使用基类的构造函数来初始化它的基类部分**。如下所示：

		Bulk_quote(const std::string& book, double p,
				    std::size_t qty, double disc) : Quote(book, p), min_qty(qty), discount(disc)
				   { }
				   
	**首先初始化基类的部分**，然后按照声明的顺序依次初始化派生类的成员。
	
5.  如果一个基类定义了一个静态成员，**则在整个继承体系中只存在该成员的唯一定义**。不论从基类中派生出来多少个派生类，对于每个静态成员来说都只存在唯一的实例。

6. 有时我们会定义这样一种类，我们不希望其他类继承它，或者不想考虑它是否适合作为一个基类，为了实现这个目的，C++11提供了一种防止继承发生的方法，即在类后跟一个关键字`final`：
	
		class NoDerived final { ...... };        // NoDerived不能作为基类
		class Base { ... };
		
		class Last final : Base { }              // Last 不能作为基类
		
		class Bad  : NoDerived { ...... };       // error
		class Bad2 : Last { ...... };            // error 
		
	我们还能把某个函数指定为`final`，如果我们已经把函数定义成`final`了，则之后任何尝试覆盖该函数的操作都将引发错误。
	
		struct D2 : B {
			void f1(int) const final;    // 不允许后续的其他类覆盖f1(int)
		};
		
		struct D3 : D2 {
			void f2();
			void f1(int) const;          // error !
		};
		
7. 回避虚函数的机制

	在某些情况下，我们希望对虚函数的调用不要进行动态绑定，而是强迫其执行虚函数的某个特定版本。使用域作用符可以实现这一目的：
	
		double undiscount = baseP->Quote::net_price(42);
	该代码强行调用`Quote`的`net_price`函数，而不管baseP实际指向的对象类型到底是什么。
	
	**什么时候我们需要回避虚函数的默认机制呢**？通常是当一个派生类的虚函数调用它覆盖的基类的虚函数版本时。在此情况下，基类的版本通常完成继承层次中所有类型都要做的共同任务，而在派生类中定义的版本需要执行一些与派生类本身密切相关的操作。
	
8.  **纯虚函数和抽象基类**

	含有纯虚函数的类都是抽象基类（abstract base class）。抽象基类负责定义接口，而后续的其他类可以覆盖该接口。**我们不能创建一个抽象基类的对象**。
	
	**派生类必须给出自己的纯虚函数的定义，否则它们仍将是抽象基类**。
	
		class Disc_quote : public Quote {
		public:
			Disc_quote() = default;
			Disc_quote(const std::string& book, double price,
						std::size_t qty, double disc) : 
						Quote(book, price),
						quantity(qty), discount(disc) { }
			double net_price(std::size_t) const = 0;   // pure virtual
		protected:
			std::size_t quantity = 0;
			double discount = 0.0;
		};
		
		class Bulk_quote : public Disc_quote {
		public:
			Bulk_quote() = default;
			Bulk_quote(const std::string& book, double price,
			         std::size_t qty, double disc) :
			         Disc_quote(book, price, qty, disc) { }
			         
			double net_price(std::size_t) const override;
		};

9.  **派生类的成员或友元只能通过派生类对象来访问基类的受保护成员。派生类对于一个基类对象中的受保护成员没有任何访问特权**。


		class Base {
		protected:
			int prot_mem;
		};  
		
		class Sneaky : public Base {
			friend void clobber(Sneaky&);    // 能访问Sneaky::prot_mem
			friend void clobber(Base&);      // 不能访问Base::prot_mem
			int j;
		};
		
		void clobber(Sneaky &s) { s.j = s.prot_mem = 0; }   // correct
		void clobber(Base &b) { b.prot_mem = 0; }           // error
		
	让我们来思考一下为什么要作这种规定。如果通过Sneaky友元函数通过Base对象访问受保护成员是正确的，那么它就可以改变一个Base对象的内容，尽管它不是Base的友元，这样我们只要定义一个形如Sneaky的新类就能非常简单地规避掉protected提供的访问保护了。
			
10.  **公有，私有和受保护继承**
	
	某个类对其继承而来的成员的访问权限受到两个因素的影响：
	
	- 基类中该成员的访问说明符
	
	- 派生类的派生列表中访问说明符
	
	**派生访问说明符对于派生类的成员（或友元）能否访问其直接基类的成员没什么影响。对基类成员的访问权限只与基类中的访问说明符有关**。
	
	**派生访问说明符的目的是控制派生类用户（包括派生类的派生类在内）对于基类成员的访问权限**。
	
			class Base {
			public:
			void pub_mem();
			protected:
				int prot_mem;
			private:
				char priv_mem;
			};
		
			class Pub_Derv : public Base {
				int f()  { return prot_mem; }    // 派生类能访问protected成员
				char g() { return priv_mem; }    // error
			};
		
			class Priv_Derv : private Base {
				// private不影响派生类的访问权限
				int f1() const { return prot_mem; }
			};
			
	如上所示，当我们这样使用代码的时候：
			
			Pub_Derv d1;
			Priv_Derv d2;
			d1.pub_mem();   // correct
			d2.pub_mem();   // error：pub_mem在派生类中是private
		
	也即是，private继承让其基类的成员都变成了private，类似地，protected继承让其基类都变成了protected。
		
11.  虚析构函数

	继承关系对基类拷贝构造最直接的影响是基类通常应该定义一个**虚析构函数**，这样我们就能动态分配继承体系中的对象了。
	
	当我们delete一个动态分配的对象的指针时将执行析构函数。如果该指针指向继承体系中的某个类型，则又可能出现指针的静态类型与被删除对象的动态类型不符的情况。例如当我们delete一个`Quote*`类型的指针，则该指针有可能实际指向了一个`Bulk_quote`类型的对象。这样的话，编译器就必须清楚它应该执行的时`Bulk_quote`的析构函数。
	
		class Quote {
		public:
			virtual ~Quote() = default;
		};	
		
		Quote *itemp = new Quote;     // 静态类型与动态类型一致
		delete itemP;                 // 调用Quote的析构函数
		itemP = new Bulk_quote;       // 静态类型与动态类型不一致
		delete itemP;                 // 调用Bulk_quote的析构函数
	
	
12.  
	





