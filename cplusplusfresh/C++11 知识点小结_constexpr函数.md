### C++11 知识点小结 ###
-----------------------
#### constexpr函数 ####

**constexpr函数（constexpr function）是指能用于常量表达式的函数**。它必须遵循以下几项规定：


 - 函数的返回类型及所有形参的类型都得是字面值类型

 - 函数体中必须有且只有一条return语句


		constexpr int new_sz() { return 42; }
		constexpr int foo = new_sz();  // correct


**为了能在编译过程中随时展开，constexpr函数被隐式地指定为内联函数**。

**constexpr函数体内也可以包含其他运行时不执行任何操作的语句**，比如空语句，类型别名以及using声明。

**我们允许constexpr函数的返回值并非一个常量**：

	constexpr size_t scale(size_t cnt) { return new_sz() * cnt; }

如果`arg`常量表达式，`scale(arg)`也是常量表达式。

	int arr[scale(2)];    // correct
	int i = 2;            // i不是常量表达式
	int a2[scale(i)];     // error
