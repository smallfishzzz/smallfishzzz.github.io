---
title: linux内核进程创建
date: 2022-07-13 17:20:07
tags: 
  - linux内核 
  - fork
  - idle
  - task
  - 进程管理
categories:
  - linux 内核代码阅读 
---





````
````

进程创建过程，如何让进程去运行，过程中的状态，以及如何退出。 

Kernel Version 4.19



<!--more-->

基于4.19 内核

# 进程结构体

## task_struct

include/linux/sched.h

```c
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
    /*
     * For reasons of header soup (see current_thread_info()), this
     * must be the first element of task_struct.
     */
    struct thread_info      thread_info;   //内存上下文，寄存器信息
#endif
    /* -1 unrunnable, 0 runnable, >0 stopped: */
    volatile long           state;      // 进程状态

    /*
     * This begins the randomizable portion of task_struct. Only
     * scheduling-critical items should be added above here.
     */
    randomized_struct_fields_start

    void                *stack;
    atomic_t            usage;
    /* Per task flags (PF_*), defined further below: */
    unsigned int            flags;
    unsigned int            ptrace;

#ifdef CONFIG_SMP
    struct llist_node       wake_entry;
    int             on_cpu;
#ifdef CONFIG_THREAD_INFO_IN_TASK
    /* Current CPU: */
    unsigned int            // cpu; 当前使用的cpu
#endif
  ... ... 

}

```





# 进程创建

## idle 进程的创建

1、执行start_kernel 的进程最后会变成第一个idle 进程，执行do_idle

2、smp架构的其他idle进程都是通过smp_init 来调用fork_idle 创建的

TODO:  都是start_kernel里的流程，用到的时候再看



## 用户态进程创建

```c
SYSCALL_DEFINE0(fork)
{
    return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
}
SYSCALL_DEFINE0(vfork)
{
    return _do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0,
            0, NULL, NULL, 0);
}
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
         int __user *, parent_tidptr,
         int __user *, child_tidptr,
         unsigned long, tls)
{
    return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}

```

## 内核线程的创建

内核线程的创建和用户态进程的创建都是调用的_do_fork，不同的是内核线程需要指定函数执行地址fn(参数2)，而用户态的除了clone不需要，那么都是调用 _do_fork，他们分别是怎么处理的，两种进程又是怎么运行的？

```c
通用接口
//1 
#define kthread_create(threadfn, data, namefmt, arg...) \
    kthread_create_on_node(threadfn, data, NUMA_NO_NODE, namefmt, ##arg)
//2 
#define kthread_run(threadfn, data, namefmt, ...)              \
({                                     \
    struct task_struct *__k                        \
        = kthread_create(threadfn, data, namefmt, ## __VA_ARGS__); \
    if (!IS_ERR(__k))                          \
        wake_up_process(__k);                      \
    __k;                                   \
})
    
//  kthreaed_create kthread_run 都会调用到 kthread_thread,
//上面的都是两个封装
    
/* 3
 * Create a kernel thread.  
 */
pid_t kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)         
{
    return _do_fork(flags|CLONE_VM|CLONE_UNTRACED, (unsigned long)fn,
        (unsigned long)arg, NULL, NULL, 0);
}

```

> 不管是 kthread_run 还是kthread_create 都会唤醒一个kthread的内核线程去调用kernel_thread 函数创建线程，然后同步返回，不同的就是kthread_run 最后会有一个唤醒（就是立马放到cpu运行队列上）而kthread_create是不会被执行的。
>
> <那个kthread的线程一直在kthreadd函数里死循环，等待消息来创建新的线程 >

## _do_fork

```c
_do_fork()
	|--> copy_process()
	|	|--> dup_task_struct() 申请task结构体、栈空间，存放的内存与父进程在同一个node上
	|	|--> 初始化task结构体的变量
	|	\--> 根据clone_flags 选择性copy 父进程的内容（默认是有父进程的cpu、fs、fd、sig等）
	\--> wake_up_new_task()  1、给进程选择一个新的cpu，并将进程放到该cpu的运行队列上（选择cpu是考虑负载的原因，尽可能给进程分配一个空闲的cpu，能够快速被调度执行）
```



## copy_thread_tls

copy_thread_tls是执行copy_process 的最后一步，这里负责设新进程的上下文，栈，sp等等，如果新进程不是PF_KTHREAD 则表示新进程p不是个内核线程，只需要设置一下pc 和sp就行，如果是内核线程则需要设置一下栈起始地址（stack_start 就是传入的fn）。

```c
include/linux/sched.h:
#define PF_KTHREAD      0x00200000  /* I am a kernel thread */

arch/arm64/kernel/process.c:
int copy_thread(unsigned long clone_flags, unsigned long stack_start,
        unsigned long stk_sz, struct task_struct *p)
{
		... ... 
    if (likely(!(p->flags & PF_KTHREAD))) {
		... ...  
    } else {
		... ...
        p->thread.cpu_context.x19 = stack_start;
        p->thread.cpu_context.x20 = stk_sz;
    }
    p->thread.cpu_context.pc = (unsigned long)ret_from_fork;
    p->thread.cpu_context.sp = (unsigned long)childregs;

}
```



## ret_from_fork

这里对内核线程和用户态进程做了区分，如果是内核线程则执行x19，这里放的就是指定的函数地址，如果是用户态进程则执行ret_to_user返回

```assembly
ENTRY(ret_from_fork)
    bl  schedule_tail                                
    cbz x19, 1f             // not a kernel thread
    mov x0, x20
    blr x19
1:  get_thread_info tsk
    b   ret_to_user
ENDPROC(ret_from_fork)
```

> ------------



# 进程状态



## 内核中状态的定义

内核中代码定义如下： include/linux/sched.h 

常用的就是TASK_RUNNING， TASK_INTERRUPTIBLE， TASK_UNINTERRUPTIBLE ， EXIT_DEAD， TASK_WAKING， TASK_NEW , TASK_IDLE

```c
/* Used in tsk->state: */
#define TASK_RUNNING            0x0000
#define TASK_INTERRUPTIBLE      0x0001
#define TASK_UNINTERRUPTIBLE        0x0002
#define __TASK_STOPPED          0x0004
#define __TASK_TRACED           0x0008
/* Used in tsk->exit_state: */
#define EXIT_DEAD           0x0010
#define EXIT_ZOMBIE         0x0020
#define EXIT_TRACE          (EXIT_ZOMBIE | EXIT_DEAD)
/* Used in tsk->state again: */
#define TASK_PARKED         0x0040
#define TASK_DEAD           0x0080
#define TASK_WAKEKILL           0x0100
#define TASK_WAKING         0x0200
#define TASK_NOLOAD         0x0400
#define TASK_NEW            0x0800
#define TASK_STATE_MAX          0x1000
/* Convenience macros for the sake of set_current_state: */
#define TASK_KILLABLE           (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
#define TASK_STOPPED            (TASK_WAKEKILL | __TASK_STOPPED)
#define TASK_TRACED         (TASK_WAKEKILL | __TASK_TRACED)

#define TASK_IDLE           (TASK_UNINTERRUPTIBLE | TASK_NOLOAD)

/* Convenience macros for the sake of wake_up(): */
#define TASK_NORMAL         (TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)

```

## 用户层看到的进程状态

> man ps 

```c
D (TASK_UNINTERRUPTIBLE) 	不可中断的睡眠状态
I (TASK_IDLE) 	 	  	   idle 的内核线程
R (TASK_RUNNING)				正在运行，或在队列中的进程
S (TASK_INTERRUPTIBLE)		可中断的睡眠状态
T (TASK_STOPPED)				停止状态
t (TASK_TRACED)				被跟踪状态
Z (TASK_DEAD - EXIT_ZOMBIE)  退出状态，但没被父进程收尸，成为僵尸状态
X (TASK_DEAD - EXIT_DEAD)    退出状态，进程即将被销毁

<    高优先级
N    低优先级
L    有些页被锁进内存
s    包含子进程
+    位于前台的进程组；
l    多线程，克隆线程  multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
```

## 系统中常见的状态：

> ps -aux 查看进程状态

- R   即 TASK_RUNNING ， 表示该进程正在cpu上运行，也就是在执行程序代码

```c
#include <stdio.h>
int main()
{
        while(1);
        return 0;
}
```

![psa](psa.png)

- S  即 TASK_INTERRUPTIBLE ，表示该进程已经进入可中断的睡眠，这种一般都是主动调用sleep，或者因为等待数据而进入阻塞时的状态，通过ps 可以看到系统大部分的进程都是 S或 I 状态

- I  即 TASK_IDLE , 表示该进程已经进入IDLE  模式



## 进程状态的切换

​	Ready(runnable)表示该进程已经在cpu的运行队列上，但是还没被调度执行。

​	Running表示正该进程已经在运行了，这个图表示进程Running与各个状态的切换关系

R->t->Ready , R ->T->Ready , R->s->Ready , R->D->Ready , R->Z , R->Z

![image-20220718154845935](image-20220718154845935.png)





# 进程退出

do_exit()  （非关机流程）

分两部分，do_exit 里释放进程相关的结构体，数据内容等等，但进程本身的堆栈和task_struct 要等到别人来释放
a)  do_task_dead() 里将进程设置为 TASK_DEAD， 并且主动放弃cpu     __schedule，

b）__schedule 切换进程后，就会释放prev 进程的stack 和task_struct 



TODO:



参考：

https://blog.csdn.net/lyndon_li/article/details/114295654
