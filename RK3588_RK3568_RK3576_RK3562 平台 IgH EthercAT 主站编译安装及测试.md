---
url: https://zhuanlan.zhihu.com/p/27402782429
title: RK3588/RK3568/RK3576/RK3562 平台 IgH EthercAT 主站编译安装及测试
date: 2025-03-16 08:28:07
tag: 
summary: 
---
目录

*   [igh 主站编译安装说明](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23igh%25E4%25B8%25BB%25E7%25AB%2599%25E7%25BC%2596%25E8%25AF%2591%25E5%25AE%2589%25E8%25A3%2585%25E8%25AF%25B4%25E6%2598%258E)

*   [一、配置内核自带网卡驱动编译为模块](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E4%25B8%2580%25E9%2585%258D%25E7%25BD%25AE%25E5%2586%2585%25E6%25A0%25B8%25E8%2587%25AA%25E5%25B8%25A6%25E7%25BD%2591%25E5%258D%25A1%25E9%25A9%25B1%25E5%258A%25A8%25E7%25BC%2596%25E8%25AF%2591%25E4%25B8%25BA%25E6%25A8%25A1%25E5%259D%2597)

*   [1. 内核配置](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%231-%25E5%2586%2585%25E6%25A0%25B8%25E9%2585%258D%25E7%25BD%25AE)
*   [编译内核](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E7%25BC%2596%25E8%25AF%2591%25E5%2586%2585%25E6%25A0%25B8)
*   [编译内核模块](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E7%25BC%2596%25E8%25AF%2591%25E5%2586%2585%25E6%25A0%25B8%25E6%25A8%25A1%25E5%259D%2597)

*   [二、交叉编译 EtherCAT 主站](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E4%25BA%258C%25E4%25BA%25A4%25E5%258F%2589%25E7%25BC%2596%25E8%25AF%2591ethercat%25E4%25B8%25BB%25E7%25AB%2599)

*   [1. 普通 linux 或 preempt-rt](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%231-%25E6%2599%25AE%25E9%2580%259Alinux%25E6%2588%2596preempt-rt)

*   [1.1 配置](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%2311--%25E9%2585%258D%25E7%25BD%25AE)
*   [1.2 编译](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%2312--%25E7%25BC%2596%25E8%25AF%2591)
*   [1.3 安装到 TF 卡根目录](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%2313-%25E5%25AE%2589%25E8%25A3%2585%25E5%2588%25B0tf%25E5%258D%25A1%25E6%25A0%25B9%25E7%259B%25AE%25E5%25BD%2595)

*   [2. xenomai](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%232-xenomai)

*   [2.1 交叉编译 xenomai 库](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%2321--%25E4%25BA%25A4%25E5%258F%2589%25E7%25BC%2596%25E8%25AF%2591xenomai%25E5%25BA%2593)
*   [2.2 配置](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%2322--%25E9%2585%258D%25E7%25BD%25AE)
*   [2.3 编译](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%2323--%25E7%25BC%2596%25E8%25AF%2591)
*   [2.4 安装到 TF 卡根目录](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%2324-%25E5%25AE%2589%25E8%25A3%2585%25E5%2588%25B0tf%25E5%258D%25A1%25E6%25A0%25B9%25E7%259B%25AE%25E5%25BD%2595)

*   [四、安装目录打包拷贝到板子](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E5%259B%259B%25E5%25AE%2589%25E8%25A3%2585%25E7%259B%25AE%25E5%25BD%2595%25E6%2589%2593%25E5%258C%2585%25E6%258B%25B7%25E8%25B4%259D%25E5%2588%25B0%25E6%259D%25BF%25E5%25AD%2590)
*   [三、igh 用户库文件说明](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E4%25B8%2589igh%25E7%2594%25A8%25E6%2588%25B7%25E5%25BA%2593%25E6%2596%2587%25E4%25BB%25B6%25E8%25AF%25B4%25E6%2598%258E)
*   [四、igh 网卡配置](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E5%259B%259Bigh%25E7%25BD%2591%25E5%258D%25A1%25E9%2585%258D%25E7%25BD%25AE)
*   [五、启动及开机自启动](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E4%25BA%2594%25E5%2590%25AF%25E5%258A%25A8%25E5%258F%258A%25E5%25BC%2580%25E6%259C%25BA%25E8%2587%25AA%25E5%2590%25AF%25E5%258A%25A8)

*   [1. 启动 ethercat 主站测试](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%231-%25E5%2590%25AF%25E5%258A%25A8ethercat%25E4%25B8%25BB%25E7%25AB%2599%25E6%25B5%258B%25E8%25AF%2595)

*   [2. 设置自动启动](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%232-%25E8%25AE%25BE%25E7%25BD%25AE%25E8%2587%25AA%25E5%258A%25A8%25E5%2590%25AF%25E5%258A%25A8)

*   [六、测试对比](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E5%2585%25AD%25E6%25B5%258B%25E8%25AF%2595%25E5%25AF%25B9%25E6%25AF%2594)

*   [1. 使用 ec_generic 驱动 1ms 周期](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%231-%25E4%25BD%25BF%25E7%2594%25A8ec_generic%25E9%25A9%25B1%25E5%258A%25A81ms%25E5%2591%25A8%25E6%259C%259F)
*   [2. 使用 ec_stmmac 驱动 125us 周期](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%232-%25E4%25BD%25BF%25E7%2594%25A8ec_stmmac%25E9%25A9%25B1%25E5%258A%25A8125us%25E5%2591%25A8%25E6%259C%259F)

*   [七、其他](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wsg1100/p/18486644%23%25E4%25B8%2583%25E5%2585%25B6%25E4%25BB%2596)  
    本文记录 EtherCAT 主站典型编译配置流程，基于 [RK3562](https://zhida.zhihu.com/search?content_id=254503265&content_type=Article&match_order=1&q=RK3562&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDIyNTc0NjEsInEiOiJSSzM1NjIiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTQ1MDMyNjUsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.v2lt1VMkPSAVn09jC6VRUzMRBil0WIsbgYJ5NTp7V7w&zhida_source=entity) 创龙 SDK 描述，3568、3576、3588 编译仅 SDK 路径和上的差别，其他流程一致，整体流程也可用于其他平台。

##   
**igh 主站编译安装说明**

1.  文件说明：

igh 编译安装说明. md：安装说明  
etherlab.tar.xz：ethercat 主站源码，包含 rk stmmac 4.19 和 5.10 驱动  

1.  **源码包说明：**

[etherlab 官方主站](https://link.zhihu.com/?target=https%3A//gitlab.com/etherlab.org/ethercat)从 2013 年 1.5.2 版本起~ 2021 年没有维护，但该时间段内有一个非官方的分支一直维护，功能比较全面。虽然 2021 年开始官方重新维护，当前版本 1.6，但功能性能、支持的 os、从站启动速度等方面不如非官方维护版本。  
所以本次 stmamc 内核驱动基于[非官方版本](https://link.zhihu.com/?target=https%3A//github.com/ribalda/ethercat)开发，是一个完整的 git 仓库，内含本次开发完整的修改提交记录，若需要使用 [etherlab 官方主站](https://link.zhihu.com/?target=https%3A//gitlab.com/etherlab.org/ethercat)自行拷贝 stmamc 驱动代码到官方代码内编译构建即可。  
内核方面，preempt-RT、xenomai、普通 liunx 均可。  

1.  文档目录说明 (**根据实际情况修改**)

*   rk3562 内核源码目录：`/home/wsg1100/TL3562-EVM/rk3562_linux_sdk_release_v1.1.0`
*   主站源码目录：`/home/wsg1100/[PetaLinux](https://zhida.zhihu.com/search?content_id=254503265&content_type=Article&match_order=1&q=PetaLinux&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDIyNTc0NjEsInEiOiJQZXRhTGludXgiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTQ1MDMyNjUsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.S1YL3p_JlwIRdIRiivB7dD51zyX3JAL9a6W2p_NXgPs&zhida_source=entity)/PetaLinux/etherlab`
*   示例安装路径为:`/media/wsg1100/rk3562_rootfs_install/`，根据实际情况进行修改，或直接指定为将 SD 启动卡挂载根目录，直接安装到启动卡。
*   交叉编译工具路径为`/home/wsg1100/TL3562-EVM/rk3562_linux_sdk_release_v1.1.0/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/`

```
SDK_DIR="/home/wsg1100/TL3562-EVM/rk3562_linux_sdk_release_v1.1.0"
KERNEL_SRC="$SDK_DIR/kernel"
XENOMAI_SRC="$SDK_DIR/xenomai"
ETHERLAB_SRC="$SDK_DIR/etherlab"
INSTALL_OUTDIR="/home/wsg1100/rk3562_rootfs_install/"

```

## **一、配置内核自带网卡驱动编译为模块**

将内核自带网卡驱动编译为模块，才能替换 EtherCAT 主站驱动网卡，重新编译 SDK 烧录，步骤如下；

### **1. 内核配置**

终端输入以下内容，配置交叉编译环境变量

```
TOOLS_PATH=$SDK_DIR/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
CROSS_PATH=$TOOLS_PATH/bin
export PATH=$TOOLS_PATH:$CROSS_PATH:$PATH
export CROSS_COMPILE=aarch64-none-linux-gnu-
export ARCH=arm64

```

  
配置内核，找到内核自带驱动，修改为编译成模块：

```
cd $KERNEL_SRC
make menuconfig

```

```
Symbol: STMMAC_ETH [=m]
Type  : tristate 
Prompt: STMicroelectronics 10/100/1000/EQOS Ethernet driver 
Location:   
      -> Device Drivers
          -> Network device support (NETDEVICES [=y])
            -> Ethernet driver support (ETHERNET [=y])
		-> STMicroelectronics devices (NET_VENDOR_STMICRO [=y])  


```

![](https://pica.zhimg.com/v2-4fdecb84d8044299077a95940df24aae_r.jpg)

  
使能 PREEMPT-RT：  

![](https://pica.zhimg.com/v2-fb583ba5854d3c36da03ee06e2557bda_r.jpg)

  
配置保存后，将修改后的`.config`文件覆盖默认配置文件，重新编译 SDK 并烧录。  
**编译内核**

```
cp .config arch/arm64/configs/rockchip_linux_defconfig
cd $SDK_DIR
./build.sh kernel

```

将编译后的`resource.img` `zboot.img` `boot.img`根据相关文档烧录到板子。  
**编译内核模块**

```
cd $KERNEL_SRC

#  创建目录
mkdir -p ${INSTALL_OUTDIR}
#  编译内核模块
make CROSS_COMPILE=aarch64-none-linux-gnu- ARCH=arm64 INSTALL_MOD_PATH=${KMODULES_OUTDIR} modules -j$(nproc --all)

# 安装内核模块
make CROSS_COMPILE=aarch64-none-linux-gnu- ARCH=arm64 INSTALL_MOD_PATH=${KMODULES_OUTDIR} modules_install INSTALL_MOD_STRIP=1


```

  
**二、交叉编译 EtherCAT 主站**

### **1. 普通 linux 或 preempt-rt**

**1.1 配置**  
主站源码解压后，然后切换到主站目录下，执行以下命令生成 autoconf 配置脚本。

```
cd ${ETHERLAB_SRC}
./bootstrap

```

生成`configure`配置脚本后进行配置。

```
./configure --host=aarch64-none-linux-gnu --with-linux-dir=${KERNEL_SRC} --prefix=/usr/local --enable-dwmac-rk --disable-8139too --enable-kernel --enable-rtmutex --disable-hrtimer --disable-eoe --disable-generic


```

  
参数说明，分为 3 方面：

*   编译环境

*   `--host`： 指定交叉编译工具
*   `--with-linux-dir`: Linux 内核源码目录, **使用绝对路径**
*   `--prefix`： 指定实际板子安装目录，这里指定为`/usr/local`。

*   主站功能配置

*   `--enable-kernel`： 编译内核模块默认启用该选项
*   `--enable-rtmutex`: 使用 rtmutex，否则使用 sem（sem 没有 onwer 不具备优先级倒置）
*   `--disable-eoe`：禁用主站 EoE 功能（有需要自行开启）

网卡驱动选择

*   `--enable-dwmac-rk`: 编译 stmmac-rk 网卡驱动
*   `--disable-8139too`： **禁止编译 8139too 网卡驱动，否则会报错**
*   `--disable-generic`: 禁用 generic 驱动

**1.2 编译**

```
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j$(nproc) modules #编译内核
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j$(nproc) #编译应用工具和ethercat库

```

**1.3 安装到 TF 卡根目录**  
安装 Ethercat 用户配置工具和库到 TF 卡根目录下：

```
#安装ethercat工具和库指定安装目录
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- install DESTDIR=${INSTALL_OUTDIR}


```

安装 Ethercat 内核模块，安装到 TF 根目录。

```
#安装内核模块到指定安装目录
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- INSTALL_MOD_PATH=${INSTALL_OUTDIR} modules_install INSTALL_MOD_STRIP=1


```

注意：`INSTALL_MOD_PATH`最好选择内核交叉编译时安装内核模块的路径，** 好处是安装过程会自动处理模块依赖，拷贝到板子后无需处理模块符号依赖。  
**2. xenomai**  
**2.1 交叉编译 xenomai 库**

```
cd ${XENOMAI_SRC}
./scripts/bootstrap

#配置
./configure --host=aarch64-none-linux-gnu --enable-smp --enable-async-cancel --enable-assert --enable-pshared --enable-tls
#编译
make -j$(nproc)
#安装到根目录
sudo make -j$(nproc) DESTDIR=${INSTALL_OUTDIR} install
cd ${XENOMAI_SRC}
./scripts/bootstrap

#配置
./configure --host=aarch64-none-linux-gnu --enable-smp --enable-async-cancel --enable-assert --enable-pshared --enable-tls
#编译
make -j$(nproc)
#安装到根目录
sudo make -j$(nproc) DESTDIR=${INSTALL_OUTDIR} install


```

**2.2 配置**  
主站源码解压后，然后切换到主站目录下，执行以下命令生成 autoconf 配置脚本。

```
cd ${ETHERLAB_SRC}
./bootstrap

```

生成`configure`配置脚本后进行配置。

```
./configure --host=aarch64-none-linux-gnu --with-linux-dir=${KERNEL_SRC} --prefix=/ --with-xenomai-dir="${INSTALL_OUTDIR}/usr/xenomai/" --enable-rtdm --enable-dwmac-rk --disable-8139too --enable-kernel --enable-rtmutex --disable-hrtimer --disable-eoe --disable-generic


```

xenomai 参数说明，分为 3 方面：

*   编译环境

*   `--host`： 指定交叉编译工具
*   `--with-linux-dir`: Linux 内核源码目录, **使用绝对路径**
*   `--prefix`： 指定实际板子安装目录，这里指定为`/usr/local`。
*   **`--with-xenomai-dir`: xenomai 应用安装库路径，用来来编译链接 xenomai 环境的 igh 库。**

*   主站功能配置

*   `--enable-kernel`： 编译内核模块默认启用该选项
*   `--enable-rtmutex`: 使用 rtmutex 否则使用 sem，（sem 没有 onwer 不具备优先级倒置）
*   `--disable-eoe`：禁用主站 EoE 功能（有需要自行开启）
*   `--enable-rtdm`: **启用 xenomai RTDM 设备**，若不启用，xenomai 应用程序操作主站是通过非实时路径，无法保证实时性

*   网卡驱动选择

*   `--enable-dwmac-rk`: 编译 stmmac-rk 网卡驱动
*   `--disable-8139too`： **禁止编译 8139too 网卡驱动，否则会报错**
*   `--disable-generic`: 禁用 generic 驱动

**2.3 编译**

```
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j$(nproc) modules #编译内核
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j$(nproc) #编译应用工具和ethercat库

```

**2.4 安装到 TF 卡根目录**

安装 Ethercat 用户配置工具和库到 TF 卡根目录下：  

```
#安装ethercat工具和库指定安装目录
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- install DESTDIR=${INSTALL_OUTDIR}


```

安装 Ethercat 内核模块，安装到 TF 根目录。

```
#安装内核模块到指定安装目录
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- INSTALL_MOD_PATH=${INSTALL_OUTDIR} modules_install INSTALL_MOD_STRIP=1


```

  
注意：`INSTALL_MOD_PATH`最好选择内核交叉编译时安装内核模块的路径，** 好处是安装过程会自动处理模块依赖，拷贝到板子后无需处理模块符号依赖。

##   
**四、安装目录打包拷贝到板子**

  
若安装目录`${INSTALL_OUTDIR}`不是直接挂载的 TF 启动卡，需要将`${INSTALL_OUTDIR}`目录压缩打包到板子，解压到板子根文件系统。

##   
**三、igh 用户库文件说明**

  
上面已经将 igh 安装到文件系统`/usr/local/`（由主站配置的`--prefix`决定），各文件如下：

```
tree /usr/local/
.
├── bin
│   ├── ethercat      #ethercat 工具，用于主站状态查看、从站寄存器读写、调试等
│   └── ethercat_mbg  #ethercat 邮箱网关工具
├── etc
│   ├── ethercat.conf #网卡配置文件！！！！！
│   ├── init.d
│   │   └── ethercat  #init.d 启动脚本，早期Linux和嵌入式系统使用该方式
│   └── sysconfig
│       └── ethercat  #网卡配置文件，systemd使用，大部分linux发行版使用该方式
├── include
│   ├── ecrt.h
│   └── ectty.h
├── lib
│   ├── libethercat.a    #igh静态库
│   ├── libethercat.la
│   ├── libethercat.so -> libethercat.so.1.1.0  #igh 动态库，主要供ethercat工具、编译使用
│   ├── libethercat.so.1 -> libethercat.so.1.1.0
│   ├── libethercat.so.1.1.0
│   └── systemd
│       └── system
│           └── ethercat.service  #systemd 服务
└── sbin
    └── ethercatctl   #systemd服务执行脚本/usr/local/sbin/ethercatctl


```

## **四、igh 网卡配置**  

1.  配置禁止网卡自动加载

经过以上步骤，当前板子中存在两个 stmmac 网卡驱动，一个是内核源码编译的，一个是 ethercat 主站编译的，**为防止两个网卡自动加载，导致 ethercat 服务网卡加载失败，新建自动加载黑名单文件`/etc/modprobe.d/dwmac_rockchip.conf`，内容如下：**

```
blacklist dwmac_rockchip
blacklist ec_dwmac_rockchip

```

1.  设置 EtherCAT 主站驱动

按如下，ECAT 网卡配置文件位于`/usr/local/etc/ethercat.conf`，修改如下：

```
blacklist dwmac_rockchip
blacklist ec_dwmac_rockchip


```

若使用双主站，按如下方式配置:

```
#指定网卡，由于硬件不同MAC不同，所以这里统一使用"FF:FF:FF:FF:FF:FF"做默认匹配,要使用实际的MAC也行，那每块板子都需要手动配置
MASTER0_DEVICE="FF:FF:FF:FF:FF:FF"
MASTER1_DEVICE="FF:FF:FF:FF:FF:FF"
#指定网卡驱动
DEVICE_MODULES="dwmac_rockchip"


```

##   
**五、启动及开机自启动**

###   
**1. 启动 ethercat 主站测试**

  
完成以上步骤后，启动主站：

```
sudo ethercatctl start  #或者
sudo systemctl start ethercat.service

```

  
sudo ethercatctl start #或者 sudo systemctl start ethercat.service  
若有如下打印，说明环境设置成功。

```
pi@NanoPi-R6C:/usr/local/sbin$ dmesg | grep EtherCAT
[ 1415.034410] EtherCAT 0: Unregistered RTDM device EtherCAT0.
[ 1415.034669] EtherCAT: Master module cleaned up.
[ 1426.442436] EtherCAT: Master driver 1.5.2 unknown
[ 1426.442989] EtherCAT 0: Registered RTDM device EtherCAT0.
[ 1426.442996] EtherCAT: 1 master waiting for devices.
.....
[ 1489.916575] EtherCAT: Accepting 72:79:EC:56:32:E6 as main device for master 0.
....
[ 1490.121255] EtherCAT 0: Starting EtherCAT-IDLE thread.



```

  
**2. 设置自动启动**  
这里以 systemd 为例，执行如下命令配置开机启动 Etehrcat 主站

```
sudo systemctl enable ethercat.service

linaro@RK356x-Tronlong:~$ sudo ethercat master
Master0
  Phase: Idle
  Active: no
  Slaves: 1
  Ethernet devices:
    Main: 7e:cc:c3:17:55:22 (attached)
      Link: UP
      Tx frames:   379317
      Tx bytes:    22786646
      Rx frames:   379316
      Rx bytes:    22786586
      Tx errors:   0
      Tx frame rate [1/s]:    285    285    285
      Tx rate [KByte/s]:     16.7   16.7   16.7
      Rx frame rate [1/s]:    285    285    285
      Rx rate [KByte/s]:     16.7   16.7   16.7
    Common:
      Tx frames:   379317
      Tx bytes:    22786646
      Rx frames:   379316
      Rx bytes:    22786586
      Lost frames: 0
      Tx frame rate [1/s]:    285    285    285
      Tx rate [KByte/s]:     16.7   16.7   16.7
      Rx frame rate [1/s]:    285    285    285
      Rx rate [KByte/s]:     16.7   16.7   16.7
      Loss rate [1/s]:          0      0      0
      Frame loss [%]:         0.0    0.0    0.0
  Distributed clocks:
    Reference clock:   Slave 0
    DC reference time: 0
    Application time:  0
                       2000-01-01 00:00:00.000000000


```

##   
**六、测试对比**

  
这里使用 rk3562 + 两个汇川 ethercat 伺服驱动器进行测试，看看 xenomai 下分别使用原生的 ec_generic 驱动和专用 igh 驱动的区别，demo 源码位于目录`examples/igh_test`下，编译安装后会随 ethercat 库安装到`--prefix`下的 bin 目录下。

###   
**1. 使用 ec_generic 驱动 1ms 周期**

加载 ec_generic 驱动，这里直接使用命令进行

```
insmod  /usr/lib/modules/`uname -r`/ethercat/master/ec_master.ko main_devices=FF:FF:FF:FF:FF:FF
 modprobe dwmac_rockchip 
modprobe ec_generic

```

启动测试程序

```
dc_motion -c 3  -d 1 -t 1000000

```

*   -c 指定 CPU 3
*   -d 1 以主站作为参考时钟
*   -t 周期 1ms

其中主站收发时间计时方式如下：

```
/*接收时间*/ 
t_receive_start = sys_time_ns(); 
ecrt_master_receive(master); 
t_receive_end = sys_time_ns(); 

/*发送时间*/ 
t_send_start = sys_time_ns(); 
ecrt_master_send(master);
 t_send_end = sys_time_ns();

```

经过 6 小时测试后结果如下：

![](https://pic2.zhimg.com/v2-14fa69650bd96d592a334a8a4d565c6b_r.jpg)

*   period ：周期任务实际周期
*   latency：预期唤醒时间与时间唤醒时间差
*   exec ：运算、pdo 处理、统计、打印等耗时
*   send ：`ecrt_master_send()`耗时
*   receive ：`ecrt_master_receive()`耗时

看这 send 、receive 结果，你是不是觉得这结果还不错？注意！我们是从应用层面计算的收发调用的时间，至于调用`ecrt_master_send()`/`ecrt_master_receive()`后，报文什么时候被真正发送，和报文什么时候接收回来是由内核机制决定的，这个接口只是送、取 ethercat 报文的时间，不是网络收发软件层面消耗的所有时间。发出去的包，只要在 1ms 后，报文已经被接收回来就行，不妨碍下个周期计算发送即可，偶尔还是有帧未及时收回来的情况：  

![](https://pica.zhimg.com/v2-343cf8332a53f0095cfa69798054bc52_r.jpg)

  
如果此时跑 500us 周期，帧未及时收回来的情况会更频繁，详细原因的见本博客其他文章。  

![](https://pic4.zhimg.com/v2-74378d857848a128c841ae5cbfba60cb_r.jpg)

  
所以，使用 ec_generic 驱动能跑，但是跑不好，xenomai 能保证 ethercat 任务调度周期，但是保证不了网络包的收发实时性，网络包实时性涉及 DC 同步等等，在此不再赘述。

###   
**2. 使用 ec_stmmac 驱动 125us 周期**

  
使用 ec_stmmac 我们就不跑 1ms 浪费时间了，**直接跑 125us 周期.**  
加载 ec_stmmac 驱动，这里直接使用命令进行

```
insmod  /usr/lib/modules/`uname -r`/ethercat/master/ec_master.ko main_devices=FF:FF:FF:FF:FF:FF modprobe ec_dwmac_rockchip

```

启动测试程序

```
dc_motion -c 3  -d 1 -t 125000

```

*   -c 指定 CPU 3
*   -d 1 以主站作为参考时钟
*   -t 周期 125us

![](https://pic3.zhimg.com/v2-cef687b7df4a6cc825a2dd51de52260a_r.jpg)

经过 2 小时左右测试后结果如下：

![](https://pic3.zhimg.com/v2-138295661e1862e20c43192621390a1c_r.jpg)

  
这 send 、receive 结果，这才是网络收发软件层面消耗的所有时间。  
**七、其他**

1.  卸载 ethercat 驱动命令：`rmmod ec_dwmac_rockchip ec_stmmac_platform ec_stmmac ec_master`
2.  加载内核自带驱动命令：`modprobe dwmac_rockchip`
3.  卸载内核自带驱动命令：`rmmod dwmac_rockchip stmmac_platform stmmac`