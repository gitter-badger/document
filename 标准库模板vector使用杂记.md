### 标准库模板vector使用杂记 ###
-----------------------------

1. 向vector对象中添加元素

	vector最有价值的地方就是可以在程序运行中高效地添加元素。这样，当我们遇到无法确定元素个数的场景的时候，就可以优先考虑使用vector模板。

	这里采用`v.push_back(t)`操作来向v的尾端添加一个值为t的元素。我们会先创建一个空的vector对象，然后在运行过程中加入元素：
	
		vector<int> v;
		
		for (int i = 0; i != 100; i++)
			v.push_back(i);

2. 遍历vector对象

	访问vector对象中的值可以有两种方式，一种是C++11推出的范围for语句，另外一种是采用vector对象索引。
	
	第一种方式使用如下：
	
		vector<int> v{1, 2, 3, 4, 5, 6, 7, 8, 9};
		for (auto &i : v)
			i *= i;
		for (auto &i : v)
			cout << i << " ";
			
		cout << endl;
		
	将上面的程序用第二种方法改写，即为：
	
		vector<int> v{1, 2, 3, 4, 5, 6, 7, 8, 9};
		for (decltype(v.size()) ix = 0; ix != v.size(); ix++)
			v[ix] *= v[ix];
		for (decltype(v.size()) ix = 0; ix != v.size(); ix++)
			cout << v[ix];
			
		cout << endl; 
		
	**注意：不能用下标的形式添加元素！**如下所示，这是一种错误的做法：

		vector<int> ivec;
		for (decltype(ivec.size()) ix = 0; ix != 10; ix++)
			ivec[ix] = ix;
			
	`ivec`是一个空的元素，根本不包含任何元素，**当然也就不能通过下标去访问任何元素！**这种行为编译器也许检查不出来，但是其运行时行为是未定义的（core dump）。
	
	**vector对象（以及string对象）的下标运算符可用于访问已存在的元素而不能添加元素。**
	
3. 要使用`size_type`，**需首先指定它使由哪种类型定义的**。vector对象的类型总是包含着元素的类型。

		vector<int>::size_type    // correct
		vector::size_type         // error


	

		
