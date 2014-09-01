### Linux Kernel 等待队列的实现 ###
---------------------------------

#### 1. 等待队列的实现 ####

当进程无法立即满足请求时，它可以进入一种休眠(sleep)的状态。一旦满足某种唤醒条件的时候，该进程就会被重新调度进CPU运行。在Linux中，对进程的休眠可用等待队列( `wait queue` ) 机制来实现。

在Linux中，一个等待队列通过一个"等待队列头"来管理，等待队列头是一个类型为`wait_queue_head_t`的结构体，定义在`<linux/wait.h>`中。可通过以下两种方式来定义并初始化一个队列头：

		// 方法1：静态定义
		DECLARE_WAIT_QUEUE_HEAD(name);
		
		// 方法2：动态定义
		wait_queue_head_t my_queue;
		init_waitqueue_head(&my_queue);

这个`wait_queue_head_t`的定义是(其实很简单，就是一个自旋锁和一个链表头)：
		
		//---> include/linux/wait.h
		struct __wait_queue_head {
			spinlock_t lock;
			struct list_head task_list;		
		};
		typedef struct __wait_queue_head wait_queue_head_t;

顾名思义，这个数据结构抽象的是一个等待队列头。该队列的成员用以下`wait_queue_head_t`对象来抽象：

		//---> include/linux/wait.h
		typedef struct __wait_queue wait_queue_t;
		
		struct __wait_queue {
			unsigned int flags;
		#define WQ_FLAG_EXCLUSIVE	0x01
			void *private;
			wait_queue_func_t func;
			struct list_head task_list;
		};

这里，让我们想一个问题，为什么kernel要用`spin_lock`的形式来实现等待队列链表头？我们知道，`spin_lock`是多处理器系统的一种最常见的同步方式，通过忙等来获取锁。链表头所串联的等待队列成员不一定都是在同一个CPU上跑，很有可能是几个进程分布在几个不同的CPU上运行，所以，对等待队列的访问就成了一个critical section，必须加锁互斥。综上，kernel就采用了`spin_lock`这种实现形式。

接下来，我们具体分析两个最简单的接口：`wait_event`和`wake_up`。前者用以将进程睡眠，后者用于唤醒进程。

##### a. `wait_event`接口的实现 #####
如下所示：

	#define wait_event(wq, condition) 		\
	do {									\
		if (condition)	 					\
			break;							\
		__wait_event(wq, condition);		\
	} while (0)

在调用这个宏的时候，会先去检查`condition`是否成立。当`condition`不成立的时候，就会转去调用另外一个真正干活的宏`__wait_event`(kernel都喜欢这么干，很多宏基本上都会封装两次)。下面就是`__wait_event`：


	#define __wait_event(wq, condition) 							\
	do {															\
		DEFINE_WAIT(__wait);										\
																	\
		for (;;) {													\
			prepare_to_wait(&wq, &__wait, TASK_UNINTERRUPTIBLE);	\
			if (condition)											\
				break;												\
			schedule();												\
		}															\
		finish_wait(&wq, &__wait);									\
	} while (0)

首先会想通过用`DEFINE_WAIT`去创建一个名为`__wait`的等待队列成员(也就是`wait_queue_t`类型)。即：

	#define DEFINE_WAIT_FUNC(name, function)				 \
		wait_queue_t name = {						         \
			.private	= current,				             \
			.func		= function,				             \
			.task_list	= LIST_HEAD_INIT((name).task_list),	 \
		}
	
	#define DEFINE_WAIT(name) DEFINE_WAIT_FUNC(name, autoremove_wake_function)

当创建完`__wait`后，会进入`for`循环，接着调用`prepare_to_wait`去改变当前进程的状态：

	// kernel/wait.c
	void
	prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state)
	{
		unsigned long flags;
	
		wait->flags &= ~WQ_FLAG_EXCLUSIVE;
		spin_lock_irqsave(&q->lock, flags);
		if (list_empty(&wait->task_list)) 	// 为空说明是新的等待队列成员，否则就是已经分配到其他队列中的
			__add_wait_queue(q, wait);
		set_current_state(state);			// 改变当前进程的状态
		spin_unlock_irqrestore(&q->lock, flags);
	}

最后，在`for`循环中调用`schedule`形成一个调用点，就可以让scheduler去让该进程睡眠。

##### b. `wake_up`接口的实现 #####
接下来，我们来看`wake_up`接口的实现。如下所示：

	#define wake_up(x)			__wake_up(x, TASK_NORMAL, 1, NULL)

其中，`__wake_up`是：

	// kernel/sched.c
	void __wake_up(wait_queue_head_t *q, unsigned int mode,
				int nr_exclusive, void *key)
	{
		unsigned long flags;
	
		spin_lock_irqsave(&q->lock, flags);
		__wake_up_common(q, mode, nr_exclusive, 0, keya )
		spin_unlock_irqrestore(&q->lock, flags);
	}

其中，`__wake_up_common`是：

	// kernel/sched.c
	static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
				int nr_exclusive, int wake_flags, void *key)
	{
		wait_queue_t *curr, *next;
	
		list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
			unsigned flags = curr->flags;
	
			if (curr->func(curr, mode, wake_flags, key) &&
					(flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
				break;
		}
	}
`curr`指针就是我们之前创建的`wait`，这里会调用`curr->func`，其实就是调用`autoremove_wake_function`：

	// kernel/sched.c
	int auto_remove_wake_function(wait_queue_t *wait, unsigned mode, int sync, void *key)
	{
		int ret = default_wake_function(wait, mode, sync, key);
		
		if (ret)
			list_del_init(&wait->task_list);	  // 删除等待队列中已经被唤醒的成员
		return ret;
	}

	int default_wake_function(wait_queue_t *curr, unsigned mode, int wake_flags,
			  void *key)
	{
		return try_to_wake_up(curr->private, mode, wake_flags); // 在使用DEFINE_WAIT_FUNC的时候，
									// private被赋值为指向current，即当前进程
	}

#### 2. 等待队列的使用 ####

#### 3. 具体的使用场景 ####

#### 4. 相关背景知识  ####

##### a. 休眠(sleep)对进程来讲意味着什么？#####
以下回答摘自`LDD3`

>当一个进程被置入休眠时，它会标记为一种特殊状态并从调度器的运行队列中移走。直到某些情况下修改了这个状态，进程才会在任意CPU上调度，也即运行该进程。

其实就一句话：休眠就是某个进程停止了工作。

##### b. Linux进程状态的分类 #####
在Linux Kernel中，一个进程常见的有以下几种状态：

- `TASK_RUNNING` ---> 意味着进程处于可运行状态。这并不意味着已经实际分配了CPU。进程可能会一直等到调度器选中它。改状态确保进程可以立即运行而无需等待外部事件。

- `TASK_UNINTERRUPTIBLE` ---> 是针对等待某事件或其他资源的睡眠进程设置的。在内核发送信号给该进程表明事件已经发生时，进程状态变为`TASK_RUNNIG`，它只要调度器选中该进程即可恢复执行。


- `TASK_UNINTERRUPTIBLE` ---> 用于因内核指示而停用的睡眠进程。它们不能由外部信号唤醒，只能由内核亲自唤醒。

##### c. 如何让Linux进程休眠？ #####
其实就是改变当前process的状态，然后调用`schedule`函数，剩下的事情就交给scheduler去做行了。比如对`prepare_to_wait`的调用：
	
	prepare_to_wait(&wq, &__wait, TASK_UNINTERRUPTIBLE);

其实它做的工作是：
	
	void
	prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state)
	{
		......
		set_current_state(state);
		.....
	}

然后在`__wait_event`中会去调用`schedule`函数去创建一个调度点。
  
