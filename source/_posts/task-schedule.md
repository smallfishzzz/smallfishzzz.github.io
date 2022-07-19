---
title: 进程管理 -- 调度
date: 2022-07-12 14:55:54
tags: 
  - linux内核 
  - 调度
  - schedule
categories:
  - linux 内核代码阅读
---



``` 
 ____       _              _       _      
/ ___|  ___| |__   ___  __| |_   _| | ___ 
\___ \ / __| '_ \ / _ \/ _` | | | | |/ _ \
 ___) | (__| | | |  __/ (_| | |_| | |  __/
|____/ \___|_| |_|\___|\__,_|\__,_|_|\___|

```

# 一、什么是内核调度

 在2.6内核之前Linux采用传统的UNIX调度算法。由于没有考虑到SMP，所以SMP不支持。从2.6内核之后调度器再一次的大的改动。2.6.23的版本中，完全公平调度器（CFS）最终成为Linux调度算法。Linux的进程调度基于调度类。每个调度类都有一个特定优先级。每一个不同的调度类，都有不同的调度算法，就是为了满足不同的需要。这表示对CPU的占用时间由操作系统决定的，具体为操作系统中的调度器。调度器决定了什么时候停止一个进程以便让其他进程有机会运行，同时挑选出一个其他的进程开始运行。且内核必须提供一种方法，让各个进程之间能够公平的使用CPU，并且还要考虑到不同场景下不同任务的优先级。

 schedule();函数就是内核使用最频繁的调用函数之一，也是理解调度操作的起点，该函数定义位置在`kernel/sched/core.c`
下面我们暂时先忽略实时进程，只考虑完全公平调度CFS。

<!--more-->

# 二、调度器的组成部分

## 2种调度场景

linux内核中有2种调度场景，一种为主动调度，第二种为周期性调度。
可以用两种方法来激活调度

    一种是直接的, 比如进程打算睡眠或出于其他原因放弃CPU
    另一种是通过周期性的机制, 以固定的频率运行, 不时的检测是否有必要（不负责调度，只负责统计变量和更改标志位）

因此当前linux的调度程序由两个调度器组成：主调度器，周期性调度器(两者又统称为通用调度器(generic scheduler)或核心调度器(core scheduler))

并且每个调度器包括两个内容：调度框架(其实质就是两个函数框架)及调度器类

## 6种调度策略

内核有6种调度策略定义在`include/uapi/linux/sched.h`，SCHED_ISO是一个预留位。

```c
/*
 * Scheduling policies
 */
#define SCHED_NORMAL        0
#define SCHED_FIFO      1
#define SCHED_RR        2
#define SCHED_BATCH     3
/* SCHED_ISO: reserved but not implemented yet */
#define SCHED_IDLE      5
#define SCHED_DEADLINE      6
```

`SCHED_NORMAL`表示为普通进程;
`SCHED_FIFO`表示先入先出的实时类，此时如果没有更高优先级的进程就会一直被执行下去，不会停止;
`SCHED_RR`表示时间片轮转的实时进程。当调度程序把CPU分配给进程的时候，它把该进程的描述符放在运行队列链表的末尾。这种策略保证对所有具有相同优先级的SCHED_RR实时进程进行公平分配CPU时间;   
`SCHED_BATCH`表示采用分时策略，根据动态优先级，分配CPU资源。在有实时进程的时候，实时进程优先调度。但针对吞吐量优化，除了不能抢占外与常规进程一样，允许任务运行更长时间，更好使用高速缓存，适合于成批处理的工作;
`SCHED_ISO`是预留位，还未定义;
`SCHED_IDLE`是优先级最低的，也是调度idle运行的;
`SCHED_DEADLINE`是新支持的实时进程调度策略，针对突发型计算，并且对延迟和完成时间敏感的任务使用，基于EDF（earliest deadline first）;

## 5种调度类

内核中定义了5种调度类，每种调度器都会对应一个调度类，这个调度类包含了该调度器的所有操作方法：
`SCHED_NORMAL`和`SCHED_BATCH`使用 fair_sched_class类，定义位置在`kernel/sched/fair.c`
`SCHED_FIFO`和`SCHED_RR`使用 rt_sched_class类，定义位置在`kernel/sched/rt.c`
`SCHED_IDLE`使用   idle_sched_class类，定义位置在`kernel/sched/idle.c`
`SCHED_DEADLINE`使用 dl_sched_class类，定义位置在`kernel/sched/deadline.c`
这些调度类都是通过`struct sched_class`结构体定义的，位置为`kernel/sched/sched.h`等等

调度类型的级别为
 stop_sched_class -> dl_sched_class -> rt_sched_class -> fair_sched_class -> idle_sched_class

# 三、怎么调度

 调度产生的时机为两种，一种是自愿被调度，一种是强制被调度。

## 1、自愿调用

自愿被调用可以是通过schedule();或者是schedule_timeout();，在用户空间也可以使用sleep或者pause也是可以的，这些调度都是显式地让出，还有一些隐式地让出cpu，被调走，这种情况就是open，write，read，等等，涉及到了阻塞等待的都会被cpu视为主动放弃cpu，这种情况是比较难找到原因的。

## 2、被调用

   强制被调也是抢占式内核，当内核开启抢占模式的时候，会多出几个调度的时机，也就是在系统调用或者是在中断上下文的时候调用preemt_enable()或者是在终端上下文的处理函数返回的回到可抢占的上下文时。

```c
schedule(void)---------->|
                         | __schedule()
                         |   |
preempt_schedule(void)-->|   |-->pick_next_task(struct rq *rq, struct task_struct *prev)

```

 不管是哪一种最终都会调用到__schedule函数，该函数调用pick_netx_task，判断调度类型，然后根据调度类型去寻找需要调度的进程。

# 四、__schedule()的实现

## 1、选择需要执行的任务

__schedule()的是实现有鲤鱼理解调度类以及进程切换 在__schedule() 中主要做两件事情，
1、确认下一个需要调度的任务;
2、进行切换;
  rq：在多核系统中，进程切换总是发生在各个cpu core上，参数rq指向本次切换发生的那个cpu对应的run queue 
  rf: rq_flags结构体定义的一个变量，其结构体内部也只有两个成员，一个是flags，另一个是val ，只是一个标志的作用
  prev：需要被切换的线程 
  next：选择并且即将切换过去的线程

函数被__sched修饰，__sched的作用在后面讲解

```c
 static void __sched notrace __schedule(bool preempt)
{
    ... ... 

    next = pick_next_task(rq, prev, &rf);

        rq = context_switch(rq, prev, next, &rf); context_switch 

    ... ... 
}
```

pick_next_task是进程调度的关键步骤，主要功能是从发生调度的CPU的运行队列中选择一个进程运行。
系统中的调度顺序是：实时进程------->普通进程------>空闲进程。
分别从属于三个调度类：rt_sched_class，fair_sched_class和 idle_sched_class（每个cpu都有且只有一个Idle 进程且不会阻塞）。
在pick_next_task 中主要检测运行队列中是否有实时进程，如果有则从实时进程中选择进程，否则从普通进程中寻找，如果普通进程也没有就选择idle进程。
在 p = fair_sched_class.pick_next_task(rq, prev, rf);函数返回值中，p 会出现三种情况：
1)、RETRY_TASK:表示有更高的优先级进程在链表中，需要遍历链表找到优先级最高的进程。
2)、一个task 结构体的指针，表示已经选择到了一个进程。
3)、NULL，则调度idle进程（什么是idle进程，后面会讲）。

```cs
static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf) 
{
    const struct sched_class *class;
    struct task_struct *p;

    if (likely((prev->sched_class == &idle_sched_class || ------------------------[1]
            prev->sched_class == &fair_sched_class) &&
           rq->nr_running == rq->cfs.h_nr_running)) {

        p = fair_sched_class.pick_next_task(rq, prev, rf); -----------------------[2]
        if (unlikely(p == RETRY_TASK))
            goto again;

        if (unlikely(!p))
            p = idle_sched_class.pick_next_task(rq, prev, rf); -------------------[3]

        return p;
    }    
again:
    for_each_class(class) {-------------------------------------------------------[4]
        p = class->pick_next_task(rq, prev, rf); 
        if (p) {
            if (unlikely(p == RETRY_TASK))
                goto again;
            return p;
        }    
    }    
    /* The idle class should always have a runnable task: */
    BUG();
}
```

[1] prev = rq->curr; 也就是当前的运行队列是idle类或者fair类并且运行队列里有可运行的任务时，就进入内部判断
[2]通过公平调度类去寻找可运行队列，如果返回的p为空说明公平调度器没有找到可运行的进程，如果返回的值等于RETRY_TASK说明可运行队列中有更高优先级的队列，则跳转到goto语句里遍历出最高优先级的任务并执行。
[3]当没有可运行的普通任务，也没有更高优先级的任务事，把cpu让给idle调度。
[4]当上面的fair发现了更高优先级任务的时候可以通过这里来遍历，将其找出来并运行。

## 2、执行切换

一旦确认好了 pre和next 的task后，context_switch 就可以执行切换的工作，

```c
context_switch(struct rq *rq, struct task_struct *prev,
           struct task_struct *next, struct rq_flags *rf)
{   
    struct mm_struct *mm, *oldmm;

    prepare_task_switch(rq, prev, next);----------------------[1]
    
    mm = next->mm;
    oldmm = prev->active_mm;----------------------------------[2]

    arch_start_context_switch(prev);

    if (!mm) {------------------------------------------------[3]
        next->active_mm = oldmm;
        mmgrab(oldmm);
        enter_lazy_tlb(oldmm, next);--------------------------[4]
    } else
        switch_mm_irqs_off(oldmm, mm, next);
    
    if (!prev->mm) {------------------------------------------[5]
        prev->active_mm = NULL;
        rq->prev_mm = oldmm;
    }

    rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);

    prepare_lock_switch(rq, next, rf);
    
    /* Here we just switch the register state and the stack. */
    switch_to(prev, next, prev);------------------------------[6]
    barrier();
    
    return finish_task_switch(prev);
}
```

[1]准备切换，会将rq锁和中断全部关掉，在完成切换后再开启。
[2]next是需要切入的进程，prev是被切走的进程，mm变量是指向next进程的空间描述符，而oldmm是指prev进程空间的描述符。active_mmprev也就是当前正在使用的进程，对于普通进程来讲，他的task_struct任务描述符的mm 和 active_mm相同，对内核线程来讲，他的task_struct的mm成员为空（因为内核线程没有进程的地址空间），但是在内核线程被调度执行的时候，总是需要一个进程的地址空间，而active_mm就是向它借用的那哦进程地址空间。
[3]如果mm为空，则表示next是内核线程，这个时候next没有运行的地址空间，只能借用prev进程正在使用的那个地址空间(prev->active_mm)，但是发现在prev的结构体内发现了有mm和active_mm，这个时候不能选择prev->mm，因为prev也可能是一个内核线程。
[4]next如果是内核的线程那么就会调用enter_lazy_tlb,标记cpu上的next task 进入lazy TLB 模式。
[5]如果需要被切走的prev是一个内核线程，则需要把它借用的地址空间销毁，（因为内核线程prev已经被挂起了，不需要地址空间了），但是为什么会吧prev的mm设置为oldmm呢？

如果切入了内核线程，那么实际的进程地址空间并没有切换，应为这个切入的内核进程只是借用了切走时使用的那个地址空间active_mm，但是在内核中的实体而言，我们在释放一个内核数据对象之前，会检测这个对象是否还有拥有者，如果没有则去释放这个数据对象，同理，内存描述符也是使用这个原理。
在context_switch里面有一个这个函数mmgrab(oldmm);它实质上是

```cs
static inline void mmgrab(struct mm_struct *mm)                      
{
    atomic_inc(&mm->mm_count);
}
```

借用别人的地址空间，在prev切换之前对其加一个引用计数，然后在内核线程被切出的时候会将mm描述符归还，在整个内核线程运行的过程中，需要使用这个mm描述符，所以对这个mm描述符的增加减少计数操作只能在内核线程之外做操作，如果线程A--->B（内核线程）--->C 。A，C为非内核线程，拥有自己独立的地址空间，但是B是内核线程，没有自己独立的地址空间，需要借助其他的地址空间来运行，在A切入B的之前会使用mmgrab()对mm引用计数，表示B借走了自己A的地址空间，这个时候B就可以在A的地址空间运行，但是B运行之后没有减少引用计数，而是在C进程里去清除计数，这就感觉很神奇，为什么B不清除计数呢？（这个是最后调用了finish_task_switch(prev)，由prev自己清除的）。所以在下一个运行的时候再删除上一次的引用计数。所以前面的那行代码是这个作用。

[6]:真正的切换函数，这里面是进行切换，那么为什么switch_to需要三个参数呢？,而且最后一个参数也是prev，通过下面的代码可以看出

```cs
switch_to(prev, next, prev);
#define switch_to(prev, next, last)                 \
do {                                    \
    ... ...\
    (last) = resume(prev, next, task_thread_info(next));        \                                                                                                                                                    
} while (0)
```

在传进去的参数中，真正参与切换的是前面两个参数prev、next，通过resume汇编代码切换后会返回一个新的last进程，这个last进程其实就是prev进程。

```cs
    .align  5  
    LEAF(resume)
    mfc0    t1, CP0_STATUS
    LONG_S  t1, THREAD_STATUS(a0)----------------------------------[1]
    cpu_save_nonscratch a0-----------------------------------------[2]
    LONG_S  ra, THREAD_REG31(a0)

#if defined(CONFIG_STACKPROTECTOR) && !defined(CONFIG_SMP)
    PTR_LA  t8, __stack_chk_guard
    LONG_L  t9, TASK_STACK_CANARY(a1)
    LONG_S  t9, 0(t8)
#endif
/
    move    $28, a2------------------------------------------------[3]
    cpu_restore_nonscratch a1 

    PTR_ADDU    t0, $28, _THREAD_SIZE - 32
    set_saved_sp    t0, t1, t2 
    mfc0    t1, CP0_STATUS
    li  a3, 0xff01
    and t1, a3
    LONG_L  a2, THREAD_STATUS(a1)----------------------------------[4] 
    nor a3, $0, a3
    and a2, a3
    or  a2, t1
#ifdef CONFIG_CPU_LOONGSON3
    or  a2, ST0_MM
#endif
    mtc0    a2, CP0_STATUS
    move    v0, a0-------------------------------------------------[5]
    jr  ra   
    END(resume) 
```

这一段汇编代码是真正意义上实现了上下文切换。
[1] 保存上一个进程的状态
[2] cpu_save_nonscratch 将a1 保存进\$16~\$23, \$29(sp), \$30，以及\$31,cpu_restore_nonscratch 与他相反。
[3] 将上一个进程放进全局指针中
[4] 将下一个的进程状态加载到a2中，a2为下一个任务的栈空间，可以根据resume(prev, next, task_thread_info(next))这句的第三个参数看到。
[5] v0是存储表达式，或者是函数返回值，在这里很明显是将a0作为函数返回值返回。

最后返回一个a0的地址，a0就是传进来的prev参数，而这个参数最后回传给了context_switch中，执行了finish_task_switch(prev);函数，这个函数是在完成切换后被执行的，主要释放cpu，释放内存，释放锁等等一些清除工作。







