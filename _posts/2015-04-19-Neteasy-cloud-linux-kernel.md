---
layout:     post
title:      "通过分析exevc系统调用处理过程来理解Linux内核如何装载和启动一个可执行程序"
subtitle:   "网易云课堂Linux内核分析课程"
date:       2015-04-19 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

# 通过分析exevc系统调用处理过程来理解Linux内核如何装载和启动一个可执行程序

# 前言说明
本篇为网易云课堂Linux内核分析课程的第七周作业，本次作业我们将具体来分析`exec*函数`对应的系统调用处理过程，来分析Linux内核如何来执行一个可执行程序,由于有一个在网易云课堂共同学习的朋友，代码部分是我们二人共同完成代码分析注释。

---
## 关键词：`exec`, `系统调用`，`进程`,`elf`,`可执行程序`

---
*运行环境：*

- Ubuntu 14.04 LTS x64
- gcc 4.9.2
- gdb 7.8
- vim 7.4 with vundle

---
# 过程分析
## 分析说明
在进行详细的分析之前，首先我们来总结一下**Linux内核装载执行ELF程序**的大概过程：
- 首先在用户层面，`shell`进行会调用`fork()`系统调用创建一个新进程
- 新进程调用`execve()`系统调用执行制定的`ELF文件`
- 原来的`shell`进程继续返回等待刚才启动的新进程结束，然后继续等待用户输入

以上总结中，`fork()`系统调用过程在上一次作业中，我们都很清楚，这一次我们将来详细分析`execve()`系统调用，分析方法与上一次作业相同，即结合内核代码对整个流程进行抽象分析(对有中间的繁杂细节我们可以进行选择性的忽略，以能够让我们关注中间的重要流程)，All reight，Let's rock and roll!


## 分析
`execve()`系统调用的原型如下：
```c
int execve(const char *filename, char *const argv[],
           char *const envp[]);
```
它所对应的三个参数分别是**程序文件名， 执行参数， 环境变量**,通过对内核代码的分析，我们知道`execve()`系统调用的相应入口是`sys_execve()`,在`sys_execve`之后,内核会分别调用`do_execve()`,`search_binary_handle()`,`load_elf_binary`等等，其中`do_execve()`是最主要的函数,所以接下来我们主要对他来进行具体分析

### do_execve
```c
int do_execve(struct filename *filename,
	const char __user *const __user *__argv,
	const char __user *const __user *__envp)
{
	return do_execve_common(filename, argv, envp);
}

//do_execve_common
static int do_execve_common(struct filename *filename,
				struct user_arg_ptr argv,
				struct user_arg_ptr envp)
{
	// 检查进程的数量限制
    
	// 选择最小负载的CPU，以执行新程序
	sched_exec();

	// 填充 linux_binprm结构体
	retval = prepare_binprm(bprm);

	// 拷贝文件名、命令行参数、环境变量
	retval = copy_strings_kernel(1, &bprm->filename, bprm);
	retval = copy_strings(bprm->envc, envp, bprm);
	retval = copy_strings(bprm->argc, argv, bprm);

	// 调用里面的 search_binary_handler 
	retval = exec_binprm(bprm);

	// exec执行成功

}

// exec_binprm
static int exec_binprm(struct linux_binprm *bprm)
{
	// 扫描formats链表，根据不同的文本格式，选择不同的load函数
	ret = search_binary_handler(bprm);
	// ...
	return ret;
}
```
- 如果想要了解`elf`文件格式，可以在命令行下面`man elf`，Linux手册中有参考.
- 在`do_exec()`中会调用`do_execve_common()`,这个函数的参数与`do_exec()`一模一样
- 在`do_execve_common()`中的sched_exec(),会选择一个负载最小的CPU来执行新进程，这里我们可以得知Linux内核中是做了**负载均衡**的.
- 在这段代码中间出现了变量`bprm`,这个是一个重要的结构体`struct linux_binfmt`，下面我贴出此结构体的具体定义:

```c
/*
 * This structure is used to hold the arguments that are used when loading binaries.
 */
// 内核中注释表明了这个结构体是用于保存载入二进制文件的参数.
struct linux_binprm {
	char buf[BINPRM_BUF_SIZE];
#ifdef CONFIG_MMU
	struct vm_area_struct *vma;
	unsigned long vma_pages;
#else
    //...
	unsigned interp_flags;
	unsigned interp_data;
	unsigned long loader, exec;
};
```

- 在`do_execve_common()`中的search_binary_handler(),这个函数回去搜索和匹配合适的可执行文件装载处理过程，下面这个函数的精简代码：

```c
int search_binary_handler(struct linux_binprm *bprm)
{
	// 遍历formats链表
	list_for_each_entry(fmt, &formats, lh) {
		if (!try_module_get(fmt->module))
			continue;
		read_unlock(&binfmt_lock);
		bprm->recursion_depth++;
		
		// 应用每种格式的load_binary方法
		retval = fmt->load_binary(bprm);
		read_lock(&binfmt_lock);
		put_binfmt(fmt);
		bprm->recursion_depth--;
		// ...
	}
	return retval;
}
```
- 这里需要说明的是，这里的`fmt`变量的类型是`struct linux_binfmt *`, 但是这一个类型与之前在`do_execve_common()`中的`bprm`是不一样的，具体定义如下:
- 这里的`linux_binfmt`对象包含了一个单链表，这个单链表中的第一个元素的地址存储在`formats`这个变量中
- `list_for_each_entry`依次应用`load_binary`的方法，同时我们可以看到这里会有递归调用，`bprm`会记录递归调用的深度
- 装载ELF可执行程序的`load_binary`的方法叫做`load_elf_binary`方法，下面会进行具体分析

```c
/*
 * This structure defines the functions that are used to load the binary formats that
 * linux accepts.
 */
struct linux_binfmt {
	struct list_head lh; //单链表表头
	struct module *module;
	int (*load_binary)(struct linux_binprm *);
	int (*load_shlib)(struct file *);
	int (*core_dump)(struct coredump_params *cprm);
	unsigned long min_coredump;	/* minimal dump size */
};
```

---
### load_elf_binary()
```c
static int load_elf_binary(struct linux_binprm *bprm)
{
	// ....
	struct pt_regs *regs = current_pt_regs();  // 获取当前进程的寄存器存储位置

	// 获取elf前128个字节，作为魔数
	loc->elf_ex = *((struct elfhdr *)bprm->buf);

	// 检查魔数是否匹配
	if (memcmp(loc->elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
		goto out;

	// 如果既不是可执行文件也不是动态链接程序，就错误退出
	if (loc->elf_ex.e_type != ET_EXEC && loc->elf_ex.e_type != ET_DYN)
		// 
	// 读取所有的头部信息
	// 读入程序的头部分
	retval = kernel_read(bprm->file, loc->elf_ex.e_phoff,
			     (char *)elf_phdata, size);

	// 遍历elf的程序头
	for (i = 0; i < loc->elf_ex.e_phnum; i++) {

		// 如果存在解释器头部
		if (elf_ppnt->p_type == PT_INTERP) {
			// 
			// 读入解释器名
			retval = kernel_read(bprm->file, elf_ppnt->p_offset,
					     elf_interpreter,
					     elf_ppnt->p_filesz);
	
			// 打开解释器文件
			interpreter = open_exec(elf_interpreter);

			// 读入解释器文件的头部
			retval = kernel_read(interpreter, 0, bprm->buf,
					     BINPRM_BUF_SIZE);

			// 获取解释器的头部
			loc->interp_elf_ex = *((struct elfhdr *)bprm->buf);
			break;
		}
		elf_ppnt++;
	}

	// 释放空间、删除信号、关闭带有CLOSE_ON_EXEC标志的文件
	retval = flush_old_exec(bprm);


	setup_new_exec(bprm);

	// 为进程分配用户态堆栈，并塞入参数和环境变量
	retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
				 executable_stack);
	current->mm->start_stack = bprm->p;

	// 将elf文件映射进内存
	for(i = 0, elf_ppnt = elf_phdata;
	    i < loc->elf_ex.e_phnum; i++, elf_ppnt++) {

		if (unlikely (elf_brk > elf_bss)) {
			unsigned long nbyte;
	            
			// 生成BSS
			retval = set_brk(elf_bss + load_bias,
					 elf_brk + load_bias);
			// ...
		}

		// 可执行程序
		if (loc->elf_ex.e_type == ET_EXEC || load_addr_set) {
			elf_flags |= MAP_FIXED;
		} else if (loc->elf_ex.e_type == ET_DYN) { // 动态链接库
			// ...
		}

		// 创建一个新线性区对可执行文件的数据段进行映射
		error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt,
				elf_prot, elf_flags, 0);

		}

	}

	// 加上偏移量
	loc->elf_ex.e_entry += load_bias;
	// ....


	// 创建一个新的匿名线性区，来映射程序的bss段
	retval = set_brk(elf_bss, elf_brk);

	// 如果是动态链接
	if (elf_interpreter) {
		unsigned long interp_map_addr = 0;

		// 调用一个装入动态链接程序的函数 此时elf_entry指向一个动态链接程序的入口
		elf_entry = load_elf_interp(&loc->interp_elf_ex,
					    interpreter,
					    &interp_map_addr,
					    load_bias);
		// ...
	} else {
		// elf_entry是可执行程序的入口
		elf_entry = loc->elf_ex.e_entry;
		// ....
	}

	// 修改保存在内核堆栈，但属于用户态的eip和esp
	start_thread(regs, elf_entry, bprm->p);
	retval = 0;
	// 
}
```
- 这段代码相当之长，我们做了相当大的精简,虽然对主要部分做了注释，但是为了方便我还是把主要过程阐述一边：

>* 检查`ELF的可执行文件`的有效性，比如魔数，程序头表中段(segment)的数量
>* 寻找动态链接的`.interp`段,设置动态链接路径
>* 根据`ELF可执行文件`的程序头表的描述，对ELF文件进行映射，比如代码，数据，只读数据
>* 初始化ELF进程环境
>* 将系统调用的返回地址修改为ELF可执行程序的入口点，这个入口点取决于程序的连接方式，对于静态链接的程序其入口就是e_entry,而动态链接的程序其入口是动态链接器
>* 最后调用`start_thread`,修改保存在内核堆栈，但属于用户态的`eip`和`esp`,该函数代码如下:

### start_thread
```c
void
start_thread(struct pt_regs *regs, unsigned long new_ip, unsigned long new_sp)
{
	set_user_gs(regs, 0); // 将用户态的寄存器清空
	regs->fs		= 0;
	regs->ds		= __USER_DS;
	regs->es		= __USER_DS;
	regs->ss		= __USER_DS;
	regs->cs		= __USER_CS;
	regs->ip		= new_ip; // 新进程的运行位置- 动态链接程序的入口处
	regs->sp		= new_sp; // 用户态的栈顶
	regs->flags		= X86_EFLAGS_IF;
	
	set_thread_flag(TIF_NOTIFY_RESUME);
}
```

# 总结
如你所见，执行程序的过程是一个十分复杂的过程，`exec`本质在于替换`fork()`后，根据制定的可执行文件对进程中的相应部分进行替换,最后根据连接方式的不同来设置好执行起始位置，然后开始执行进程.


---
## 参考资料
- Understanding The Linux Kernel, the 3rd edtion
- Linux内核设计与实现，第三版，Robert Love, 机械工业出版社
