### C++11 知识点小结 ###
----------------------

#### constexpr和常量表达式 ####

**常量表达式（const expression）的定义：**

>值不会改变并且在**编译过程中**就能得到计算结果的表达式。

显然，**字面值属于常量表达式**。**用常量表达式初始化的const对象也是常量表达式**。

	const int max_files = 20;         // max_files 是常量表达式
	const int limit = max_files + 1;  // limit是常量表达式
	int staff_size = 27;              // staff_size不是常量表达式，因为它不是const对象
	const int sz = get_size();        // sz不是常量表达式，因为其具体值要在运行时才能确定

**需求：在一个复杂系统中，很难（几乎肯定不可能）分辨一个初始值到底是不是常量表达式**。当然可以定义一个const变量并把它的初始值设为我们认为的某个常量表达式，但在实际使用中，尽管要求如此却常常发现初始值并非常量表达式的情况。

因此，C++11新标准规定：**允许将变量声明为constexpr类型以便由编译器来验证变量的值是否是一个常量表达式**。声明为constexpr的变量一定是一个常量，而且必须要用常量表达式来初始化。

	constexpr int mf = 20;           // 20是字面值，所以是常量表达式
	constexpr int limit = mf + 1;    // mf + 1是常量表达式
	constexpr int sz = size();       // 只有当size是一个constexpr函数时，这才是一条正确的语句，
                                     // 否则编译报错
                                     
>**---> 一个constexpr指针的初始值必须是nullptr或者0，或者是存储于某个固定地址中的对象**。

>**---> constexpr把它所定义的对象置为了顶层const**

	const int *p = nullptr;          // p是一个指向整型常量的指针
	constexpr int *q = nullptr;      // q是一个指向整数的常量指针
	
