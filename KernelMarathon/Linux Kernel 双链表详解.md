### Linux Kernel 双链表详解 ###
---------------------------------
#### 1. Linux Kernel链表的实现 ####

在Linux Kernel中，如果要说使用最多的数据结构，那就非链表莫属。在`<linux/list.h>`中，Linux Kernel以一种简洁明了的方式实现了链表的绝大多数操作(其实可以说是所有操作)。下面我们就来看看在Linux Kernel中，链表这种经典的数据结构是如何被演绎的。

首先来看这个最基本的双链表数据结构 (**链表中用的最多的还是双链表**)：

	struct list_head {
		struct list_head *next, *prev;
	};

不难理解，单个链表头结构中包含一个next和prev指针。

那么，这个简单的数据结构是如何在Linux Kernel中被广泛使用的呢？答案是：**作为数据结构成员嵌入到另外一个数据结构中去**。比如我自己设计了一个玩具数据结构如下所示：
		
	struct fox {
		int data;
		struct list_head fox_list;
	};
	
为了实现将不同的fox组合成一个list，我们需要在fox数据结构中嵌入`struct list_head`成员。

接着让我们看看主流程是如何将不同的fox串联起来的：
		
					......
		struct fox red_fox, blue_fox, green_fox;
		
		LIST_HEAD(module_list);
		
		list_add(&red_fox.fox_list, module_list);
		list_add(&blue_fox.fox_list, module_list);
		list_add(&green_fox.fox_list, module_list);
					......
					
如上所示，我们可以以`module_list`作为入口，来遍历串联在这条链表上的`struct fox`上的`module_list`成员，然后通过`container_of`来访问父结构。

这种以一种子数据结构嵌入父数据结构的方式，其实在Linux Kernel中非常普遍，几乎所有关键的数据结构都是由好几种基础数据结构组合合成。通过对子数据结构的处理，然后再用`container_of`来访问父结构，这其实可以来构建C语言的OO编程，例如Linux Kernel中的VFS和设备驱动模型。
	
#### 2. `list.h`中的 API ####

##### a. `struct list_head`的初始化 #####
可以采用以下两种初始化方式：

	// 方法1：静态方式
	static LIST_HEAD(modules);
		
	// 方法2：动态方式
	struct list_head list;
	INIT_LIST_HEAD(&list);
		
以上两种方式都是创建了一个链表头。其中，方法一是以宏的方式实现，方法二是以内嵌函数的方式实现。从具体实现的角度来看，二者其实都没什么太大的区别，都是创建一个链表头，并把`next`和`prev`都指向自己。如下所示：

	struct list_head {
		struct list_head *next, *prev;
	};

	#define LIST_HEAD_INIT(name) { &(name), &(name) }

	#define LIST_HEAD(name) \
		struct list_head name = LIST_HEAD_INIT(name)

	static inline void INIT_LIST_HEAD(struct list_head *list)
	{
		list->next = list;
		list->prev = list;
	}	

##### b. 链表的插入，删除，替换，移动，切分和合并#####

注意：`list.h`中的API传入的参数都是`struct list_head`的指针类型，所以我们在日常使用中经常通过对该数据结构取地址的方式来使用下列API。
	
- 插入操作

		static inline list_add(struct list_head *new, struct list_head *head);
												         // 将new插入到head后面
		static inline list_add_tail(struct list_head *new, struct list_head *head);
		                                                 // 将new插入到head前面 
		// 注意，head可以使list中任意一个节点
	
比如我们这样使用：
		
		struct foo_module {
			unsigned int number;
			struct list_head module_list;
		};
	
		static LIST_HEAD(modules);	
		
		int main(void) {
			struct foo_module module_a, module_b, module_c, module_d;
			list_add(&module_a.module_list, &modules);
			list_add(&module_b.module_list, &modules);
			list_add(&module_c.module_list, &modules);
			list_add_tail(&module_d.module_list, &modules); // 因为是循环链表，所以放在头结点前面也就是放在最后
		}
		
		// The modules list will be:
		// modules ---> module_c ---> module_b ---> module_a ---> module_d
		
- 删除操作

		static inline list_del(struct list_head *entry);
										// 删除指定节点
		static inline list_del_init(struct list_head *entry);
		                               //  删除指定节点并用INIT_LIST_HEAD重新初始化该节点
		                               
- 替换操作

		static inline list_replace(struct list_head *old, struct list_head *new);
										// 将指定的old节点替换成new节点
		static inline list_replace_init(struct list_head *old, struct list_head *new);
										// 将指定的old节点替换成new节点并用INIT_LIST_HEAD重新初始化old

- 移动操作

		static inline list_move(struct list_head *list, struct list_head *head);
										// 删除list节点并将其移动到head节点之后
		static inline list_move_tail(struct list_head *list, struct list_head *head);
										// 删除list节点并将其移动到head节点之前
		static inline list_rotate_left(struct list_head *head);
										// 将head开始的链表进行循环左移

- 切分操作

		static inline void list_cut_position(struct list_head *list, 
							struct list_head *head, struct list_head *entry);
				// 将从head开始(不包含head)到entry开始这段链表添加到list上，如果list原始不为空，则此操作会清除之前数据
							
- 合并操作

		static inline void list_splice(const struct list_head *list, struct list_head *head);
				// 将新链表list添加到head之后
		static inline void list_splice_tail(const struct list_head *list, struct list_head *head);
				// 将新链表list添加到head之前
		static inline void list_splice_init(const struct list_head *list, struct list_head *head);
				// 同上，但是会用INIT_LIST_HEAD去初始化list指针
		static inline void list_splice_tail_init(const struct list_head *list, struct list_head *head);
				// 同上，但是会用INIT_LIST_HEAD去初始化list指针

##### c. 链表判断操作 #####

链表判断操作主要是指判断当前链表是否为空，是否为单节点等一系列判断操作。主要由以下几个API：

	static inline int list_is_last(const struct list_head *list, const struct list_head *head);
				// 测试list节点是否是以head节点开始的链表的最后一个元素
	static inline int list_is_singular(const struct list_head *head);
				// 测试head节点是不是只有一个entry，即是否只有一个元素
	static inline int list_empty(const struct list_head *head);
				// 测试list是否为空
	static inline int list_empty_careful(const struct list_head *head);
				// ????
	
		
##### d. 链表遍历操作 #####

`list.h`提供了很多链表的遍历操作(例如正向遍历和反向遍历)，我们首先先介绍两个最简单的遍历API：
	
	#define list_for_each(pos, head） \
		for (pos = (head)->next; pos != (head); \
			pos = pos->next)

	#define list_for_each_safe(pos, n, head) \
		for (pos = (head)->prev, n = pos->next; \
			 pos != (head); \
			 pos = n, n = pos->next)
				
这样便引出一个问题：为什么需要有safe版本的遍历操作呢？我们可以看到，safe版本的遍历是将`pos->next`保存到了`n`。试想这么一个场景：
	
	......
	list_for_each(pos, head) {
		list_del(pos);
		// do something else...	
	}
如果是使用非安全版本的遍历函数便会导致一个严重问题：`pos`指针都已经被delete了，那么for循环中的`pos->next`就肯定会访问到一个不可访问地址从而导致段错误。如果是使用safe版本的遍历函数，以上delete操作就是安全的，因为我们使用了中间变量`n`去保存`pos->next`，从而避免了这种情况。

#### 3. 如何使用Linux Kernel链表 ---> 一个小实验 ####

下面会采用一个小实验来将本篇讲到的内容做一个小小的总结，这个实验其实就是一段很简单的代码，通过使用`<linux/list.h`中提供的基础 API 来完成我们对链表的操作。

首先，我会设计一个玩具数据结构叫`foo_module`，内部嵌入一个`struct list_head`数据类型的`module_list`。接着我会创建一个`foo_module`数组，分别用`modules_list1`和`modules_list2`串起来，然后在利用这两个list做一些切分和合并的动作。所有代码如下所示，并参考其中的注释来理解。

	#include <stdio.h>
	#include "list.h"

	#define MAX_MODULES_NO       20
	#define DEBUG                 0

	struct foo_module {
    	unsigned int number;
    	struct list_head module_list;
	};

	static LIST_HEAD(modules_list1);
	static LIST_HEAD(modules_list2);
	static LIST_HEAD(modules_new);

	static struct foo_module foo_modules[MAX_MODULES_NO];

	static void init_foo_modules_arr(struct foo_module *foo_modules)
	{
    	int i;
    	for (i = 0; i < MAX_MODULES_NO; i++) {
        	foo_modules[i].number = i;
    	}
		#if DEBUG
   		for (i = 0; i < MAX_MODULES_NO; i++) {
        	printf("foo_modules[%d].number = %d\n", i, foo_modules[i].number);
   		}
		#endif
	}

	int main(void)
	{
    	int i;
    	struct foo_module *pos, *tmp;

    	init_foo_modules_arr(foo_modules);
    
    	for (i = 0; i < 5; i++) {
        	list_add(&(foo_modules[i].module_list), &modules_list1);
        	list_add(&(foo_modules[i + 5].module_list), &modules_list2);
    	}

    	// operations in modules_list1, including list_add and list_add_tail
    	list_add_tail(&(foo_modules[11].module_list), &(foo_modules[3].module_list));
    	list_add_tail(&(foo_modules[13].module_list), &(modules_list1));

    	// operations in modules_list2, including list_del and list_replace
    	list_del(&(foo_modules[8].module_list));
    	list_replace(&(foo_modules[9].module_list), &(foo_modules[19].module_list));

    	printf("scan modules_list1 ===>\n");        // it will be ===> <-H->4->11->3->2->1->0->13
    	list_for_each_entry_safe(pos, tmp, &modules_list1, module_list) {
        	printf("number = %d\n", pos->number);
    	}

    	printf("scan modules_list2 ===>\n");        // it will be ===> <-H->19->7->6->5
   		list_for_each_entry_safe(pos, tmp, &modules_list2, module_list) {
        	printf("number = %d\n", pos->number);
   		}

    	// operations in modules_list1, including list_move, list_move_tail and list_rotate_left
    	list_rotate_left(&modules_list1);
    	list_move(&(foo_modules[2].module_list), &modules_list1);
    	list_move_tail(&(foo_modules[7].module_list), &modules_list1);

    	printf("scan modules_list1 ===>\n");        // it will be ===> <-H->2->11->3->1->0->13->4->7
    	list_for_each_entry_safe(pos, tmp, &modules_list1, module_list) {
       		printf("number = %d\n", pos->number);
    	}

    	printf("scan modules_list2 ===>\n");        // it will be ===> <-H->19->6->5
    	list_for_each_entry_safe(pos, tmp, &modules_list2, module_list) {
        	printf("number = %d\n", pos->number);
    	}

    	// operations about cuting and splicing
    	list_cut_position(&modules_new, &(foo_modules[2].module_list), &(foo_modules[4].module_list));
    
    	printf("scan modules_new ===>\n");          // it will be ===> <-H->11->3->1->0->13->4
    	list_for_each_entry_safe(pos, tmp, &modules_new, module_list) {
        	printf("number = %d\n", pos->number);
    	}

    	printf("scan modules_list1 ===>\n");        // it will be ===> <-H->2->7
    	list_for_each_entry_safe(pos, tmp, &modules_list1, module_list) {
        	printf("number = %d\n", pos->number);
    	}

    	list_splice(&modules_new, &modules_list2);
    	printf("scan modules_list2 ===>\n");        // it will be ===> <-H->11->3->1->0->13->4->19->6->5
    	list_for_each_entry_safe(pos, tmp, &modules_list2, module_list) {
        	printf("number = %d\n", pos->number);
    	}

    	list_splice_tail(&modules_list2, &modules_list1);
    	printf("scan modules_list1 ===>\n");        // it will be ===> <-H->2->7->11->3->1->0->13->4->19->6->5
    	list_for_each_entry_safe(pos, tmp, &modules_list1, module_list) {
        	printf("number = %d\n", pos->number);
    	}

    	return 0;
	}


`
