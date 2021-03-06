# Lab5 report

姚宇奇（2015011351）	计55

## 练习0

对之前实验代码的改进：

### trap.c

idt_init()：将T_SYSCALL对应的中断描述符设置为陷阱门描述符（特权级为1），权限为用户态权限；其余描述符设计为中断门描述符（特权级为0），权限为内核态。由于Lab1时我已经采用了符合要求的实现方式，实际不需做修改。

trap_dispatch()：每100次时钟中断时不再进行输出，改为将当前进程的need_resched设为1（表示需要重新调度），同时注意检查当前进程是否为空。

### proc.c

alloc_proc()：对wait_state、cptr、yptr、optr成员进行初始化。

do_fork()：在执行alloc_proc()后将新进程的父进程设为当前进程，并检查当前进程的wait_state是否为0；在将新进程插入hash_list后用set_links()函数设置新进程的relation links，取代之前简单插入proc_list并增加进程计数的处理方法。

## 练习1

完善load_icode()函数中设置trapframe的部分，使得os能够正确加载应用程序并执行。

### 实现思路

为了使进程能够转到用户态执行，将tf_cs的值设置为USER_CS，tf_ds、tf_es和tf_ss都设置为USER_DS。tf_esp指向用户栈栈顶USTACKTOP，tf_eip则指向加载的ELF格式文件中的用户程序入口elf->e_entry，tf_eflags初始化为允许中断的状态FL_IF。

### CPU让用户态进程执行的过程

第一条进程创建完成后，ucore通过user_main-->kernel_thread-->do_fork的调用关系创建出了一个新的子进程并将其唤醒，然后借助系统调用通过kernel_execve-->do_execve-->load_icode将待执行程序的内容加载入内存中。至此用户进程的执行环境搭建完毕，中断执行IRET，由于在load_icode中设置了中断帧的eip，将切换至用户进程的第一条语句执行。

## 练习2

完善copy_range()函数的实现，使得do_fork()函数能够将当前进程（父进程）的内存空间中的合法内容拷贝给新进程（子进程）。

### 实现思路

先通过page2kva函数计算出父进程页面和子进程页面的起始地址，再调用memcpy函数做页面拷贝。最后用page_insert()函数建立起子进程页面的物理地址到线性地址的映射。

### COW机制的实现思路

在执行fork()过程时，不再直接将父进程的内容copy给子进程，而是直接将父进程的PDE赋值给子进程的PDE，并将子进程PDE的标志位设置为只读。这样，每当子进程需要写时会产生一个page fault，在此时为子进程新建一个PDE，复制原有页面的内容，并将权限设置为可写。

## 练习3

### fork

fork函数通过系统调用SYS_FORK实现进程的复制。SYS_FORK的实现过程为对do_fork()函数和wakeup_proc()函数的调用。其中do_fork()函数的实现主要在lab4中完成，主要功能为为新建立的进程分配资源；wakeup_proc()函数的功能则是将新进程的状态设置为等待。

### exec

exec函数通过系统调用SYS_EXEC完成进程的运行。SYS_EXEC的实现为do_execve()函数，它的主要工作分两步：首先做好用户态内存空间清空的准备，即将当前进程的mm设为内核空间页表，然后根据引用计数决定是否释放此进程占用的内存空间，最后将该内存管理指针设为空；然后通过load_icode()函数将应用程序执行码加载进新的用户态虚拟空间中。load_icode()的实现主要包括：建立新的mm、建立新进程的页目录表、解析ELF格式执行程序、建立物理地址与虚拟地址的映射、设置用户栈、更新虚拟内存空间、重新设置中断帧。至此，用户进程的执行环境搭建完毕，中断返回IRET将跳转至用户进程指令入口开始执行。

### wait

wait函数通过SYS_WAIT系统调用实现等待子进程以及对子进程的回收。SYS_WAIT通过调用do_wait()函数完成上述功能，具体实现为：首先判断是否等待某一特定PID的子进程，然后检查该子进程的状态，若不为PROC_ZOMBIE，则该进程尚未退出，父进程睡眠，调用schedule()以选择新的进程执行；否则，开始对子进程的回收，将子进程控制块从proc_list和hash_list中删除，并释放子进程的内核堆栈和进程控制块。

### exit

exit函数通过SYS_EXIT系统调用完成进程的退出。该系统调用执行的主要函数为do_exit()，具体实现如下：首先判断该进程是否为允许退出的用户进程，然后回收此用户进程占用的大部分用户态虚拟空间，接着唤醒父进程完成最后的回收工作；如果该进程还有子进程，需要将这些子进程的父进程设为内核线程init；最后执行schedule()函数选择新的进程执行。

以上函数影响进程执行状态的方式均为通过执行对应系统调用产生中断，如果此时进程的need_resched值为1，则ucore会对其进行调度。

用户态进程执行状态生命周期图：

PROC_UNINIT -- (do_fork, do_execve) --> PROC_RUNNABLE -- (do_wait/do_sleep) --> PROC_SLEEPING --

​                                                                                  ↑         ↓                                                                                      ↓
​                                                                                  ↑          → (do_exit) --> PROC_ZOMBIE                                  ↓
​                                                                                   ← ----------------------- (wakeup_proc) --------------------------  ←

## 知识点对应

### 练习1

实验知识点：ELF格式程序的加载与解析，中断帧的设置，IRET指令

课程知识点：ELF格式，中断处理机制

### 练习2

实验知识点：ucore中虚拟内存管理的实现，Copy On Write机制

课程知识点：虚拟内存管理，fork()方法的实现

### 练习3

实验知识点：ucore的系统调用，内联汇编，ucore的进程调度机制

课程知识点：进程的状态切换，进程调度，系统调用