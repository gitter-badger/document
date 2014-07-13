### C++11 知识点小结 ###
----------------------

#### 范围for（range for）语句 ####

C++11提供了一种遍历给定序列的每个元素并对序列中的每个值执行某种操作的语句，其语法形式为：

	for (declaration : expression)
		statement

其中，
	
**expression部分是用于表示序列的一个对象**

**declaration部分负责定义一个变量，该变量将被用于访问序列中的基础元素，每次迭代，declaration部分的变量会被初始化为expression部分的下一个元素值**

一个string对象表示一个字符的序列，**因此string对象可以用为范围for语句中的expression部分**，如下所示：

	sting str("some string");
	
	for (auto c : str)
		cout << c << endl;
		
再来一个稍微复杂的例子：

	string s("Hello World !!!");
	
	decltype(s.size()) punct_cnt = 0;
	
	for (auto c : s)
		if (ispunct(c))
			++punct_cnt;
	cout << punct_cnt
	     << " punctuation characters in " << s << endl;
	     
如果你想修改string对象中字符的值，**必须把循环变量定义成引用类型**。

	string s("Hello World !!!");
	
	for (auto &c : s)
		c = toupper(c);
	cout << s << endl;
