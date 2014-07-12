### C++ const 限定符小结 ###
--------------------------
### 5. const 成员函数 ###

典型地，在一个类中，如下所示：

	class Sales_data {
		......
		std::string isbn() const { return bookNo; }
		......
	};
	
isbn函数后回紧跟一个const关键字，这表明这是一个const成员函数。因为成员函数通过一个名为`this`的额外的隐式参数来访问调用它的那个对象，比如我们这么调用`isbn`函数：

	total.isbn();
	
其实编译器是看成：
	
	Sales_data::isbn(&total);
	
所以，这个函数其实可以看成：

	std::string isbn() const { return this->bookNo; }
	
**默认情况下，this的类型是指向类类型非常量版本的常量指针**。因为不能改变this的指向，所以this是一个常量指针。但是我们可以通过this指针改变被指向类的成员变量，所以是所指对象默认是非常量。这样，我们在默认情况下**不能把this绑定到一个常量对象上，也使得我们不能在一个常量对象上调用普通的成员函数。** 

举个例子：

	......
	class Sales_data {
	public:
		......
		std::string isbn() { return bookNo; }
		......
	private:
		std::string bookNo;
	};
	
	int main()
	{
		const Sales_data const_data;
		const_data.isbn();
		return 0;
	}
	
上面的程序如果用g++编译的话，会报如下错误：

> error: passing ‘const Sales_data’ as ‘this’ argument of ‘std::string Sales_data::isbn()’ discards qualifiers [-fpermissive]

如果我们去掉`const_data`的const属性，编译没有问题；如果我们将`isbn()`变成const成员函数，编译也可通过。

所以：

> **---> 紧跟在参数列表后面的const表示this是一个指向常量的指针**

现在我们可以再次把`isbn()`理解成：

	std::string Sales_data::isbn(const Sales_data * const this)
	{ return this->isbn; }
	
因为this是指向常量的指针，所以常量成员函数不能改变调用它的对象的内容。但是，被指向的对象可以不是const的。
