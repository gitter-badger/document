### lambda表达式总结 ###
------------------------
#### Overview ####
------------------
我们可以向一个算法传递任何类别的**可调用对象（callable object）**。对于一个对象或一个表达式，如果可以对其使用**调用运算符**，则称它为可调用的。即：如果`e`是一个可调用的表达式，则我们可以编写代码`e(args)`，其中`args`是一个用逗号分隔的一个或多个参数的列表。

在C++中，有四种可调用对象：（1）函数；（2）函数指针；（3）重载了函数调用运算符的类；（4）lambda表达式。

#### Summary ####
-----------------
1. **什么是lambda表示式**

	一个lambda表达式表示**一个可调用的代码单元**。我们可以将其理解为一个未命名的内联函数。与任何函数类似，一个lambda具有一个返回类型、一个参数列表和一个函数体。但与函数不同，**lambda可能定义在函数内部**。一个lambda表达式具有如下形式：
	
		[capture list](parameter list) -> return type { function body }

	**capture list**：捕获列表。这是一个lambda所在函数中定义的局部变量的列表（通常为空）。
	
	**paramter list**，**return type**和**funciton body**与任何普通函数一样。
	
	与普通函数不同，lambda必须使用**尾置返回**来指定返回类型。
	
	我们可以忽略参数列表和返回类型，但必须永远包含捕获列表和函数体：
	
		auto f = [ ] { return 42; };  // 定义了一个可调用对象f，它不接受参数，只返回42
		cout << f() << endl;          // 打印42
		
	**如果忽略返回类型，lambda根据函数体中的代码推断出返回类型**。如果函数体只是一个return语句，则返回类型从返回的表达式的类型推断出来，否则返回类型为void。
	
2. **向lambda传递参数**

   与普通函数不同，lambda不能有默认参数，所以，其实参数目永远与形参数目相等。一旦，形参初始化完毕，就可以执行函数体了。
   
   如：
   
   		[](const string &a, const string &b)
   		  { return a.size() < b.size(); }
   		  
   下面介绍一下如何使用lambda的捕获列表。
   
   - **值捕获**
   
     **采用值捕获的前提是变量可以拷贝**，被捕获的变量的值是在lambda**创建**时拷贝，而不是**调用时拷贝**。
     
			void fcn1()
			{
				size_t v1 = v2;
				auto f = [v1] { return v2; };
				v1 = 0;
				auto j = f();   // j为42
			}
			
   - **引用捕获**
	
	  当我们确保被引用的对象在lambda执行的时候是存在，我们就可以采用引用方式捕获一个变量。
	  
	  		void fucn2()
	  		{
	  			size_t v1 = v2;
	  			auto f2 = [&v1]{ return v1; };
	  			v1 = 0;
	  			auto j = f2();  // j为0
	  		}
	  		
   - **隐式捕获**
     
       除了显式列出我们希望使用的来自所在函数的变量之外，还可以让**编译器根据lambda体中代码来推断我们要使用哪些变量**。为了指示编译器推断捕获列表，应在捕获列表中写一个`&`或`=`。**`&`告诉编译器采用捕获引用方式，`=`则表示采用值捕获方式**。
       
       如：
       		
       		......
       		size_t sz = 10;
       		......
       		wc = find_if(words.begin(), words.end(),
       		             [=](const string &s)
       		                { return s.size() >= sz; });
       		......
       	
       	**如果我们希望对一部分变量采用值捕获，对其他变量采用引用捕获，可以混合使用隐式捕获和显式捕获**：
       	
       		void biggies(vector<string> &words, vector<string>::size_type sz,
       		             ostream& os = cout, char c = ' ')
       		{
       			// os隐式捕获，c是显式值捕获
       			for_each(words.begin(), words.end(),
       			         [&, c](const string &s) { os << s << c; } );
       		}
       		
       **ATTENTION**： **当我们混合使用隐式捕获和显式捕获，显式捕获的变量必须使用与隐式捕获不同的方式**。即当隐式捕获是引用方式，则显式捕获则必须采用值方式。
       
       请参考这张**lambda捕获列表**：
       
	|lambda捕获列表 |  | 
	| ------------ | ------------- |
	|`[ ]`  | 空捕获列表。lambda不能使用所在函数中的变量。一个lambda只有捕获变量后才能使用它们 | 
	| `[names]`| `names`是一个逗号分隔的名字列表。这些名字都是lambda所在函数的局部变量。默认情况下，捕获列表中的变量都被拷贝。名字前如果使用了`&`，则采用引用捕获方式。 |
	| `[&]`| 隐式捕获列表，采用引用捕获方式。lambda体中所使用的来自所在函数的实体都采用引用方式使用|
	| `[=]` | 隐式捕获列表，采用值捕获方式。  |
	| `[&, identifier_list]` | `identifier_list`是一个逗号分隔的列表。包含0个或多个来自所在函数的变量。这些变量采用值捕获方式，而任何隐式捕获的变量都采用引用方式捕获。`identifier_list`中名字前面不能使用`&`。 |
    | `[=, identifier_list]` | `identifier_list`中变量都采用引用方式捕获，而任何隐式捕获的变量都采用值方式捕获。`identifier_list`中的名字不能包括`this`，且这些名字之前必须使用`&`。

3. **可变lambda**

	**默认情况下，对于一个值被拷贝的变量，lambda不会改变其值**。如果我们希望能改变一个被捕获的变量的值，就必须在参数列表首加上关键字`mutable`。
	
	因此，可变lambda能省略参数列表：
	
		void fcn3()
		{
			size_t v1 = 42;
			auto f = [v1]() mutable { return ++v1; };
			v1 = 0;
			auto j = f();     // j为43
		}
		
	一个引用捕获的变量是否（如往常一样）可以修改依赖于此引用指向的**是一个const类型还是一个非const类型**。
4. **指定lambda返回类型**

	**默认情况下**，如果一个lambda体包含`return`之外的任何语句，则编译器假定此lambda返回`void`。与其他返回`void`的函数类似，被推断返回`void`的lambda不能返回值。
	
	如：
		
		transform(vi.begin(), vi.end(), vi.begin(), [](int i) { return i < 0 ? -i; i; }); // correct
		
		transform(vi.begin(), vi.end(), vi.begin(), [](int i) { if (i < 0) return -i; else return i;)} 
		// 不仅有return语句，还有if语句。编译器假定返回void，但是它返回int，所以出错
		// 应该这样表达描述：
		
		transform(vi.begin(), vi.end(), vi.begin(), [](int i) -> int
												     { if (i < 0) return -i; else return i; });

5. **lambda表达式的本质**  
	
	当我们编写了一个lambda后，编译器将该表达式**翻译成一个未命名类的未命名对象**。该类中含有一个**重载的函数调用运算符**。
	
	当我们使用：
	
		stable_sort(words.begin(), words.end(), [](const string &a, const string &b)
		                                        { return a.size() < b.size(); });
		                                        
	其行为类似于下面这个类的一个未命名对象：
	
		class ShorterString {
		public:
			bool operator() (const string &s1, const string &s2) const
			{
				return s1.size() < s2.size();
			}
		};
		
	我们也可以将其使用在`stable_sort`中：
	
		stable_sort(words.begin(), words.end(), ShorterString());
		
	默认情况下，lambda不能改变捕获的变量，所以其产生的类当中的函数调用运算符是一个const成员函数。
	
	当一个lambda表达式通过引用捕获变量时，将由程序负责确保lambda执行时引用所引的对象确实存在。因此，编译器可以直接使用该引用而无须在lambda产生的类中将其存储为数据成员。
	
	相反，通过值捕获的变量被拷贝到lambda中，因此，这种lambda产生的类必须为每个值捕获的变量建立对应的数据成员，同时创建构造函数。
	
	如：
	
		auto wc = find_if(words.begin(), words.end(), [sz](const string &a)
		                                              {return a.size() >= sz; });
		                                              
	将产生如下的类：
	
		class SizeComp {
		public:
			SizeComp(size_t n) : sz(n) { }
			
			bool operator() (const string &s) 
			{
				return s.size() >= sz;
			}
			
		private:
			size_t sz;
		};
		
	则我们可以这样使用：
	
		auto wc = find_if(words.begin(), words.end(), SizeComp(sz));
		
	lambda表达式产生的类不含默认的构造函数、赋值运算符及默认析构函数；它是否含有默认的拷贝/移动构造函数则通常要视捕获的数据成员类型而定。
	
