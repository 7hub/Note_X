t> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/wq20202/article/details/130718111)

### 查看内核版本安装必要包

```
uname -a

```

![](https://i-blog.csdnimg.cn/blog_migrate/e71223b739c7f877d1bc3f895b079524.png)

安装必要包

```
apt install autoconf automake libtool make libncurses-dev flex bison libelf-dev libssl-dev zstd net-tools

```

###  下载内核以及补丁

[https://mirrors.edge.kernel.org/pub/linux/kernel/](https://mirrors.edge.kernel.org/pub/linux/kernel/ "https://mirrors.edge.kernel.org/pub/linux/kernel/")

下载 Linux 内核，找你的版本，不是上面几十兆的文件，往下翻有 Linux 开头的 100 多 M 的

![](https://i-blog.csdnimg.cn/blog_migrate/c91292c0b867d5692b60f28aab2babd1.png)

[https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/ "https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/")

下载实时补丁，注意找和内核一样的

![](https://i-blog.csdnimg.cn/blog_migrate/4bcdec1abb5d98e78f9ff5432435f986.png)

### 解压以及打补丁 

```
tar -zxvf linux-5.19.tar.gz
xz -d patch-5.19-rt10.patch.xz
```

![](https://i-blog.csdnimg.cn/blog_migrate/a222d6d195a6678e0daf1a1ce02ef0e1.png)

```
cd linux-5.19/
patch -p1 < ../patch-5.19-rt10.patch
```

###  内核配置

```
make menuconfig

```

进入界面化配置后的操作

General Setup -> Preemption Model 设置为 Fully Preemptible Kernel(RT)  
General Setup -> Timers subsystem -> Timer [tick](https://so.csdn.net/so/search?q=tick&spm=1001.2101.3001.7020) handling 设置为 Full dynticks system  
General Setup -> Timers subsystem 开启 High Resolution Timer Support  
Processor type and features -> Timer frequency 设置为 1000 HZ

记得保存后 exit

```sh
vi .config

```

CONFIG_SYSTEM_TRUSTED_KEYS=""

CONFIG_SYSTEM_REVOCATION_KEYS=""

![](https://i-blog.csdnimg.cn/blog_migrate/9c2aa941b4a8f369f36aec64bb593463.png)

 保存退出

### 编译安装

```
make -j`nproc`

```

![](https://i-blog.csdnimg.cn/blog_migrate/4e073f38a40793390d3a604196a07739.png)

 完成后

```
make modules_install
make install
```

### 配置 GRUB 启动项

```
vim /etc/default/grub

```

1, 注释掉下面这行将会显示引导菜单

GRUB_TIMEOUT_STYLE=hidden

2, 适当修改超时时间

GRUB_TIMEOUT=5 超时时间，单位 s

3，更新启动项配置

```
update-grub

```

### 重启

![](https://i-blog.csdnimg.cn/blog_migrate/8aa91396fb037cc7290677d5cb3808a5.png)

 ![](https://i-blog.csdnimg.cn/blog_migrate/11b132c5dceeb142e5c12fe7eaef0b6f.png)

 ![](https://i-blog.csdnimg.cn/blog_migrate/010e8d45a647de0041a9a6c91e187baf.png)

### 测试

```
apt-get install rt-tests 
cyclictest -t 5 -p 80 -i 1000
```

 cyclictest 将以最高优先级在 5 秒钟内进行 1000 次循环测试，以测量 Linux 系统的实时性能。测试完成后，cyclictest 会输出一些有关测试结果的统计信息

![](https://i-blog.csdnimg.cn/blog_migrate/fa2569de7211263d5f0e17a8fb4cd775.png)

原文链接：[https://www.ycyaw.com/Linux/167.html](https://www.ycyaw.com/Linux/167.html "https://www.ycyaw.com/Linux/167.html")