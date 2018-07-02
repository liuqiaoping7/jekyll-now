---
layout: post
title: Embedded system & Cross Compilation
---
以下我们选取TI公司的DM814x/AM387x EVM Baseboard为平台来讨论嵌入式系统、交叉编译环境的一些常识。

#  1、背景 #
##   1.1、嵌入式系统 #
这是维基百科词条，说明了嵌入式系统的特性、常见的试用领域：  
>An embedded system is a computer system with a dedicated function within a larger mechanical or electrical system, often with real-time computing constraints. It is embedded as part of a complete device often including hardware and mechanical parts. Embedded systems control many devices in   
common use today. Ninety-eight percent of all microprocessors are manufactured as components of embedded systems.
Examples of properties of typical embedded computers when compared with general-purpose counterparts are low power consumption, small size, rugged operating ranges, and low per-unit cost. This comes at the price of limited processing resources, which make them significantly more difficult to program and to interact with. However, by building intelligence mechanisms on top of the hardware, taking advantage of possible existing sensors and the existence of a network of embedded units, one can both optimally manage available resources at the unit and network levels as well as provide augmented functions, well beyond those available.For example, intelligent techniques can be designed to manage power consumption of embedded systems.

##    1.2、交叉编译 #
这是维基百科词条，说明了交叉编译的定义、存在的意义：  
>A cross compiler is a compiler capable of creating executable code for a platform other than the one on which the compiler is running. For example, a compiler that runs on a Windows 7 PC but generates code that runs on Android smartphone is a cross compiler. A cross compiler is necessary to compile code for multiple platforms from one development host. Direct compilation on the target platform might be infeasible, for example on a microcontroller of an embedded system, because those systems contain no operating system. In paravirtualization, one computer runs multiple operating systems and a cross compiler could generate an executable for each of them from one main source.

#  2、获取资料 #
通常芯片公司推出一款芯片都会给出说明特性、开发套件。  
见 http://www.ti.com/product/TMS320DM8148/ 。
+   说明特性  
通常是给二次开发的项目负责人看的。项目负责人的职能是面对具体的问题或者需求找到合适的解决方案，而说明特性就是告诉他们如果你碰到了以下领域的问题或者以下指标需求，请选择我。  
见 http://www.ti.com/product/TMS320DM8148/description#descriptions 。
+   开发套件  
通常是给二次开发的项目工程师看的。通常以评估板卡来展示外围接口扩展、以示例软件来展示实际业务能力。  
见 http://www.ti.com/product/TMS320DM8148/toolssoftware。  
以下我们将以项目工程师的角色进行实际业务二次开发。

#  3、环境搭建 #

##    3.1 设置环境变量包含交叉编译工具链路径 #
    ruisu@ruisu:~/share$ sudo vi /etc/bash.bashrc
    在文件末尾添加:
    export PATH=$PATH:/home/ruisu/share/dvrrdk-rs8148/ti_tools/cgt_a8/arago/linux-devkit/bin
    注：如果提示
    /bin/sh: 1: /home/ruisu/share/dvrrdk-rs8148/dvr_rdk/../ti_tools/cgt_a8/arago/linux-devkit/bin/arm-arago-linux-gnueabi-gcc: not found
    64位系统需要32位库支持：
    ruisu@ruisu:~/share/dvrrdk-rs8148/dvr_rdk$ sudo apt-get install ia32-libs
    再编译应该就没有问题了。

##    3.2 配置tftp服务 #
    安装tftp：
    ruisu@ruisu:~/share$ sudo apt-get install tftp-hpa tftpd-hpa xinetd
    这里tftpd为服务端tftp为客户端(可以用于测试服务端是否成功)。
    创建tftp目录：
    ruisu@ruisu:~/share$ mkdir tftproot
    ruisu@ruisu:~/share$ chmod 777 tftproot
    修改tftp配置文件，如果没有就创建：
    ruisu@ruisu:~/share$ sudo vi /etc/xinetd.d/tftp
    添加
    service tftp
             {
                 disable         = no
                 socket_type     = dgram
                 protocol        = udp
                 wait            = yes
                 user            = ruisu
                 server          = /usr/sbin/in.tftpd
                 server_args     = -s /home/ruisu/share/tftproot
                 source          = 11
                 cps             = 100 2
                 flags =IPv4
             }
    ruisu@ruisu:~/share$ sudo vi /etc/inetd.conf
    添加
    tftp	dgram	udp	wait	nobody		/usr/sbin/tcpd
    /usr/sbin/in.tftpd	/home/ruisu/share/tftproot
    ruisu@ruisu:~/share$ sudo vi /etc/default/tftpd-hpa
    改为
    #RUN_DAEMON="no"
    #OPTIONS="-s /home/ruisu/share/tftproot -c -p -U tftpd"
    TFTP_USERNAME="tftp"
    TFTP_DIRECTORY="/home/ruisu/share/tftproot"
    TFTP_ADDRESS="0.0.0.0:69"
    TFTP_OPTIONS="-l -c -s"
    重启tftp服务：
    ruisu@ruisu:~/share$ sudo service xinetd restart
    xinetd stop/waiting
    xinetd start/running, process 5159
    测试一下 tftp服务：
    ruisu@ruisu:~/share$ tftp 127.0.0.1 ，能在非tftp目录下下载文件即为成功。

##    3.3 配置nfs服务 #
    安装nfs服务器端：
    ruisu@ruisu:~/share/tftproot/rootfs-avcap$ sudo apt-get install nfs-kernel-server
    设置NFS-Server目录：
    ruisu@ruisu:~/share/tftproot/rootfs-avcap$ sudo vi /etc/exports
     添加
    /home/ruisu/share/rootfs-avcap	\*(rw,sync,no_subtree_check,no_root_squash)
    重启portmap（如果有必要）和nfs-kernel-server服务：
    ruisu@ruisu:~/share/rootfs-avcap$ sudo service portmap restart
    ruisu@ruisu:~/share/rootfs-avcap$ sudo service nfs-kernel-server restart
    测试本机能否挂载成功：
    ruisu@ruisu:~/share/rootfs-avcap$ sudo mount -t nfs 192.168.1.119:/home/ruisu/share/rootfs-avcap /mnt/share


#  4、二次开发 #
