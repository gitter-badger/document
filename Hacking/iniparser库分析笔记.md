### iniparser库分析笔记 ###
2013年6月2日下午4:21

**Author : GhostComputing**
#### 该死的序 ####
最近由于工作上需要，需要解析ini文件并将其转换成raw data的二进制格式，所以必须找到一个好用的ini文件解析工具。上github一查，发现**ndevilla/iniparser**就是一个小巧玲珑的ini文件分析库的C实现。说句实话，ini文件座位一种极为简单的配置文件格式，解析起来起来并不困难，但是看到前人已经实现好的模块，程序员懒惰的习惯立马上来了，**没必要重复造轮子**是吧。

#### 核心源码目录 ####
	src/
	├── dictionary.c
	├── dictionary.h
	├── iniparser.c
	└── iniparser.h

作者用了一个dictionary数据结构去保存解析后的section和key-value对。比如有如下这么一个ini文件：

	[host]
	serverip = 192.168.1.111
	ipaddr   = 192.168.1.114
	ethaddr  = 88:88:11:22:33:44
	
进过iniparser吃进去解析后会生成这么一个**dictionary**：
	
	"host"          ---> NULL
	"host:serverip" ---> "192.168.1.111"
	"host:ethaddr"  ---> "88:88:11:22:33:44"

采用dictionary的好处有：

1.	如果配置文件有重复配置信息，比如说：

		[host]
		serverip = 192.168.1.111
		serverip = 192.168.1.115
这样的写法在字典里面就会以最后一个配置为准。因为字典会将每个key进行hash，这样如果有重复的key的时候，可以很快发现并进行更改；
2. ini文件本来就是一种key-value的对应关系，从代码实现的角度来说，可以用关联数组(associated array)来实现，而iniparser实现的dictionary其实就是关联数组；
3. 可以实现key-value对无序地读写。

#### 源码分析 ####
> Talking is cheap, show me the code   
> Linus Torvalds


先从dictionary.c讲起。`dictionary`的结构如下所示：

	typedef struct _dictionary_ {
    	int             n ;     /** Number of entries in dictionary */
    	int             size ;  /** Storage size */
    	char        **  val ;   /** List of string values */
    	char        **  key ;   /** List of string keys */
    	unsigned     *  hash ;  /** List of hash values for keys */
	} dictionary ;
其中，`val`和`key`都是指向指针数组的指针。`hash`是指向`unsigned int`类型数组的指针，保存的是每个key值的hash值。

这个`dictionary.h`的API有如下几个：

	unsigned dictionary_hash(const char * key); 
	---> 计算key值的hash值，其中key是字符串
	
	dictionary * dictionary_new(int size);      
	---> dictionary的创建
	
	void dictionary_del(dictionary * vd);      
	---> dictionary的销毁
	
	char * dictionary_get(dictionary * d, const char * key, char * def); 
	---> 获取对应key的键值
	
	int dictionary_set(dictionay * d, const char * key, const char * val);
	---> 设置对应key的键值
	
	void dictionary_unset(dictionary * d, const char * key);
	---> 从字典中删除对应键值
	
	void dictionary_dump(dictionary * d, FILE * out);
	---> 以一定格式将字典中的内容dump到某个文件流中
这里插入一个小知识，我在看`dictionary_new`这个函数的时候学到的：

	malloc和calloc的区别：
	malloc：不会对分配的空间进行初始化；
	calloc：会对分配好的空间进行初始化，全部填充为0；
这二者可能实现细节上还有差异，但是就开发者使用上来看，用`calloc`可能可控性会更好一些。

以下重点讲讲`dictionary_set`函数。
这个函数的流程是(该死，我不知道mac上面有哪些便宜或者免费的画流程图的软件)：

**情景一：插入的key值已经在dictionary中**

先计算key值的hash值 ---> 根据hash值在dictionary的hash列表中寻找拥有相同hash值的key-value --->
为避免hash collision，如果找到这样的key，再对比一下相应的key ---> 确定是同样的key，修改对应的value即可；

**情景二：存储空间已经满了**

这个很简单，重新分配即可扩大空间即可；

**情景三：这是新的key-value对，而且存在新的空间**

从当前的dictionary入口n开始，循环遍历dictionary，找到可用空间。因为不考虑顺序，所以key-value之间是无序的。

其他的api，如getter，也是同setter同样的道理，在获取字典的key-value的时候先去扫描一下字典的hash列表，找到相同的hash后再进行key的比较。这样可以快速提高效率，避免字符串之间的比较。

讲过了基本的数据结构，现在来讲讲最关键的一个文件：`iniparser.c`。

初始的时候我还以为ini文件的解析是一件很麻烦的事情，待看过作者的实现后，发现不过也就那么一回事，用到的是典型的`state machine`的写法。
先不去讨论具体的函数实现，让我们来构思一下，假如现在需要你来设计一个ini解析器，你该从何下手？提示：先画出状态图。
	                                          
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	



	



