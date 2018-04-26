# Lab4 Report

姚宇奇(2015011351)	计55

## 练习1

实现alloc_proc()函数，完成一个进程控制块的分配和初始化功能。

### 实现思路

主要工作为对struct proc_struct结构的初始化。该结构体成员如下所示：

```c
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list 
    list_entry_t hash_link;                     // Process hash list
};
```

其中除最后两个list_entry_t类型的成员外，其余成员均需要初始化。大部分成员的初始值设为0或NULL即可，以下三个成员需按照特殊的初值进行初始化：

```c
proc->state = PROC_UNINIT;
proc->pid = -1;
proc->cr3 = boot_cr3;
```

此外，context和name[]需要通过memset()方法将初值设为0.

### 思考题

`struct context context`的含义：进程的上下文，即部分寄存器的值。在本次实验中的作用为保存进程上下文，实现内核态的进程切换。

`struct trapframe *tf`的含义：中断帧的指针，总是指向内核栈的某个位置。在进程从用户空间跳到内核空间时，记录了进程中断前的状态。作用为当进程回到用户空间时，调整中断帧以恢复让进程继续执行的各寄存器值；同时在嵌套中断发生时，通过tf定位当前的中断帧。

## 练习2

实现do_fork()函数，为新创建的内核线程分配资源，并且复制原进程的状态。

### 实现思路

1. 分配并初始化进程控制块，通过调用alloc_proc()函数实现；
2. 分配并初始化内核栈，通过调用setup_stack()函数实现；
3. 复制原进程的内存管理信息到新进程，通过copy_mm()函数和传入的clone_flag标志实现；
4. 设置进程在内核态正常运行和调度需要的中断帧和执行上下文，通过copy_thread()函数实现；
5. 将设置好的进程控制块放入hash_list中（通过hash_proc()函数实现）和proc_list中；
6. 唤醒新线程，通过wakeup_proc()函数实现；
7. 将返回值设为子线程id。

其中，前三步的执行有可能失败，此时应当在回收对应空间之后返回错误码；另外，要将子进程的parent指定为当前进程(current)，并在进程控制块加入hash_list()之前调用get_pid()为子进程分配一个不重复的pid，同时注意增加进程集的计数(nr_process)。

### 思考题

ucore实现了给每个新fork的进程一个唯一的id。这是因为在给进程分配新的id之前，ucore通过local_intr_save()函数加了一个保护锁，暂时不允许中断，分配完成、进程插入链表、计数完成后再将锁释放，这就保证了get_pid()方法得到的pid一定是唯一的。

## 练习3

proc_run()函数的运行过程：首先关中断，然后通过load_esp0()修改TSS的esp0，通过lcr3()修改页表项，通过switch_to()函数完成上下文的切换，至此完成进程调度，再开中断。

在本实验的执行过程中，创建并运行了两个内核进程，分别是用于初始化内核中各子系统的idleproc，和初始化完成后idleproc调度的实验进程initproc。

local_intr_save(intr_flag)和local_intr_restore(intr_flag)的主要功能是屏蔽中断和重新打开中断，防止进程调度过程被中断干扰。

## 与参考答案实现的区别

本次实验中我的实现与参考答案的实现没有本质性区别，只有一些细节上实现略有不同，例如在初始化进程控制块时name数组多初始化了一位，以及在为新线程分配资源时为新线程某些属性赋值的语句位置与参考答案有差别。

## 与OS知识点的对应

对进程控制块的初始化涉及到的知识点包括进程的组成和进程控制块的组成；为新创建的内核线程分配资源的过程涉及到进程的复制、进程控制块的组织和进程的状态等知识点；分析进程切换的过程涉及到进程的调度过程、调度时进程信息的切换等知识点。不过，实验并未涉及到进程的状态切换、进程的终止与回收、调度策略等知识点。