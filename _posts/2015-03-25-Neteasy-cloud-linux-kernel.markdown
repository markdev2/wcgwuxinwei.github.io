---
layout:     post
title:      "通过从代码层面分析Linux内核启动来探知操作系统的启动过程"
subtitle:   "网易云课堂`Linux内核分析`课程的第三周作业"
date:       2015-03-25 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

#通过从代码层面分析Linux内核启动来探知操作系统的启动过程
##前言说明
本篇为网易云课堂`Linux内核分析`课程的第三周作业，我将围绕Linux 3.18的内核中的`start_kernel`到`init`进程启动过程来深入探知操作系统的启动，文中的代码来自`Linux Kernel Organization`的`3.18.9`内核源码

##本篇关键词：`init进程`，`idle进程`，`Linux内核启动`

##分析
##分析说明
- 分析过程将把主要精力放在关键代码的分析上，代码分析的方式我是采用注释说明的方法，这样比较简洁直观，对于一些关键过程我会在代码后面采用图文说明的方式。
- 文中我已经将3.18内核代码编译，而且加入debug调试信息，由于这里的过程在课堂上已经有详细讲解，这个过程我就不在赘述。

---

```c
/* star_kernel是linux内核入口 */
asmlinkage __visible void __init start_kernel(void) 
{
	char *command_line;
	char *after_dashes;

	/*
	 * Need to run as early as possible, to initialize the
	 * lockdep hash:
	 */
	lockdep_init();
	set_task_stack_end_magic(&init_task);
	smp_setup_processor_id();
	debug_objects_early_init();
    ...
    
	boot_cpu_init();
	page_address_init();
	pr_notice("%s", linux_banner);
	setup_arch(&command_line);
	mm_init_cpumask(&init_mm);
	setup_command_line(command_line);
	setup_nr_cpu_ids();
	setup_per_cpu_areas();
	smp_prepare_boot_cpu();	/* arch-specific boot-cpu hooks */

    ...
	build_all_zonelists(NULL, NULL);
	page_alloc_init();
    ...
	setup_log_buf(0);
	pidhash_init();
	vfs_caches_init_early();
	sort_main_extable();
	trap_init();
	mm_init();
    ...
    /* Do the rest non-__init'ed, we're now alive */
	rest_init();
}

```
- `init/main.c`
- 以上为`start_kernel`函数的一个代码片段，在该函数之前的执行都是汇编，以C语言程序的思维来看，`start_kernel`就是整个Linux内核的“main”函数，即整个Linux内核的入口函数，在`start_kernel`中，开始有一个`set_task_stack_end_magic(&init_task)`,这个函数中间的形参`init_task`,通过寻找在`init/init_task.h`中找到了`struct task_struct init_task = INIT_TASK(init_task)`（struct task中保存进程的相关信息，类似PCB）,经过初始化`init_task`后，静态构造进程，这是Linux第一次拥有了进程，这就是后来的idle进程（pid为0），**从`start_kernel`之前的汇编代码到`start_kernel`执行，这里都会纳入idle进程的上下文**（之前的汇编代码就是为了idle进程的执行做准备）。
- 最后`rest_init()`标志着Linux内核初始化完成，在`rest_init()`中开始产生**第一个真正意义上的进程**，也就是init进程（即进程号为1的进程,其他所有用户进程的祖先进程），接下来就对`rest_init()`部分做详细分析

---

```c
static noinline void __init_refok rest_init(void)
{
	int pid;

	rcu_scheduler_starting();
	/*
	 * We need to spawn init first so that it obtains pid 1, however
	 * the init task will end up wanting to create kthreads, which, if
	 * we schedule it before we create kthreadd, will OOPS.
	 */
	kernel_thread(kernel_init, NULL, CLONE_FS);
	numa_default_policy();
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
	complete(&kthreadd_done);

	/*
	 * The boot idle thread must execute schedule()
	 * at least once to get things moving:
	 */
	init_idle_bootup_task(current);
	schedule_preempt_disabled();
	/* Call into cpu_idle with preempt disabled */
	cpu_startup_entry(CPUHP_ONLINE);
}
```
- `kernel/fork`
- `kernel_thread(kernel_init, NULL, CLONE_FS)`，这里通过这个函数创建了init进程，该函数具体代码如下：
```c
pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
	return do_fork(flags|CLONE_VM|CLONE_UNTRACED, (unsigned long)fn,
		(unsigned long)arg, NULL, NULL);
}
```
第一次参数为注册一个回调函数，`kernel_init`这个回调函数，`do_fork`是创建一个新的进程， 在此之中会为创建init进程进行各种工作，如初始化运行堆栈，调用相应的回掉函数等，通过回调`kernel_init`可以创建init进程，接下来具体分析下`kernel_init`

---
```c
static int __ref kernel_init(void *unused)
{
	int ret;

	kernel_init_freeable();
	/* need to finish all async __init code before freeing the memory */
	async_synchronize_full();
	free_initmem();
	mark_rodata_ro();
	system_state = SYSTEM_RUNNING;
	numa_default_policy();

	flush_delayed_fput();

	if (ramdisk_execute_command) {
		ret = run_init_process(ramdisk_execute_command);
		if (!ret)
			return 0;
		pr_err("Failed to execute %s (error %d)\n",
		       ramdisk_execute_command, ret);
	}

	/*
	 * We try each of these until one succeeds.
	 *
	 * The Bourne shell can be used instead of init if we are
	 * trying to recover a really broken machine.
	 */
	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		pr_err("Failed to execute %s (error %d).  Attempting defaults...\n",
			execute_command, ret);
	}
	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;

	panic("No working init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/init.txt for guidance.");
}
```
- `init/main.c`
- 在`kernel_init`中我们重点关注以下代码，在这段代码中实际上是通过`run_init_process`来执行`/sbin/init`,通过中断向量0x80（system_call）来从内核发起系统调用，如果`/sbin/init`调用失败，则会继续调用接下来的文件`/etc/init`,`/bin/init`,`/bin/sh`，
```c
...
if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;
...
```

---
接下来我们回到`rest_init`的代码片段，`rest_init`执行完后，idle进程已经结束了他的使命，开始成为一个真正的`idle`进程，即真正的空闲进程，从这里开始内核的初始化真正结束了，用户态的阶段开始了
```c
//rest_init
...省略
/*
 * The boot idle thread must execute schedule()
 * at least once to get things moving:
 */
init_idle_bootup_task(current);
schedule_preempt_disabled();
/* Call into cpu_idle with preempt disabled */
cpu_startup_entry(CPUHP_ONLINE);
```
- 这里的`cpu_startup_entry(CPUHP_ONLINE)`中的代码片段
```c
void cpu_startup_entry(enum cpuhp_state state)
{
    ...代码省略
    arch_cpu_idle_prepare();
    cpu_idle_loop();
}

static void cpu_idle_loop(void)
{
	while (1) {
		...代码省略
		__current_set_polling();
		tick_nohz_idle_enter();
        ...
		arch_cpu_idle_enter();
		arch_cpu_idle_exit();
		...
		schedule_preempt_disabled();
		...
```
- 我们在这里看到一个`cpu_idle_loop()`，这里其中是一个死循环，而且从中可以看到CPU不断地进入idle状态不断的推出idle状态。
- 从这里我们可以得到一个这样的结果并总结，**idle进程是一个唯一的内核态进程，init进程是第一个用户态进程**，idle进程在内核初始化的时候的工作就是创建init进程。

##我对Linux系统启动过程的一点理解
首先，启动计算机，载入汇编代码，直到`start_kernel`执行的这个阶段,`idle`进程（0号进程）就是从这个时间段产生的，这个阶段为`idle`执行上下文做准备，`start_kernel`的`init_task`就是`idle`进程（此时还没有Linux进程，仅仅是模拟的一个进程），然后在`rest_init`初始化并产生init进程(1号进程)，整个操作系统开始从内核态向用户态转换。


##署名信息
    吴欣伟 原创作品转载请注明出处 《Linux内核分析》MOOC课程: http://mooc.study.163.com/course/USTC-1000029000

