### C++11 知识点小结 ###
----------------------

#### initializer_list形参 ####

**需求：**有时我们无法提前预知应该向函数传递几个实参。例如，我们想要编写代码输出程序产生的错误信息，此时最好用同一个函数实现该项功能，以便对所有错误的处理能够整齐划一。然而，错误信息的种类不同，所以调用错误输出函数时传递的实参也各不相同。为了能编写处理不同数量实参的函数，C++11新标准提供了`initializer_list`的标准库类型。

**`initializer_list`其实是一种模板类型**。

>**--->如果函数的实参数量未知但是全部实参的类型都相同，我们可以使用`initializer_list`类型的形参**

	void error_msg(int err_code, initializer_list<string> i1)
	{
		if (err_code != 42) {
			cout << "no debug message" << endl;
		} else {
		    for (auto beg = i1.begin(); beg != i1.end(); beg++)
		    	cout << *beg << " ";
		    cout << endl;
	}
	
**记住：`initializer_list`对象中的元素永远是常量值，我们无法改变`initializer_list`对象中元素的值**。

**当我们想向`initializer_list`形参中传递一个值的序列时，必须把序列放在一对花括号内：**

	error(42, {"Hello", "World", "OK"});
