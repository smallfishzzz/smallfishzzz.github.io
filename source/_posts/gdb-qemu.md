---
title: gdb+qemu调试linux内核
date: 2022-07-13 17:24:18
tags: 
  - linux内核 
  - gdb
  - qemu
categories:
  - linux 调试技巧 
---





gdb + qemu 调试linux 内核

<!--more-->

# 搭建环境

## archlinux 环境  

安装kvm，qemu，libvirt, ovmf, virt-manager 等等

> sudo pacman -S qemu libvirt ovmf virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat  dmidecode   ebtables iptables  gdb

## 安装libguestfs

用于访问修改虚拟机vm 磁盘映像的工具(前提要配置AUR源,并安装 yaourt 或者 yay)

> yay -S libguestfs

##  启动libvirt服务

> sudo systemctl enable   libvirtd.service  
>
> sudo systemctl start   libvirtd.service

## 安装虚拟机 

安装好 qemu 后自带有qemu-img 命令， 使用qemu-img 创建一个镜像  -f 是指定镜像格式这里使用qcow2 

> qemu-img  create  debian.img  -f qcow2  40 G
>
> qemu-img info debian.img

## 纯命令行安装iso

将iso 挂载上，由于没有图形界面，所以需要提取iso的 内核文件和根文件系统

> mkdir  ./mnt
> mount -o loop  ./xxx.iso   ./mnt

安装 

> qemu-system-x86_64 -m 4096M -smp 4 -boot c -cpu host -hda ./debian.img --enable-kvm --nographic    -append console=ttyS0  -cdrom ./xxxiso \
> -kernel ./tmp/install.amd/vmlinuz  \
>  -initrd    ./tmp/install.amd/initrd.gz  



-m 指定虚拟机内存内存大小，默认单位是MB，

-boot  -d:启动顺序，哪个设备先启动,-d代表先从cd-rom启动，然后驱动从硬盘镜像正常启动。-c选项从硬盘镜像先启动。

-enable-kvm使用KVM进行加速

-smp 4: 虚拟机有4个vcpu，设置有几个核来模拟操作系统

-net nic -net user:默认启用运行虚拟机中的以太网链接

-hda testing-image.img:指定使用的硬件驱动的路径（之前创建的镜像路径）

-cdrom ubuntu-16.04.iso:最后告诉qemu从iso文件中启动系统

-cdrom添加fedora的安装镜像。可在弹出的窗口中操作虚拟机（如果有图形界面），安装操作系统，安装完成后重起虚拟机便会从硬盘(test.qcow2)启动。

 

-nographic -append console=ttyS0 : Redirectstandard output to host machine

重定向输出参数，如果不指定ttyS0则将无法在当前窗口看到安装过程，当然，有时候需要调整console参数才能在当前窗口查看到信息。为了使用户能以root身份通过串口登录，需要在该文件中添加“ttyS0”，说明系统认为这里的COM1是安全的

--kernel    使用哪个内核  （每个iso放的位置都不一样， 具体用find 搜一下在那）

--initrd    使用哪个initrd

>  安装后kill 掉， 我也不知道咋退出来

>  debootstrap 也可以构建根文件系统，但是感觉有很多问题，没仔细去研究

 

# 启动调试

## 启动qemu虚拟机

qemu 启动刚刚装好的img，--nographic 意思是文字界面的，在终端上打开, -s 提供本地端口1234（default） 给gdb 远程用， -S 是在启动的时候断住， 其他参数参数大概就字面意思， 命令运行后不会有什么反应，就是因为-S 断住了。

> qemu-system-x86_64 -enable-kvm \    
>     -m 4096M            \
>     -smp 4              \
>     --nographic         \
>     -hda ./debian.img   \
>     -s -S

## 使用gdb 连接

netstat  -nap |grep 1234  可以看到有qemu的端口是否已经打开，打开就说明ok了

>  gdb ./tmp/install.amd/vmlinuz     

因为vmlinux 是无符号的，所以这里看不出来有什么信息，

```c
Reading symbols from ./vmlinuz-5.10.0-15-amd64...
(No debugging symbols found in ./vmlinuz-5.10.0-15-amd64)
(gdb) target remote:1234
Remote debugging using :1234
0x000000000000fff0 in ?? ()
(gdb) bt
#0  0x000000000000fff0 in ?? ()
#1  0x0000000000000000 in ?? ()
(gdb) c
Continuing.
    
```

 再按c 就可以继续运行了，同时虚拟机这边也会继续运行,gdb 这边也可以继续使用

<img src="qemu-1.png" alt="qemu-1" style="zoom: 67%;" />



>  但是！！！， 这都没符号啊，我调毛线

## 使用自己编译的内核启动

> 这里问了一个不愿意透露姓名的大佬(Lonely Wind Wolf)，好像整个过程都是请教大佬的，但是没办法，我写的史记我做主

### 获取debian.img 的initrd， grub参数

获取qcow2 img 里面的内容， qcow2 格式的文件不能够直接挂

```c
sudo modprobe nbd
sudo qemu-nbd -c /dev/nbd0 --read-only /path/to/debian.img
udisksctl mount -b /dev/nbd0p1

```

这里会告诉你挂在在哪个位置,进去然后找到boot 目录，将initrd 文件拷贝出来， 并将grub.cfg 的cmdline内容拷贝出来，主要是用的uuid,  这里拷贝的是initrd.img-5.10.0-15-amd64  ， uuid 用的是6e2a155d-caba-4aae-97cb-fcdf73f18935 

```c
➜  6e2a155d-caba-4aae-97cb-fcdf73f18935 cd boot 
➜  boot ls
config-5.10.0-13-amd64  initrd.img-5.10.0-13-amd64  System.map-5.10.0-15-amd64
config-5.10.0-15-amd64  initrd.img-5.10.0-15-amd64  vmlinuz-5.10.0-13-amd64
grub                    System.map-5.10.0-13-amd64  vmlinuz-5.10.0-15-amd64
➜  boot cd grub 
➜  grub ls
fonts  grub.cfg  grubenv  i386-pc  locale  unicode.pf2
➜  grub cat grub.cfg |grep "root=UUID" 
	linux	/boot/vmlinuz-5.10.0-15-amd64 root=UUID=6e2a155d-caba-4aae-97cb-fcdf73f18935 ro  quiet
		linux	/boot/vmlinuz-5.10.0-15-amd64 root=UUID=6e2a155d-caba-4aae-97cb-fcdf73f18935 ro  quiet

➜  grub 

```

编译内核，找到bzImage, 再简单的修改一下 qemu启动参数，大概是这个样子：

> qemu-system-x86_64 -enable-kvm \
>         -m 4096M  \
>         -smp 4 \
>         -nographic  \
>         -kernel   ./arch/x86/boot/bzImage   \
>         -initrd ./initrd.img-5.10.0-15-amd64 \
>         -append " console=ttyS0 nokaslr  root=UUID=6e2a155d-caba-4aae-97cb-fcdf73f18935 " \
>         -hda   ./debian.img  \
>         -s -S                                                                                       

然后运行qemu， 启动gdb重复前面的步骤连接上虚拟机就ok了

```c
(gdb) target remote:1234
Remote debugging using :1234
0x000000000000fff0 in exception_stacks ()
(gdb) bt 
#0  0x000000000000fff0 in exception_stacks ()
#1  0x0000000000000000 in ?? ()
(gdb) c
Continuing.
^C
Thread 1 received signal SIGINT, Interrupt.
0xffffffff81bb296e in default_idle ()
(gdb) bt 
#0  0xffffffff81bb296e in default_idle ()
#1  0xffffffff81bb2b1d in default_idle_call ()
#2  0xffffffff8109a84f in do_idle ()
#3  0xffffffff8109a672 in do_idle ()
#4  0x4e9ed763dcb1d700 in ?? ()
#5  0xffff8880bffdf523 in ?? ()
#6  0x00000000000000e0 in ?? ()
#7  0x0000000000000048 in ?? ()
(gdb)
```

现在可以开始大调特调了 ～～ 



> 如果还是没有调试信息则需要手动打开CONFIG_DEBUG_INFO，千万不要打开CONFIG_DEBUG_INFO_REDUCED ，这个默认是关闭的，否则

![image-20220718173942788](image-20220718173942788.png)



引用:

https://blog.csdn.net/u011795345/article/details/78681213
