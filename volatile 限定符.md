### volatile 限定符 ###
----------------------

直接处理硬件的程序常常包含这样的数据元素，**它们的值由程序直接控制之外的过程控制**。例如，程序可能包含一个由系统时钟定时更新的变量。**当对象的值可能在程序的控制或检测之外被改变时，应该将该对象声明为`volatile`**。

**关键字`volatile`告诉编译器不应对这样的对象进行优化**。

	volatile int display_register;  // 该int的值可能发生改变
	volatile Task *curr_task;       // curr_task指向一个volatile对象
	volatile int iax[max_size];     // iax的每个元素都是volatile
	
const和volatile限定符互相没有什么影响，二者可以同时存在。它们二位合称“C-V限定符”。

volatile修饰变量的方式跟const是一样的，如下所示：

	volatile int v;               // v是一个volatile int
	int *volatile vip;            // vip是一个volatile指针，指向int
	volatile int *ivp;            // ivp 是一个指针，指向volatile int
	volatile int *volatile vivp;  // 这个不用我多说什么的了吧
	
	int *ip = &v;  // error
	ivp = &v;      // correct
	vivp = &v;     // correct
	
**跟const的使用同样道理，我们可以将非volatile引用和指针传给volatile形参**。

**只有volatile的成员函数才能被volatile的对象调用，这点跟const也是一样的**。

>**---> const和volatile的一个重要区别是我们不能使用合成的拷贝/移动构造函数及赋值运算符初始化volatile对象或从volatile对象赋值**。

所以此时我们必须自定义。这点未来有时间我才慢慢研究一下。
	
	

