# Lab6 Report

姚宇奇(2015011351)	计55

## 练习0

对Lab5代码的改进：

### proc.c

alloc_proc()函数：对新增的rq, run_link, time_slice, lab6_run_pool, lab6_stride, lab6_priority进行初始化。将rq置为NULL，用list_init()函数对链表run_link初始化，将lab6_run_pool的left, right, parent成员指针都设为NULL，其余成员置为0即可。

### trap.c

trap_dispatch()函数：根据提示，将代码更新为如下形式：

```c
ticks ++;
assert(current != NULL);
sched_class_proc_tick(current);
break;
```

## 练习1

sched_class中函数指针的使用如下所示：

```c
struct sched_class default_sched_class = {
    .name = "RR_scheduler",
    .init = RR_init,
    .enqueue = RR_enqueue,
    .dequeue = RR_dequeue,
    .pick_next = RR_pick_next,
    .proc_tick = RR_proc_tick,
};
```

这个结构通过函数指针的方式为进程调度算法提供了切换接口。以下分别描述这些函数的功能。

### RR_init

负责进程队列的初始化。

### RR_enqueue

控制进程入队。首先将进程插入队列末尾；然后检查进程的时间片：如果时间片为0，说明进程当前执行的时间片已经用完（或大于max_time_slice，说明时间片异常），需要重置为max_time_slice，等到下一次有机会运行时再继续执行。最后将该进程的rq成员设置为当前队列，并将队列进程数加1.

### RR_dequeue

控制进程出队。将该进程从队列中删除，并将进程数量减去1。

### RR_pick_next

选取下一个就绪进程。选取队列头部的元素，返回对应的进程控制块指针。

### RR_proc_tick

每个timer到时的时候调用此函数，将时间片减1。如果时间片归零，则将need_resched置为1，等到下一个中断到来时进行重新调度，将其放入队列尾，再从队列头取出新的进程执行。

因此，RR调度算法的调度过程为：维护一个就绪进程的队列，每当timer到时，递减当前进程时间片，将其移到队列尾部，且重置它的时间片，然后从队列头取出一个新的进程继续执行。

### “多级反馈队列调度算法”的实现思路

维护多个优先级不同的就绪进程队列，优先级越低时间片(max_time_slice)越大。调度时，优先调度高优先级队列中的进程，调度方法同RR算法；如果高优先级队列中没有可调度的进程，则调度低优先级队列。高优先级队列中的进程如果在当前时间片没有完成，则调入下一级队列中，获得更大的时间片；以此类推，直至完成。新进程加入时先加入最高优先级的队列；此时如果低优先级队列中的进程正在执行，则当前时间片运行完成后马上执行新的进程（进程抢占）。

## 练习2

Stride Scheduling算法的特点在于，并非像RR算法那样平均分配时间资源，而是根据进程的优先级(lab6_priority)分配时间资源。为此，可以利用ucore实现的优先级队列skew_heap这一数据结构来帮助我们的实现。

需要实现五个函数：stride_init(), stride_enqueue(), stride_dequeue(), stride_pick_next(), stride_proc_tick()。

### stride_init()

与RR算法类似，需要对run pool进行初始化。

### stride_enqueue()

注意到USE_SKEW_HEAP的宏定义，实现两个版本的入队方法。定义了skew_heap的情况下，通过skew_heap_insert()函数进行入队；否则用与RR相同的方法插入list中。然后处理方式与RR相同，初始化时间片并更新进程总数。

### stride_dequeue()

如果使用skew_heap()，则使用skew_heap_remove()函数进行出队操作；否则，采用与RR相同的方法从list中删除。然后更新进程总数。

### stride_pick_next()

如果使用skew_heap，则在确认run pool不为空的情况下，将其转化为proc_struct类型作为下一个进程；否则遍历list，找到stride最小（优先级最高）的进程。如果下一个进程的stride为0，则将stride增加一个最大步长；否则增加一个最大步长乘以优先级的倒数，以实现根据优先级分配时间资源。

### stride_proc_tick()

同RR算法。

其中，最大步长(BIG_STRIDE)根据系统能表示的最大的带符号整数大小来确定。由于Stride采用无符号32位整数表示，且需要通过两个stride的差的符号来比较不同的stride，因此BIG_STRIDE应为带符号32位整数的最大值，即$2^{31}-1$。