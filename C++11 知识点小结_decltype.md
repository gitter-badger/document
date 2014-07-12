### C++11 知识点小结 ###
----------------------
### 2. decltype 类型指示符 ###

**需求：**有时会遇到这种情况：希望从表达式的类型推断出要定义的变量的类型，但是不想用该表达式的值初始化变量（所以用不了auto，因为auto用该表达式的值进行初始化）。
	
为了解决这个问题，C++11新标准引入了**第二种类型说明符**`decltype`。**它的作用是选择并返回操作数的数据类型。**在此过程中，编译器分析表达式并得到它的类型，**却不实际计算表达式的值**。

	decltype(f()) sum = x;  // sum 的类型就是函数f的返回类型

编译器并不实际调用函数f，而是使用当调用发生时f的返回值类型作为sum的类型。

>**---> 如果decltype使用的表达式是一个变量，则decltype返回该变量的类型（包括顶层const和引用在内）**

	const int ci = 0, &cj = ci;
	decltype(ci) x = 0;           // x是const int类型
	decltype(cj) y = x;           // y的类型是const int&，绑定到变量x上
	decltype(cj) z;               // error，z是一个引用，必须初始化
	
>**---> 如果decltype使用的表达式不是一个变量，则decltype返回表达式结果对应的类型**

	int i = 42, *p = &i, &r = i;
	decltype(r + 0) b;            // correct，b是一个未初始化的int
	
>**---> 如果表达式的内容是解引用操作，则decltype将得到引用类型**

	decltype(*p) c;               // error，c是int& 类型，必须初始化
	
>**---> 如果给变量加上一层或多层括号，编译器会把它当成是一个表达式。变量是一种可以作为赋值语句左值的特殊表达式，所以这样的decltype将会得到引用类型**

	decltype((i)) d;             // error，d是int &，必须初始化
	decltype(i) e;               // correct，e是一个未初始化的int
	
**注意：decltype((variable))的结果永远是引用，而decltype(variable)结果只有当variable本身就是一个引用才是引用。**
