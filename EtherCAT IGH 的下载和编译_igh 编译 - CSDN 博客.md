---
url: https://blog.csdn.net/u014077947/article/details/127402412?spm=1001.2014.3001.5501
title: EtherCAT IGH 的下载和编译_igh 编译 - CSDN 博客
date: 2025-03-19 22:31:59
tag: 
summary: 
---
## [EtherCAT](https://so.csdn.net/so/search?q=EtherCAT&spm=1001.2101.3001.7020) IGH 的下载和编译

### 1、源码下载地址说明

[EtherCAT 官方下载网站](https://etherlab.org/download/ethercat/)

[EtherCAT 官方 git 下载网站](https://gitlab.com/etherlab.org/ethercat.git)

### 2、编译前一点小说明：

*   刚学习的时候，看到很多人说在看 EtherCAT IGH 的文档的时候说 EtherCAT IGH 只支持 2.6 和 3.x 的内核。这句话在 《EtherCAT IGH 1.52.pdf》中的 <1.1 Feature Summary> 提到了这么一句话 **Designed as a kernel module for Linux 2.6 / 3.x**，但是不知道是不是一直没有更新过来，还是有一些其他的原因，这句话应该是有问题的。经过测试其实是没有这个限制的，我现在在 Ubuntu 22.04, 内核版本为 5.15.0 的系统上面都编译安装成功了。
    
*   目前（2022.10.18）最新的 EtherCAT 版本应该是 **v1.5.2** 。当内核的版本超过 4.15.x 的时候，编译会出错。因为从 4.15 开始内核 timer 使用方式更改 [1]。这个后面在常见的编译错误中还会提到。
    

### 3、编译和安装

**其实在下载的源代码中的根目录中有一个文件 INSTALL，这个文件讲的就是如何安装 EtherCAT IGH。**

#### 3.1 编译配置

前面的一些操作会因为下载的源代码来源不一样有一些区别。

*   下载的压缩包

```
tar -xjf ethercat-1.5.2.tar.bz2
cd ethercat-1.5.2

```

*   下载的 git repo

```
cd ethercat
# 这个是用来生成配置文件的
./bootstrap 

```

后面的操作基本就是一样的了。

**注意：这里的配置每个人都可以设置的不一样，而在 EtherCAT IGH 也提供了很多的编译选项供用户选择。**

```
./configure --enable-8139too=no 

```

#### 3.2 安装

```
make all modules
sudo make modules_install install
sudo depmod

```

或者

```
make
make modules
sudo make install
sudo make modules_install
sudo depmod

```

#### 3.3 配置主站

1、安装完成后，会在 / opt / 目录下生成一个 etherlab / 文件夹，让看一下这个文件夹内有些什么，发现包含一些库文件和配置文件等。

```
@:~$ cd /opt/etherlab/
@:/opt/etherlab$ ls
bin  etc  include  lib  sbin

```

2、配置[网络设备](https://so.csdn.net/so/search?q=%E7%BD%91%E7%BB%9C%E8%AE%BE%E5%A4%87&spm=1001.2101.3001.7020)信息

```
cd /etc
sudo mkdir sysconfig
sudo cp /opt/etherlab/etc/sysconfig/ethercat /etc/sysconfig
sudo cp /opt/etherlab/etc/init.d/ethercat /etc/init.d
sudo cp /opt/etherlab/etc/ethercat.conf /etc

```

使用 [ifconfig](https://so.csdn.net/so/search?q=ifconfig&spm=1001.2101.3001.7020) 命令获取到网卡的 mac 地址。

```
@:~$ ifconfig
enp2s0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 98:23:a6:89:57:de 

```

从上面的可以看到网卡地址为： **98:23:a6:89:57:de**。  
修改下面这两个文件中的 MASTER0_DEVICE 和 DEVICE_MODULES 的数值。

```
sudo gedit /etc/sysconfig/ethercat

```

```
sudo gedit /usr/local/etc/sysconfig/ethercat

```

**修改如下**  
MASTER0_DEVICE=“98:29:a6:56:57:ce”  
DEVICE_MODULES=“generic”

如果是专用的网卡的话，那么 DEVICE_MODULES 的数值可以是这些值 **8139too, e100, e1000, e1000e, r8169, generic, ccat, igb**。***generic** 一般是用来指代通用网卡的。

3、配置用户态库

```
cd /etc/udev/rules.d
#新建一个ethercat的rule文件
sudo gedit 99-ethercat.rules

```

*   向文件中添加下面内容：  
    KERNEL==“EtherCAT[0-9]”, MODE=“0777”

**下面这个不执行好像也可以**

保存后退出，然后执行

```
sudo udevadm control --reload-rules 

```

4、配置实时权限

```
sudo gedit /etc/security/limits.conf
```

*   在该文件的最下方按照如下格式添加一行：  

```sh
    <username> hard rtprio 99
```

    
* 比如说改成这个样子： username hard rtprio 99

### 4、运行主站以及添加命令行工具

1、运行主站

```
@:~$ sudo /etc/init.d/ethercat start
Starting EtherCAT master 1.6.0-rc1  done
```

如果安装没有问题，会出现下面的提示：  
Starting EtherCAT master 1.5.2 done

就说明是安装成功了的。

2、 停止主站

```
@:~$ sudo /etc/init.d/ethercat stop
Shutting down EtherCAT master 1.6.0-rc1  done

```

3、添加命令行工具

```
vim ~/.bashrc
```

在其中添加如下代码：  

```
PATH=$PATH:/opt/etherlab/bin
```


然后执行

```
source ~/.bashrc
```

最后就可以愉快的使用 EtherCAT 提供的方便的命令行工具了。

### 5、编译可能遇到的问题

1、下面这个博主写的比较好，记录了一些常见的编译 EtherCAT IGH 会遇到的问题。  
[linux5.4 内核搭建 igh 主站第二次尝试](https://blog.csdn.net/ze3000/article/details/121155889#2.%20init_timer%E9%97%AE%E9%A2%98)

2、下面这个博主写的比较好，主要是这个博客的评论里面记录了一些其他人在编译 EtherCAT IGH 会遇到的问题以及博主的解决方法。  
[Linux 下 IGH Ethercat Master 安装](https://blog.csdn.net/qq_43530144/article/details/104042114)

3、checking for kernal for 8139too driver… configure error

这个是因为 8139too 网卡在当前 kenel 下不支持，解决办法：将对应的报错驱动禁用掉就可以了。

```sh
./configure --enable-8139too=no 

```

### 6、启动主站的时候可能遇到的问题

1、ERROR: could not insert ‘ec_master’: Invalid argument

```
@:~$ sudo /etc/init.d/ethercat start
Starting EtherCAT master 1.6.0-rc1 modprobe: ERROR: could not insert 'ec_master': Invalid argument failed.

```

如果没有按照 **3.3 配置主站** 中的 _2、配置网络设备信息_ 重新修改这两个文件中的内容，那么在启动主站的时候就会报这个错误。

2、Starting EtherCAT master 1.6.0-rc1 modprobe: FATAL:

```
@:~$ sudo /etc/init.d/ethercat start
Starting EtherCAT master 1.6.0-rc1 modprobe: FATAL: Module ec_master not found in directory /lib/modules/5.15.0-43-generic
 failed

```

这个错误应该是在编译之后没有运行 depmod 导致的。在编译的那个文件目录下面运行下面这句话即可。

```
sudo depmod

```

3、Starting EtherCAT master 1.5.2 ERROR: modinfo: could not find module ec_e1000 done

如果是报这种问题，一般都是因为 configure 的时候有没有加选项–enable-e1000，把这个选项加上去就好了。

### 7、修改源代码之后如何重新编译

有的时候我们在调试的时候，可能会去修改源代码，增加一些调试信息。那么我们在修改源代码之后如何重新编译呢？

*   在不修改编译配置的情况下，基本上按照 **### 3.2 安装** 的说明重新编译安装即可.
*   然后按照 **3.3 配置主站** 中的 _2、配置网络设备信息_ 重新修改这两个文件中的内容，也可能只要修改其中一个文件即可。

### 8、参考引用

感谢下面各位大佬的文章。

*   [1] [【实操填坑】在树莓派上编译 EtherCAT IgH Master 主站程序](https://www.shuzhiduo.com/A/E35p74LbJv/)