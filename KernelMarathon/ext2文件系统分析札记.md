### ext2文件系统分析札记###
---------------------------

##### PS：本文都是在`linux-2.6.35`完成，而对其他linux release版本的分析其实大同小异

1.ext2文件系统可以作为一个ko文件，然后动态加载和卸载模块。所以，当我们加载ext这个ko的时候，其入口和出口分别是：


		//---> fs/ext2/super.c
		static int __init init_ext2_fs(void)
		{
			int err = init_ext2_xattr();
			if (err)
				return err;
			err = init_inodecache(); // 在SLAB上创建ext2_inode_info类型的cache：ext2_inode_cachep
			if (err)
				goto out1;
			err = reister_filesystem(&ext2_fs_type); // 向linux注册ext2文件系统
			if (err)
				goto out;
			return 0;
		out:
			destroy_inodecache();
		out1:
			exit_ext2_xattr();
			return err;
		}

		static void __exit exit_ext2_fs(void)
		{
			unregister_filesystem(&ext2_fs_type); 
			destroy_inodecache();
			exit_ext2_xattr();
		}
		
		module_init(init_ext2_fs) // 加载模块的入口函数
		module_init(exit_ext2_fs) // 卸载模块的出口函数

当我们打开`register_filesystem`一看，你会发现这个函数非常简单：

	
		//---> fs/filesystem.c
		int register_filesystem(struct file_system_type *fs)
		{
			int res = 0;
			struct file_system_type **p;
			
			BUG_ON(strchr(fs->name, '.'));	// 文件系统名不能有'.'这个字符，否则会BUG_ON
			if (fs->next)
				return -EBUSY;	// 这是一个细节，fs的下一个指针一定要为NULL，否则会出现一个漏洞
				
			// 初始化file_system_type的链表头，这个就是嵌入到file_system_type的struct list_head数据结构
			INIT_LIST_HEAD(&fs->fs_supers); 
			
			// 因为需要修改到全局变量p，p将指向全局文件系统类型链表，所以此处需要加锁访问
			write_lock(&file_systems_lock);
			
			// find_filesytem通过文件系统名称和长度来遍历全局文件系统链表，
			// 如果找到就返回对应的file_system_type结构，否则返回最后一个文件系统类型结构的next指针
			p = find_filesystem(fs->name, strlen(fs->name)); 

			if (*p)	// 如果该指针非NULL，则说明该文件系统已经被注册过
				res = -EBUSY;
			else	// 如果该指针为NULL，则将其指向当前欲注册的file_system_type类型
				*p = fs;
			
			// critical section结束
			write_unlock(&file_systems_lock);
			return res;
		}
		
以上，通过`register_filesystem`之后，ext2对应的file_system_type结构会被串联到全局文件系统类型链表的末尾。这里我们还要思考一个问题：我们只是简单地将文件系统类型结构关联到linux中而没有安装相应的function pointer，那么当对ext2文件系统进行文件操作的时候，如何能去调用ext2对应的函数呢？答案是：**该过程发生在`mount`系统调用中**，请见下节分析。


2.ext2文件系统的mount过程。

我们知道，mount是linux的一个标准的system call，其实我们可以用以下一个小技巧来review一下linux中system call：

	$ grep -r -n SYSCALL_DEFINE * | grep -E ^[^arch] | grep -E ^[^scripts] > system_call.txt
	
由于我们不关注特定平台的system call，所以我们排除了一些干扰选项。强大的grep和正则又一次帮了我们的忙，让我们瞬间理清了linux(注意是`2.6.35`)中大致的系统调用 (我就不黑Windows下要用什么高效方式才能做到这点，maybe是Google或度娘@@)。如果你是想看x86架构下完整的系统调用，可以参考`arch/x86/kernel/syscall_table_32.S`。很明显，在x86下有337号系统调用，而我们提取出来的系统调用则有318个(`wc -l system_call.txt`)，还有一些被遗漏了，不过没关系，我们的重点非特定平台的系统调用。我们在执行一句：

	$ cat system_call.txt | grep -E ^fs | wc -l
	
输出结果是140，也就是说在文件系统中，存在着140个系统调用，占了我们刚刚统计出来318个系统调用的44%！可见，绝大多数系统调用都是文件系统方面的系统调用(其次是kernel目录下的系统调用，有128个)，这更加说明了，文件系统是一个多么重要的子系统！

我们接下来要讨论的mount系统调用，其原型是：

	SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name,
					char __user *, type, unsigned long, flags, void __user*, data)


通过函数原型可以看出来，当你要将执行moun动作的时候，至少需要一个dev_name，dir_name，type和flags。意思就是说将某个设备文件跟某个目录(挂载点)用某种类型(type)的文件系统以某种模式(flags)联系在一起。比如我们可以通过以下这个小实验，来测试文件系统：

	$ dd if=/dev/zero of=/dev/ext2_test_dev bs=1024 count=1024

这样便创建了一个大小为1024 * 1024 = 1048576 bytes的全为0的binary文件。接着，我们用`mkfs.ext2`工具来创建ext2文件系统：

	$ mkfs.ext2 /dev/ext2_test_dev

这个我们便将`ext2_test_dev`这个文件格式化成了ext2格式。最后我们在进行挂载的动作：

	$ mount -t ext2 /dev/ext2_test_dev /mnt/ext2_test_dir 

这样我们就可以通过ext2的方式来读写`ext2_test_dev`这个块设备。当你想要删除这个连接时候，你可以进行：

	$ umount /mnt/ext2_test_dir

好，言归正传，让我们来看看mount的过程，整个系统调用可以简写成：

	
	//---> fs/namespace.c	
	SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name,
					char __user *, type, unsigned long, flags, void __user *, data)
	{
		......
		// 这个api传进了三个user space字符串，第一步我们必须将其转换到kernel space来
		// dev_name 和 type用copy_mount_string()
		// dir_name 用 getname()
		// data 用 copy_mount_options()
		ret = copy_mount_string(type, &kernel_type);
		......
		kernel_dir = getname(dir_name);
		......
		ret = copy_mount_string(dev_name, &kernel_dev);
		......
		ret = copy_mount_options(data, &data_page);
		......
		// do_mount才是真正干活的函数
		ret = do_mount(kernel_dev, kernel_dir, kernel_type, flags, (void *)data_page);
		......
	}

让我们怀着紧张而又刺激的心情进入`do_mount`函数：

	//---> fs/namespace.c
	long do_mount(char *dev_name, char *dir_name, char *type_page, 
				unsigned long flags, void *data_page)
	{
		......
		// 获取挂载点 mountpoint
		// 根据dir_name将相应的信息填入struct path中
		retval = kern_path(dir_name, LOOKUP_FOLLOW, &path);
		......
		// 不管这个
		retval = security_sb_mount(dev_name, &path, type_page, flags, data_page);

		......
		// 从flags中分离出各种模式并合并到mnt_flags
		if (!(flags & MS_NOATIME))
			mnt_flags |= MNT_RELAIME;

		if (flags & MS_NOSUID)
			mnt_flags |= MNT_NOSUID;
		......

		// 接下来就根据flags中不同的模式来执行不同的mount函数
		if (flags & MS_REMOUNT)
			retval = do_remount(&path, flags & ~MS_REMOUNT, mnt_flags, data_page);
		else if (flags & MS_BIND)
			retval = do_loopback(&path, dev_name, flags & MS_REC);
		else if (flags & (MS_SHARED | MS_PRIVATE | MS_SLAVE | MS_UNBINDABLE))
			retval = do_move_mount(&path, dev_name);
		else
			retval = do_new_mount(&path, type_page, flags, mnt_flags, dev_name, data_page);
		......
	}
	
看到这么多不同的mount，我们就挑`do_new_mount`来介绍。这个函数是：

	//---> fs/namespace.c
	static int do_new_mount(struct path *path, char *type, int flags,
				int mnt_flags, char *name, void *data)
	{
		struct vfsmount *mnt;
		......
		lock_kernel();		// 传说中的大内核锁呀
		mnt = do_kern_mount（type, flags, name, data);
		unlock_kernel();	// 传说中的大内核锁呀
		......
		return do_add_mount(mnt, path, mnt_flags, NULL);
	}
	
很明显，这里有两个很重要的函数，`do_kern_mount`和`do_add_mount`。先看`do_kern_mount`：
	
	//---> fs/super.c
	struct vfsmount *
	do_kern_mount(const char *fstype, int flags, const char *name, void *data) // name是dev_name
	{
		struct file_system_type *type = get_fs_type(fstype); // 获取ext2的file_system_type类型
		struct vfsmount *mnt;
		......
		mnt = vfs_kern_mount(type, flags, name, data);
		......
	}

最为关键的函数出来了：`vfs_kern_mount`，如下所示：
	
	//---> fs/super.c
	struct vfsmount *
	vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
	{
		struct vfsmount *mnt;
		......
		mnt = alloc_vfsmnt(name); // name是dev_name
		......
		error = type->get_sb(type, flags, name, data, mnt); // 终于看到call 到ext2的get_sb函数了！！！柳暗花明！！！
		......												// 让我们直接跳到section3来将get_sb吧
	}

3.我们从section1可以知道，当我们加载ext2文件系统的时候，已经在全局的文件系统类型结构的链表中添加了ext2的文件系统类型结构，现在就让我们看看这个`file_system_type`是一个什么东西，其定义如下所示：
	
	//---> inclucde/linux/fs.h
	struct file_system_type {
		const char *name;	// 文件系统名称
		int fs_flags;
		int (*get_sb)(struct file_system_type *, int, const char *, void *, struct vfsmount *); // 超级块构造函数
		void (*kill_sb)(struct super_block *); // 超级块析构函数
		struct module *owner; // 通常是THIS_MODULE
		struct file_system_type *next;  // 指向下一个文件系统的file_system_type，用来串联全局的文件系统类型链表
		struct list_head fs_supers;	    // 用来串联进全局超级块链表的struct list_head

		// 下面这些就暂时不去管了
		struct lock_class_key s_lock_key
		struct lock_class_key s_umount_key;
		struct lock_class_key s_vfs_rename_key;

		struct lock_class_key i_lock_key;
		struct lock_class_key i_mutex_key;
		struct lock_class_key i_mutex_dir_key;
		struct lock_class_key i_alloc_sem_key;	
	};

在ext2中，这个结构会被配置成：
	
	//---> fs/ext2/super.c
	static struct file_system_type ext2_fs_type = {
		.owner = THIS_MODULE,
		.name  = "ext2",
		.get_sb = ext2_get_sb,
		.kill_sb = kill_block_super,
		.fs_flags = FS_REQUIRES_DEV,
	};

当我们执行mount系统调用的时候，会先去调用`do_kern_mount`，然后调用`vfs_kern_mount`，接着在`vfs_kern_mount`中会去get一个当前文件系统类型指针，然后再执行这个结构里面的`get_sb`函数来创建超级块。在VFS架构中，每种文件系统类型都会对应一个超级块结构，该结构包含了各种跟文件系统类型相关的数据。暂时先不展开来讲。让我们来看看`ext2_get_sb`函数：

	//---> fs/ext2/super.c
	static int ext2_get_sb(struct file_system_type *fs_type, int flags, const char *dev_name,
							void *data, struct vfsmount *mnt）
	{
		return get_sb_bdev(fs_type, flags, dev_name, data, ext2_fill_super, mnt);
	}

而`get_sb_bdev`的大体逻辑是：
	
	int get_sb_bdev(struct file_syste_type *fs_type,
			int flags, const char *dev_name, void *data,
			int (*fill_super)(struct super_block *, void *, int),
			struct vfsmount *mnt)
	{
		struct block_device *bdev;
		struct super_block *s;
		......
		bdev = open_bdev_exclusive(dev_name, mode, fs_type); // 获取一个block_device设备
		......
		s = sget(fs_type, test_bdev_super, set_bdev_super, bdev); // 获取超级块
		......
		if (s->s_root) { // 如果挂载点的dentry存在
			......
		} else {		 // 如果挂载点的dentry不存在
			......
			// 注意，这里将会call到ext2_fill_super函数
			error = fill_super(s, data, flags & MS_SILENT ? 1 : 0);
			......		
		}
		
		// 这个函数只是简单地将超级块对象和挂载点目录结构填入mnt结构中
		// mnt->mnt_sb = sb;
		// mnt->mnt_root = dget(sb->s_root);
		simple_set_mnt(mnt, s); 
		......	
	}
