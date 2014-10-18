### chapter 9 顺序容器 ###
--------------------------
1. 对于下面的程序任务，`vector`、`deque`和`list`哪种容器最为合适 ？解释你的选择的理由。如果没有哪一种容器优于其他容器，也请解释理由。

   (a) 读取固定数量的单词，将它们按字典序插入到容器中。我们将在下一章中看到，关联容器更合适这个问题。
    
   用`vector`输入，然后用`sort`排序。   
   
   (b) 读取未知数量的单词，总是将新单词插入到末尾。删除操作在头部进行。
   
   由于是头尾添加和删除元素，用`deque`.
    
   
   (c) 从一个文件读取未知数量的整数。将这些数排序，然后将它们打印到标准输出。

	用`vector`输入，然后用`sort`排序，接着遍历输出。  
	
	
2. 定义一个`list`对象，其元素类型是`int`的`deque`。

   这题其实是想考察容器内元素也可以是容器。
   
   		list<deque<int>> l_;
   		
3. 构成迭代器范围的迭代器有何限制 ？
   
   如果满足如下条件，两个迭代器`begin`和`end`构成一个迭代器范围：
   - 它们指向一个容器中的元素，或者是容器最后一个元素之后的位置，且
   - 我们可以通过反复递增`begin`来到达`end`

4. 编写函数，接受一对指向`vector<int>`的迭代器和一个`int`值。在两个迭代器指定的范围中查找给定的值，返回一个布尔值来指出是否找到。

		bool find_element(vector<int>& vi, int target)		{    		for (auto it = vi.cbegin(); it != vi.cend(); ++it)    		{        		if (*it == target)            		return true;    		}    		return false;		}

5. 重写上一题的函数，返回一个迭代器指向找到的元素。注意，程序必须处理未找到给定值的情况。

		vector<int>::const_iterator find_element(vector<int>& vi, int target)		{    		for (auto it = vi.cbegin(); it != vi.cend(); ++it)    		{        		if (*it == target)            		return it;    		}    		return vi.cend();		}6. 下面程序有何错误 ？你应该如何修改它 ？

   		list<int> lst1;
   		list<int>::iterator iter1 = lst1.begin(),
   		                    iter2 = lst1.end();
   		                    
   		while (ite1 < iter2) / * ... */
   		
   	迭代器不支持`operator<`操作。除非我们自己重新实现`list`。`list` 支持的是`operator!=`操作。所以我们可以这样修改：
   	
   		......
   		while(iter1 != iter2) /* ... */
   		......7. 为了索引`int`的`vector`中的元素，应该使用什么类型 ？
		vector<int>::size_type
8. 为了读取`string`的`list`中的元素，应该使用什么类型 ？如果写入`list`，又该使用什么类型 ？
		list<string>::value_type ---> 读取元素
		list<string>::reference  ---> 写入元素
9. `begin`和`cbegin`两个函数有什么不同 ？

	`begin`  ---> 返回的时`iterator`或`const_iterator`类型的指向首元素的迭代器，返回类型取决于变量类型
	
	`cbegin` ---> 返回`const_iterator`类型的指向首元素的迭代器10. 下面4个对象分别是什么类型 ？
		vector<int> v1; 		const vector<int> v2;		auto it1 = v1.begin(), it2 = v2.begin();		auto it3 = v1.cbegin(), it4 = v2.cbegin();

	由于`begin`重载了const和非const两个版本，所以`it1`是`vector<int>::iterator`类型，`it2`是`vector<int>::const_iterator`。
	
	`cbegin`就是直接返回`const_iterator`类型，所以`it3`和`it4`都是`vector<int>::const_iterator`。
	
11. 对6种创建和初始化`vector`对象的方法，每一种都给出一个实例。解释每个`vector`包含什么值。
    
    	vector<string> svec;  // svec是一个包含string类型的空容器
    	
    	vector<string> svec(another_svec); // svec是another_svec的一个副本
    	
    	vector<string> svec(10); // svec是含有10个string对象的容器，每个string进行值初始化，即为空串
    	
    	vector<string> svec(10, "hello"); // svec是含有10个string对象的容器，每个string都初始化为“hello”
    	
    	vector<string> svec{"hello", "world", "fuck", "the", "king"}; // C++11中加入的花括号初始化
    	
  		vector<string> svec = {...}; // 同上
  		
  		vector<string> sevc(another.begin(), another.end()); // 记住，容器类型要相融


12. 对于接受一个容器创建其拷贝的构造函数，和接受两个迭代器创建拷贝的构造函数，解释它们的不同。
    
    一个是完全拷贝，且必须是同型的容器；另外一个可以指定迭代器范围，只需要容器元素类型相容即可。

13. 如何从一个`list<int>`初始化一个`vector<double>` ? 从一个`vector<int>`又该如何创建 ？编写代码验证你的答案。 

	    list<int> ilist = {123, 456, 789, 987 };    	vector<double> dvec(ilist.begin(), ilist.end());		#if 0    	for (vector<double>::const_reference d : dvec)  // C++11        	cout << d << endl;		#else    	for (vector<double>::const_iterator it = dvec.begin();         	it != dvec.end(); ++it)       		cout << *it << endl;        #endif
     反过来转换也是同理。

14. 编写程序，将一个`list`中的`char *`指针（指向C风格字符串）元素赋值给一个`vector`中的`string`。
	
		vector<string> svec;
		list<const char*> oldstyle = {"hello", "world"};
		svec.assign(list.begin(), list.end());

15. 编写程序，判定两个`vector<int>`是否相等 ？

       	bool isVectorEqual(const vector<int>& v1, const vector<int>& v2)
       	{
       		return v1 == v2;
       	}
16. 重写上一题的程序，比较一个`list<int>`中的元素和一个`vector<int>`中的元素。

		bool isVectorEqualsList(vector<int>& v, list<int>& l)
		{
			vector<int> l_to_v(l.begin(), l.end());
			return l_to_v == v;
		}
17. 假定`c1`和`c2`是两个容器，下面的比较操作有何限制（如果有的话）？
       
        if (c1 < c2)
    
    首先，两个容器必须是同类型；其次，容器中的元素已经定义了`operator<`操作。

18. 编写程序，从标准输入读取`string`序列，存入一个`deque`中。编写一个循环，用迭代器打印`deque`中的元素。

		......
    	string word;    	deque<string> dstring;    	while (cin >> word)         	dstring.push_back(word);    	for (deque<string>::const_iterator it = dstring.begin();         	 it != dstring.end(); ++it)        	 cout << *it << endl;
		......	
19. 重写上题的程序，用`list`替代`deque`。列出程序要做出哪些改变。

     直接换。没啥好改变的。
     
20. 编写程序，从一个`list<int>`拷贝元素到两个`deque`中。值为偶数的所有元素都拷贝到一个`deque`中，而奇数值都拷贝到另一个`deque`中。
    
    	......
    	list<int> ilist;    	deque<int> odd_num, even_num;   		for (int i = 0; i != 20; i++)        	ilist.push_back(i);    	for (list<int>::const_iterator it = ilist.begin();             it != ilist.end(); ++it)    	{        	if (*it % 2)            	odd_num.push_back(*it);        	else            	even_num.push_back(*it);    	}
		......21. 如果我们将308页中使用`insert`返回值将元素添加到`list`中的循环程序改写为将元素插入到`vector`中，分析循环将如何工作。
    
    原理都是一样。
    
22. 假定`iv`是一个`int`的`vector`，下面的程序存在什么错误 ？你将如何修改 ？

		vector<int>::iterator iter = iv.begin(),
		                      mid = iv.begin() + iv.size() / 2;
		
		while (iter != mid)
			if (*iter == some_val)
				iv.insert(iter, 2 * some_val);
				
	因为`iter`一直指向原始容器的开始元素，然后又进行了`insert`动作，这样指向原始位置的迭代器就会失效，因为`vector`中的元素进行了重排。我们应该使用`insert`函数的返回值作为新的`iter`。


23. 在本节第一个程序中，若`c.size()`为1，则`val`、`val1`、`val3`和`val4`的值会是多少 ？
	
	略。

24. 编写程序，分别使用`at`、下标运算符、`front`和`begin`提取一个`vector`中的第一个元素。在一个空`vector`上测试你的程序。

		......
	    vector<int> ivec;    	for (int i = 123; i != 133; i++)        	vec.push_back(i);    	auto it = ivec.begin();    	cout << ivec[0] << endl;    	cout << ivec.at(0) << endl;   	 	cout << ivec.front() << endl;    	cout << *it << endl;
		......
25. 对于第312页中删除一个范围内的元素的程序，如果`elem1`与`elem2`相等会发生什么？如果`elem2`是尾后迭代器，或者`elem1`和`elem2`皆为尾后迭代器，又会发生什么 ？
    
    我试验了几种情况，
    
    	if elem1 == elem2
    		不会发生delete动作
    	else if elem1 > elem2
    		恩，会发生一些有趣的现象，我看到的时元素不仅没有减少反而增加了，这是一个研究课题
    	else 
    		正常发生delete动作

26. 使用下面代码定义的`ia`，将`ia`拷贝到一个`vector`和一个`list`中。使用单迭代器版本的`erase`从`list`中删除奇数元素，从`vector`中删除偶数元素。
    
    	int ia[] = { 0, 1, 1, 2, 3, 5, 8, 13, 21, 55, 89 };
27. 		









   