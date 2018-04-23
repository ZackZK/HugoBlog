---
date: 2015-07-13T00:00:00Z
description: 介绍linux如何创建并配置swap分区
modified: 2015-07-13
tags:
- linux
- swap
title: 如何创建swap分区
---

在安装运行某些软件的时候，有时会碰到一些内存不足的情况，这个时候一个办法就是使用swap分区来虚拟出额外的内存空间。
我们主要使用了swapon这个命令：

    {{< highlight bash >}}
    root@iZ23om0selfZ:~# swapon -h

    Usage:
    swapon \[options] \[<spec>]

    Options:
    -a, --all              enable all swaps from /etc/fstab
    -d, --discard          discard freed pages before they are reused
    -e, --ifexists         silently skip devices that do not exis
    -f, --fixpgsz          reinitialize the swap space if necessary
    -h, --help             display help and exit
    -p, --priority <prio>  specify the priority of the swap device.
    -s, --summary          display summary about used swap devices and exit
    -v, --verbose          verbose mode
    -V, --version          display version and exit

    The <spec> parameter:
    -L <label>             LABEL of device to be used
    -U <uuid>              UUID of device to be used
    LABEL=<label>          LABEL of device to be used
    UUID=<uuid>            UUID of device to be used
    <device>               name of device to be used
    <file>                 name of file to be used
    {{< / highlight >}}

# 具体方法如下：

1\. 先查看当前系统swap分区情况

    {{< highlight bash >}}
    sudo swapon -s
    {{< / highlight >}}
    
    输出如下：
    {{< highlight bash >}} 
    root@iZ23om0selfZ:~# swapon -s
    Filename                                Type            Size    Used    Priority
    /mnt/swap.0                             file            1048572 796108  -1
    {{< / highlight >}}
    
    上面的例子里，说明系统正在使用一个1GB大小的/mnt/swap.0 swap文件。

2\. 创建一个linux swap文件供swap使用

    {{< highlight bash >}}
    sudo dd if=/dev/zero of=/mnt/swapfile bs=1024 count=256k
    sudo mkswap /mnt/swapfile
    {{< / highlight >}}
    
    其中/mnt/swapfile是本例中的swap文件名， 该文件大小 1024 * 256K = 256M


    输出：
    {{< highlight bash >}}
    Setting up swapspace version 1, size = 10236 KiB
    no label, UUID=f6949220-6671-48c7-afd5-54d2b3f02e70
    {{< / highlight >}}
    
3\. 激活swap文件

    {{< highlight bash >}}
    sudo swapon /mnt/swapfile
    {{< / highlight >}}
   
4\. 放入fstab文件

    在/etc/fstab中加入下面一行
    
    {{< highlight bash >}}
    /mnt/swapfile       none    swap    sw      0       0
    {{< / highlight >}}
   
5\. 性能上的考虑

    swap系统中swappiness参数会影响系统swap策略和时机。swappiness=0的时候表示最大限度使用物理内存，
    然后才是swap空间，swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面swap的交换策略。
    一般建议改为10.

    可以临时将其修改，方法如下
    
    {{< highlight bash >}}
    # Set the swappiness value as root
    echo 10 > /proc/sys/vm/swappiness

    # Alternatively, run this 
    sysctl -w vm.swappiness=10

    # Verify the change
    cat /proc/sys/vm/swappiness
    10

    # Alternatively, verify the change
    sysctl vm.swappiness
    vm.swappiness = 10
    {{< / highlight >}}

    也可以修改/etc/sysctl.conf文件，加入或者修改下面一行内容使重启后仍然有效：
    {{< highlight bash >}}
    vm.swappiness = 10
    {{< / highlight >}}
