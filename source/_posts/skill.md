---
title: linux调试小技巧
date: 2022-07-13 14:16:43
tags: 
  - linux内核 
categories:
  - linux 调试技巧 
---

 

linux调试小技巧，帮助快速调试用户态程序或者内核态进程

<!--more-->



# 时间统计

## 用户态时间统计：

```c
NAME
       gettimeofday, settimeofday - get / set time
SYNOPSIS
       #include <sys/time.h>

       int gettimeofday(struct timeval *tv, struct timezone *tz);
       int settimeofday(const struct timeval *tv, const struct timezone *tz);

        struct timeval {
               time_t      tv_sec;     /* seconds */
               suseconds_t tv_usec;    /* microseconds */
           };

           struct timezone {
               int tz_minuteswest;     /* minutes west of Greenwich */
               int tz_dsttime;         /* type of DST correction */
           };
------------------------------------------------------------------------------
例：
#include <sys/time.h>
unsigned long gettime(void)
{
	struct timeval time;
	gettimeofday(&time,0);

	return (time.tv_sec * 1000000 + time.tv_usec);
}
```

## 内核态时间统计

> jiffies  或者  ktime_get()



# 设置进程亲和性

a) taskset   

b)numctl

c)代码：

```c
#define _GNU_SOURCE

int bind_cpu(int pid, int cpu)
{
	int ret;
	cpu_set_t mask;
	CPU_ZERO(&mask);
	CPU_SET(cpu, &mask);
	ret = sched_setaffinity(pid, sizeof(mask), &mask);
    
	if(ret < 0){
		printf("set affinity failed [%d]\n", ret);
	}
	printf("bind to %d success \n", cpu);
	return ret;
}

```



# dpkg 工具

debian包管理系统上，想下载一个进程的代码或者找到可执行程序的软件包 可通过dpkg  -S (内建的找不到)

> ~$ which deepin-terminal
> /usr/bin/deepin-terminal
>
> ~$ dpkg  -S /usr/bin/deepin-terminal 
> deepin-terminal: /usr/bin/deepin-terminal



dpkg -L  可以列出软件包具体安装了哪些内容

> ~$ dpkg -L deepin-terminal 
> /.
> /usr
> /usr/bin
> /usr/bin/deepin-terminal
> /usr/include
> /usr/include/terminalwidget5
> /usr/include/terminalwidget5/BlockArray.h
> /usr/include/terminalwidget5/Character.h
> /usr/include/terminalwidget5/CharacterColor.h
>
> ...

