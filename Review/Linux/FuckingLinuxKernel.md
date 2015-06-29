#####About Linux
+ Linux is a modular, UNIX-like monolithic kernel
+ Kernel is the heart of the OS that executes with special hardware permissions (kernel mode)
+ "Core kelnel" provides framework, data structures, support for drivers, modules, subsystems.

#####About Kernel
+ System monitor
+ Controls and mediates access to hardware
+ Implements and supports fundamental abstractions
	- Processes, files, devices, etc.
+ Schedules/ allocates system resources:
	- Memory, CPU, disk, etc.
+ Enforces security and protection.
+ Responds to user request for service (system calls).
+ Etc...

#####Kernel Design Goals
+ Performance: efficiency, speed
+ Stability: robustness, resilience
+ Capability: features, flexibility, compatability
+ Security, protection
	- Protect users from each other & system from bad users
+ Portability
+ Extensibility

#####Architecture Approaches
+ Monolithic
+ Layered
+ Modularized
+ Micro-kernel
+ Virtual machine

#####Advantages & Disadvantages of the /proc File System
+ Advantages	
	- Coherent,intuitive interface to the kernel
	- Great for tweaking and collecting status info
	- Easy to use and program for
+ Disadvantages
	- Certain amount of overhead, must use fs calls
	- User can possibly cause system instability

#####Compile the Linux Kernel
+ cp, cd, tar -xzvf, make clean, make mrproper
+ make menuconfig/xconfig/oldconfig/config
+ make/ make modules_install/ make install
+ cd /boot & mkinitrd -o initrd.img-[kernel version]
+ reboot

#####Booting
+ BIOS (Basic Input/Output System)
+ MBR (Master Boot Record)
+ Bootloader: GRUB/LILO

#####What makes kernel programming different?
+ OS management
	- Process management, memory management
	- File systems
+ Types of devices
+ Loaded as modules or static in the kernel
+ Challenges
	- Portability
	- IPC
	- Hardware management
	- Interface Stability

#####Module - Advantages and disadvantages
+ Advantages
	- Allow the dynamic insertion and removal of code from the kernel at run-time
	- Save memory cost
+ Minor Disadvantage
	- Fragmentation Pelnalty -> decrease memory performance

#####Module related command
+ find . -name "*.ko"
+ lsmod / insmod / rmmod / modinfo

#####About /proc
+ A pseudo file system
+ Realtime, resides in the virtual memory
+ Tracks the processes running on the machine and the state of the system
+ A new /proc file system is created every time Linux reboots
+ Highly dynamic. The size of the proc directory is 0 and the last time of modification is the last bootup time

#####Modules VS Programs
+ Modules
	- module_init() and module_exit functions
	- printk() / Kernel Space / Written in C (typically)
+ Programs
	- main()
	- printf() / User Space / C,C++,Java,etc.

#####Process, LWP and Threads
+ Process
	- an instance of a program in execution
+ (User) Thread
	- an execution flow of the process
+ Lightweight process (LWP)
	- used to offer better support for multithreaded applications
	- share resources: address space, open files...
	- associate a LWP with each thread

#####About Process
+ Descriptor: *task_struct* 
+ Process descriptor pointers : 32-bit
+ Process ID: 16-bit
	- Linux associates different PID with each process or LWP
	- Threads in the same group have a common PID
+ Parenthood Relationships among processes
	- Process 0 and 1: created by the kernel
	- Process 1 (init): the ancestor of all processes

#####Scheduling Policy
+ Based on time-sharing
	- Time slice
+ Based on priority ranking
	- Dynamic
+ Classification of processes
	- Interactive processes
	- Batch processeses
	- Real-time processes

#####Linux (user) processes are preemptive
+ When a new process has higher priority than the current
+ When its time quantum expires

#####How long must a quantum last?
+ Neither too long nor too short
	- Too short: overhead for process switch
	- Too long: processes no longer appear to executed concurrently
+ Always a compromise
	- Rule of Thumb: to choose a duration as long as possible, while keeping good system response time

#####The Scheduling Algorithms
+ Example: CFS
	- The same virtual runtime for each task. / Increasing the priority for sleeping task.
	- Red-black tree

#####Kernel Control Paths
+ A sequence of instructions executed in kernel mode on behalf of current process
+ Interrupts or exceptions
+ Lighter than a process (less context)

#####Kernel Preemption
+ Reduce the dispatch latency of the user mode processes
	- Delay between the time they become runnable and the time they actually begin running
+ The kernel can be preempted only when it is executing an exception handler (in particular a system call) and the kernel preemption has not been explicitly disabled

#####When Synchronization is Necessary
+ A race condition can occur when the outcome of a computation depends on how two or more interleaved kernel control paths are nested
+ To identify and protect the critical regions in exception handlers, interrupt handlers, deferrable functions and kernel threads

#####When Synchronization is not Necessary
+ The same interrupt cannot occur until the handler terminates
+ Interrupt handlers and softirqs are non-preemptable, non-blocking
+ A kernel control path performing iterrupt handling cannot be interrupted by a kernel control path executing a derrable function or a system call service routine
+ Softirqs cannot be interleaved

#####Synchronization Primitives
+ Per-CPU variables
+ Atomic operation
+ Memory Barriers
+ Spin Locks
+ Seqlock
+ Read-copy update (RCU)
+ Semaphores
+ Local interrupt disabling
+ Local softirq disabling

#####Symmetric Multiprocessing (SMP)
+ Kernel can execute on any processor
	- Allowing portion of the kernel to execute in parallel
+ Typically each processor does self-scheduling from the pool of available process or threads
+ Multiprocessor OS Design Considerations
	- Simultaneous concurrent processes or threads
	- Scheduling
	- Synchronization
	- Memory Management
	- Reliability and Fault Tolerance
+ Non-uniform memory access (NUMA)
+ SMP Scheduling
	- Time sharing
	- Space sharing
	- WTF the above are?
+ Synchronization Probelm
+ Linux对SMP的支持：
	- 启动过程：BSP(Bootstrap Processor)负责操作系统的启动，在启动的最后阶段，BSP通过IPI(处理器间中断)激活各个AP(Application Processor，即应用CPU)，在系统的正常运行过程中，BSP和AP基本上是无差别的。
	- 进程调度：与UP系统的主要差别是执行进行切换后，被换下的进程有可能会切换到其他CPU上继续运行。在计算优先权时，如果进程上次运行的CPU也是当前CPU，则会适当提高优先权，这样可以更有效地利用Cache。
	- 中断系统：为了支持SMP，在硬件上需要APIC(高级可编程中断控制器)中断控制系统。Linux定义了各种IPI的中断向量以及传送IPI的函数。

####Application Components
+ Activities
	- visual user interface focused on a single thing a user can do
	- basic component of most applications
	- each is a subclass of the base Activity
+ View
	- Each activity has a default window to draw in 
	- The content of a window is a view or a group of views
+ Service
	- Does not have a visual interface
	- runs in the background indefinitely
- Broadcast Receiver
	- Receive and react to broadcast announcement
+ Content Provider
	- makes some of the application data avaliable to other applications
	- it's the only way to tranfer data between apps in Android
+ Intent
	+ an intent object with a message content
	+ A/S/B are started by intent.
