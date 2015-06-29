---
####Objective

Add a feature to the linux kernel, which records the total times a process is scheduled to be executed on CPU.

<!--more-->

####Basic Idea
Add a counter varible to the struct <b>task_struct</b>, init the counter varible to 0 when a process is created/forked. Every time the process is scheduled to be executed, increase the counter and write the number to a proc file.

####Details of how to modify the kernel

#####0. Environment
+ VMware Fusion 7
+ Ubuntu 12.04.5
+ Kernel Version 3.18.8

#####1. /include/linux/sched.h
######line#1234
```c++
	struct task_struct{
	+	unsigned long ctx;
		...
	}
```

#####2. /kernel/fork.c
######line#1622
```c++
	long do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr){
		  
		  	...
		  
		  	p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace);

		+	p->ctx = 0;
		
		  	...
		  
		  }
```
		  
#####3. /kernel/sched/core.c
######line#2864
```c++
	asmlinkage __visible void __sched schedule(void)
	{
		struct task_struct *tsk = current;
	
	+	tsk->ctx = tsk->ctx + 1;
	
		sched_submit_work(tsk);
		__schedule();
	}
```
	
#####4. /fs/proc/base.c
######line #319
```c++
	+	/*
	+	 * Provides /proc/PID/ctx
	+	 */
	+	static int proc_pid_ctx(struct seq_file *m, struct pid_namespace *ns,
	+				      struct pid *pid, struct task_struct *task)
	+	{
	+		return seq_printf(m, "%llu\n", (unsigned long long)task->ctx);
	+	}
```
	
######line #2603
```c++
	+	ONE("ctx",  S_IRUGO, proc_pid_ctx),
```
	
######line #2949
```c++
	+	ONE("ctx",  S_IRUGO, proc_pid_ctx),
```

#####5. Compile

####Results
#####1.Test Code
```C
	// block.c
	#include <stdio.h>
	int main(){
		while(1) getchar();
		return 0;
	}
```
```bash
	gcc block.c -o block
	./block
```
#####2.Find the process id
```bash
	ps -e | grep block
```
#####3.Observe the counter
```bash
	cd /proc/[PID]
	cat ctx
```
#####4.Increse the counter
Type anything to the block process to make it scheduled to be executed and check the content of ctx file again.
#####5.End
```bash
	kill [PID]
```
