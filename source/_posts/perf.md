---
title: linux perf 使用
date: 2022-07-11 15:40:50 
tags: 
  - linux内核 
  - perf
  - 火焰图
categories:
  - linux 调试技巧
---



# 1 perf工具 

​	 Perf是Linux kernel自带的系统性能优化工具。Perf的优势在于与Linux Kernel的紧密结合，它可以最先应用到加入Kernel的new feature。perf可以统计硬件相关的信息如cpu cache miss、tlb miss、ips等等，软件上可以查看代码执行热点函数、热点指令、cpu迁移、等等，从而帮助开发者来优化程序性能。



<!--more-->



# 2 获得perf

a) 系统安装 

debian、ubuntu、uos 下：`sudo apt install linux-perf` 

archlinux下 ` sudo pacman -S perf` 

b) 获取内核源码，在源码目录下执行`make tools/perf` 或者`cd tools/perf  &&  make ` 

编译成功后在`tools/perf`目录下会生成perf 文件，cp 到`/usr/bin/` 目录下就ok了

>  编译尽量去掉警告，不然会有很多功能不支持，原因是有很多依赖包没有安装，最好根据编译提示都装一边

# 3 常用方法

## a) perf stat

> Usage: perf stat [<options>] [<command>]

```c 
~$ sudo perf stat
^C
 Performance counter stats for 'system wide':
          9,546.15 msec cpu-clock                 #    7.957 CPUs utilized         
             5,160      context-switches          #    0.541 K/sec                 
               952      cpu-migrations            #    0.100 K/sec                 
             3,340      page-faults               #    0.350 K/sec                 
       678,561,527      cycles                    #    0.071 GHz                   
       548,416,618      instructions              #    0.81  insn per cycle       
   <not supported>      branches                                                   
         3,406,596      branch-misses  
              
       1.199787833 seconds time elapsed
```



> perf stat  ls 



```c 
~$ sudo perf stat ls
^C
Performance counter stats for 'ls':
              0.91 msec task-clock                #    0.648 CPUs utilized         
                 0      context-switches          #    0.000 K/sec                 
                 0      cpu-migrations            #    0.000 K/sec                 
                90      page-faults               #    0.099 M/sec                 
         1,595,716      cycles                    #    1.760 GHz                   
         1,454,990      instructions              #    0.91  insn per cycle       
   <not supported>      branches                                                   
            15,844      branch-misses 
                  
       0.001398443 seconds time elapsed

```

> context-switches   进程的上下文切换次数
> cpu-migrations      进程的cpu迁移次数
> branch-miss           进程代码的分支预测miss次数(硬件性)   

 等等一些性能指标，还有更多指标可以通过perf list 来查看，每个硬件提供的可能不相同， 如果想使用下面的指标通过-e 指定 ' ,' 隔开如  sudo perf stat -e branch-misses,L1-dcache-loads      [command]



```c
~$ perf list

List of pre-defined events (to be used in -e):

  branch-misses                                      [Hardware event]
  bus-cycles                                         [Hardware event]
  cache-misses                                       [Hardware event]
  cache-references                                   [Hardware event]
  cpu-cycles OR cycles                               [Hardware event]
  instructions                                       [Hardware event]
  stalled-cycles-backend OR idle-cycles-backend      [Hardware event]
  stalled-cycles-frontend OR idle-cycles-frontend    [Hardware event]

  alignment-faults                                   [Software event]
  bpf-output                                         [Software event]
  context-switches OR cs                             [Software event]
  cpu-clock                                          [Software event]
  cpu-migrations OR migrations                       [Software event]
  dummy                                              [Software event]
  emulation-faults                                   [Software event]
  major-faults                                       [Software event]
  minor-faults                                       [Software event]
  page-faults OR faults                              [Software event]
  task-clock                                         [Software event]

  duration_time                                      [Tool event]
  L1-dcache-load-misses                              [Hardware cache event]
  L1-dcache-loads                                    [Hardware cache event]
  L1-icache-load-misses                              [Hardware cache event]
  L1-icache-loads                                    [Hardware cache event]
  branch-load-misses                                 [Hardware cache event]
  branch-loads                                       [Hardware cache event]
  dTLB-load-misses                                   [Hardware cache event]
  dTLB-loads                                         [Hardware cache event]
  iTLB-load-misses                                   [Hardware cache event]
  iTLB-loads                                         [Hardware cache event]
```





## b) perf top

## c) perf record

