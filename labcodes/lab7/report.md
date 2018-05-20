# Lab7 Report

姚宇奇(2015011351)	计55

## 练习0

需要修改的只有trap.c中的trap_dispatch()函数，将原先的sched_class_proc_tick(current)替换为run_timer_list()即可。

## 练习1

### 内核级信号量

ucore中信号量结构的成员如下：

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```

其中value为信号量计数器，>0表示共享资源的空闲数，<0表示该信号量等待队列中的进程数，=0则表示等待队列为空。信号量通过sem_init函数进行初始化（分别对value和wait_queue进行初始化），其P操作由down函数实现，V操作则由up函数实现。二者具体的实现过程如下：

- down()：先关中断，判断当前信号量value是否大于0（是否有空闲资源），如果有则value减一，开中断，直接返回。如果没有，则需要将当前进程加入等待队列，然后开中断，调用schedule()调度其他进程；如果进程被V操作唤醒，则先关中断，将进程关联的wait变量从等待队列中删除，再开中断。

- up()：同样先关中断，如果等待队列为空则直接将value加一；否则调用wakeup_wait函数将等待队列中的第一个进程唤醒，并将wait变量从队列中删除。最后开中断，返回。

  哲学家就餐流程简要描述如下：每位哲学家吃饭前通过phi_take_forks_sema函数拿起叉子，吃完后通过phi_put_forks_sema函数放下叉子。拿起叉子时先对临界区mutex进行P操作，将哲学家状态设定为饥饿，然后执行phi_test_sema函数试图得到两支叉子：如果成功得到则这位哲学家对应信号量执行V操作。然后对临界区执行V操作，对信号量执行P操作，如果之前没有得到叉子则进程进入阻塞。放下叉子时，同样先对临界区进行P操作，然后将哲学家状态设定为思考，接着通过phi_test_sema检查左右两边的人是否可以用餐，最后对临界区执行V操作。

### 用户态进程/线程信号量机制的设计

用户态使用信号量时，不需要对信号量的数据结构进行修改，但对信号量的初始化、P操作和V操作涉及到对内核资源的处理，因此都必须建立相应的系统调用完成。

异同：结构和操作过程类似，不同之处在于用户态信号量需要借助系统调用完成，以及一个用户进程可能需要使用多个系统临界区，因此需要多个内核信号量进行并发控制。

## 练习2

### 内核级条件变量

管程的结构如下：

```c
typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```

其中mutex是一个二值信号量，确保每次只有一个进程进入管程；cv为条件变量，next用于进程的同步，next_count决定睡眠的进程数量。

条件变量数据结构如下：

```c
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;
```

sem用于发出wait_cv，使得等待条件C的进程退出管程而睡眠，而发出signal_cv的进程通过sem来唤醒睡眠的进程。count则表示等待这个条件变量的睡眠的进程个数。owner表示宿主管程。

wait_cv方法和signal_cv方法在ucore中的实现分别为cond_wait()函数和cond_signal()函数。实现过程如下：

- cond_signal：如果cv.count大于0，说明当前有睡眠的进程。由于只允许一个进程在管程中执行，因此如果要唤醒进程A，则进程B自己就需要睡眠，所以将管程的next_count加一，唤醒cv.sem，并让进程B睡眠在管程的next上，然后把next_count减一。
- cond_wait：首先cv.count加一，表示有新的进程需要睡眠。如果此时管程的next_count大于0，说明此时有进程执行cond_signal后睡眠，此时应唤醒monitor.next；否则，说明此时睡眠的进程是因为互斥条件而无法进入管程的进程，因此需要唤醒monitor.mutex。最后，cv.count减一，表示等待此条件的睡眠进程数减少了1。

实现了这两个方法之后，就可以顺势用管程解决哲学家就餐问题了。本实验中需要实现的是phi_take_forks_condvar()函数和phi_put_forks_condvar()函数中的管程相关部分。具体实现逻辑也比较简单，描述如下：

- phi_take_forks_condvar：对mutex执行P操作后，将对应哲学家状态设为HUNGRY，然后通过phi_test_condvar尝试让他就餐。如果失败，则让进程睡眠，等待他对应的条件变量。
- phi_put_forks_condvar：对mutex执行P操作后，将对应哲学家状态设为THINKING，然后分别让他左右两边的人尝试就餐。

### 用户态进程/线程条件变量机制的设计

与信号量机制的设计类似，这里的设计同样不需要改变原有数据结构，与内核态的区别只在于用户态下需要通过系统调用来对条件变量进行操作。

### 能否不用基于信号量的机制实现条件变量？

可以。只要将管程相关结构中的信号量改为相应的队列，然后将管程操作中的signal和wait方法换成对于队列的原子操作即可。

## 知识点分析

实验知识点：底层支撑技术（禁用中断、定时器、等待队列），信号量的设计实现，管程和条件变量的设计实现，哲学家就餐问题的解决方案，内核级信号量/管程/条件变量的实现

OS课程知识点：同步互斥技术、信号量技术、管程与条件变量