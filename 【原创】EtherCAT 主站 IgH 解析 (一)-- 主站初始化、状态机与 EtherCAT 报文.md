---
url: https://www.cnblogs.com/wsg1100/p/14433632.html
title: 【原创】EtherCAT 主站 IgH 解析 (一)-- 主站初始化、状态机与 EtherCAT 报文
date: 2025-03-18 10:39:41
tag: 
summary: 
---
目录

*   [0 获取源码](#0-获取源码)
*   [1 启动脚本](#1-启动脚本)
    *   [1.1 start](#11-start)
    *   [1.2 stop](#12-stop)
*   [2 主站实例创建](#2-主站实例创建)
    *   [2.1 Master Phases](#21-master-phases)
    *   [2.2 数据报与状态机](#22-数据报与状态机)
        *   [数据报](#数据报)
        *   [状态机](#状态机)
    *   [2.3 master 状态机及数据报初始化](#23-master状态机及数据报初始化)
    *   [2.4 初始化 EtherCAT device](#24-初始化ethercat-device)
    *   [2.5 设置 IDLE 线程的发送间隔:](#25-设置idle-线程的发送间隔)
    *   [2.6 初始化字符设备](#26-初始化字符设备)
*   [3 网卡](#3-网卡)
*   [4 IDLE 阶段内核线程](#4-idle阶段内核线程)
    *   [4.1 数据报发送](#41-数据报发送)
    *   [4.2 数据报接收](#42-数据报接收)

版权声明：本文为本文为博主原创文章，转载请注明出处。如有问题，欢迎指正。博客地址：[https://www.cnblogs.com/wsg1100/](https://www.cnblogs.com/wsg1100/)

## 0 获取源码

IgH EtherCAT Master 现已迁移到 gitlab：[https://gitlab.com/etherlab.org/ethercat](https://gitlab.com/etherlab.org/ethercat) 文档在 [ethercat_doc](https://docs.etherlab.org/ethercat/1.6/pdf/ethercat_doc.pdf) 。  
可以使用以下命令克隆存储库：

```
git clone https://gitlab.com/etherlab.org/ethercat.git
git checkout stable-1.5


```

上面的是 igh 官方的仓库，下面是其他 Ethercat 主站：  
[https://github.com/ribalda/ethercat](https://github.com/ribalda/ethercat) 基于官方，功能更为全面的 igh etehrcat 主站。  
[https://github.com/leducp/KickCAT](https://github.com/leducp/KickCAT) 一个 C++ 写的全新 etehrcat 主站，目前功能不完善。  
[https://github.com/ethercrab-rs/ethercrab](https://github.com/ethercrab-rs/ethercrab) 一个纯 rust 语言编写的全新 etehrcat 主站，目前功能不完善。

本文主要讲 igh。

## 1 启动脚本

igh 通过脚本来启动，可以是 systemd、init.d 或 sysconfig。分别位于源码 script 目录下：

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20210219114206819.png)

对于 systemd 方式，编译时由`ethercat.service.in`文件生成`ethercat.service`。`ethercat.service`中指定了执行文件为`ethercatctl`.`ethercatctl`文件由 ``ethercatctl.in` 生成。init.d 和 sysconfig 类似，都是生成一个可执行脚本, 且脚本完成的工作一致，主要完成加载主站模块、网卡驱动、给主站内核模块传递参数、卸载模块等操作。

`ethercat.conf`共同的配置文件，配置主站使用的网卡、驱动等信息。下面看脚本 start 和 stop 所做的工作。

### 1.1 start

1.  加载 ec_master.ko
    
    模块参数：
    
    *   main_devices : 主网卡 MAC 地址，多个 main_devices 表示创建多个主站，MAC 参数个数 master_count。
    *   backup_devices : 备用网卡 MAC 地址，多个 backup_devices 表示创建多个主站，MAC 参数个数 backup_count。
    *   debug_level : 调试 level，调试信息输出级别。
    *   eoe_interfaces eoe 接口，eoe_count 表示 eoe_interfaces 的个数。
    *   eoe_autocreate 是否自动创建 eoe handler。
    *   pcap_size Pcap buffer size。
2.  遍历配置文件中`/etc/sysconfig/ethercat`的环境变量 DEVICE_MODULES，位于`ethercat.conf`中。
    
3.  在每个 DEVICE_MODULES 前添加前缀`ec_`替换，如 DEVICE_MODULES 为`igb`的话，添加前缀后为`ec_igb`。
    
4.  `modinfo`检查该模块是否存在。
    
5.  对于非`generic`和`rtdm`的驱动，需要先将网卡与当前驱动`unbind`，**unbind 后的网卡才能被新驱动接管**。
    
6.  加载该驱动。
    

start 加载了两个内核模块，ec_master.ko 和网卡驱动 ec_xxx.ko，ec_master 先根据内核参数（网卡 MAC）来创建主站实例，此时主站处于 **Orphaned phase**。后续加载网卡驱动 ec_xxx.ko，执行网卡驱动 probe，根据 MAC 地址将网卡与主站实例匹配，此时主站得到操作的网卡设备，进入 **Idle phase**。详细过程见后文。

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/ecdev-offfer.png)

### 1.2 stop

卸载内核模块 ec_master.ko 和网卡驱动 ec_xxx.ko。

1.  遍历配置文件中的环境变量 DEVICE_MODULES。
2.  在每个 DEVICE_MODULES 前添加前缀‘ec_’替换。
3.  lsmod 检查该模块是否被加载。
4.  卸载模块。

后文 “主站” 和”master“均表示主站或主站实例对象，slave 和从站表示从站或从站对象，中英混用，不刻意区分。

## 2 主站实例创建

一个使用 IgH 的控制器中可以有多个 EtherCAT 主站，每个主站可以绑定了一个主网卡和一个备用网卡，一般备用网卡不使用，线缆冗余功能时才使用备用网卡（文中假设只有一个主网卡）。

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20210523100016531.png)

start 过程中执行`insmod ec_master.ko`，这个时候，先调用的就是 `module_init` 调用的初始化函数`ec_init_module()`。先根据参数`main_devices` 的个数`master_count`, 每个 master 需要一个设备节点来与应用程序交互，所以`master_count`决定需要创建多少个 matser 和多少个字符设备；

这里先分配注册`master_count`个字符设备的主次设备号`device_number`和名称, 这样每个 master 对应的设备在文件系统中就是`/dev/EtherCAT0`、`/dev/EtherCAT1`...(Linux 下)。

```
if (master_count) {
        if (alloc_chrdev_region(&device_number,
                    0, master_count, "EtherCAT")) {
            EC_ERR("Failed to obtain device number(s)!\n");
...
        }
    }

 class = class_create(THIS_MODULE, "EtherCAT");


```

解析模块参数得到 MAC 地址, 保存到数组`macs`中。

```
 for (i = 0; i < master_count; i++) {
        ret = ec_mac_parse(macs[i][0], main_devices[i], 0);

        if (i < backup_count) {
            ret = ec_mac_parse(macs[i][1], backup_devices[i], 1);
        }
    }


```

分配`master_count`个主站对象的内存，调用`ec_master_init()`初始化这些实例。

```
    if (master_count) {
        if (!(masters = kmalloc(sizeof(ec_master_t) * master_count,
                        GFP_KERNEL))) {
            ...
        }
    }

    for (i = 0; i < master_count; i++) {
        ret = ec_master_init(&masters[i], i, macs[i][0], macs[i][1],
                    device_number, class, debug_level);
       ...
    }


```

### 2.1 Master Phases

igh 中，状态机是其核心思想，一切操作基于状态机来执行，对创建的每个 EtherCAT 主站实例都需要经过如下阶段转换 (见图 2.3)，主站各阶段操作如下：

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20200917142438559.png)

**Orphaned phase** 此时主站实例已经分配初始化，正在等待以太网设备连接，即还没有与网卡驱动联系起来，此时无法使用总线通讯。

**Idle phase** 当主站与网卡绑定后，Idle 线程`ec_master_idle_thread`开始运行，主站处于 IDLE 状态,`ec_master_idle_thread`主要完成从站拓扑扫描、配置站点地址等工作。该阶段命令行工具能够访问总线，但无法进行过程数据交换，因为还缺少总线配置。

**Operation phase** 应用程序请求主站提供总线配置并激活主站后，主站处于 **operation** 状态，`ec_master_idle_thread`停止运行，内核线程变为`ec_master_operation_thread`, 之后应用可周期交换过程数据。

具体的后面会说到。

### 2.2 数据报与状态机

继续看 master 初始化代码`ec_master_init`前，我们先了解数据报与状态机的关系，这对后续理解很有帮助。

#### 数据报

EtherCAT 是以[以太网](https://zh.wikipedia.org/wiki/%E4%BB%A5%E5%A4%AA%E7%BD%91)为基础的[现场总线](https://zh.wikipedia.org/wiki/%E7%8F%BE%E5%A0%B4%E7%B8%BD%E7%B7%9A)系统，EtherCAT 使用标准的 IEEE802.3 以太网帧，在主站一侧使用标准的以太网控制器，不需要额外的硬件。并在以太网帧头使用以太网类型 0x88A4 来和其他以太网帧相区别（EtherCAT 数据还可以通过 UDP/IP 来传输，本文已忽略），标准的 IEEE802.3 以太网帧中数据部分为 EtherCAT 的数据，标准的 IEEE802.3 以太网帧与 EtherCAT 数据帧关系如下：

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/ethercat-fram-format.png)

EtherCAT 数据位于以太网帧数据区，EtherCAT 数据由 **EtherCAT 头**和**若干 EtherCAT 数据报文**组成。其中 EtheRCAT 头中记录了 EtherCAT 数据报的长度、和类型，类型为 1 表示与从站通讯。EtherCAT 数据报文内包含多个子报文，每个子报文又由子报文头、数据和 WKC 域组成。子报文结构含义如下。

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/datagram-hy.png)

整个 EtherCAT 网络形成一个环状，主站与从站之间是通过 EtherCAT 数据报来交互，一个 EtherCAT 报文从网卡 TX 发出后，从站 ESC 芯片 EtherCAT 报文进行交换数据，最后该报文回到主站。网上有个经典的 EtherCAT 动态图 (刷新后动态图重新播放).

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/EthercatOperatingPrinciple.svg)

认识 EtherCAT 数据帧结构后，我们看 IgH 内是如何表示一个 EtherCAT 数据报文的？EtherCAT 数据报文在 igh 中用对象 ec_datagram_t 表示。

```
typedef struct {
    struct list_head queue; /**< 发送和接收时插入主站帧队列. */
    struct list_head sent; /**< 已发送数据报的主站列表项. */
    ec_device_index_t device_index; /**< 发送/接收数据报的设备。 */
	
    ec_datagram_type_t type; /**< 帧类型 (APRD, BWR, etc.). */
    uint8_t address[EC_ADDR_LEN]; /**< Recipient address. */
    uint8_t *data; /**< 数据. */
    ec_origin_t data_origin; /**< 数据保存的地方. */
    size_t mem_size; /**< Datagram \a data memory size. */
    size_t data_size; /**< Size of the data in \a data. */
    uint8_t index; /**< Index (set by master). */
    uint16_t working_counter; /**< 工作计数. */
    ec_datagram_state_t state; /**数据帧状态 */
#ifdef EC_HAVE_CYCLES
    cycles_t cycles_sent; /**< Time, 数据报何时发送. */
#endif
    unsigned long jiffies_sent; /**< Jiffies,数据报何时发送. */
#ifdef EC_HAVE_CYCLES
    cycles_t cycles_received; /**< Time, 何时被接收. */
#endif
    unsigned long jiffies_received; /**< Jiffies,何时被接收. */
    unsigned int skip_count; /**< 尚未收到的重新排队数. */
    unsigned long stats_output_jiffies; /**< Last statistics output. */
    char name[EC_DATAGRAM_NAME_SIZE]; /**< Description of the datagram. */
} ec_datagram_t;



```

可以看到上面子报文中各字段大都在 ec_datagram_t 中有表示了, 不过多介绍，其它几个成员简单介绍下，

`device_index`表示这个数据报是属于哪个网卡设备发送接收的，IgH 中一个 master 实例可以使用多个多个网卡设备来发送 / 接收 EtherCAT 数据帧，`device_index`就是表示 master 下的网络设备的。

`data_origin`表示每个 master 管理着多个空闲的 ec_datagram_t，这些 ec_datagram_t 分成了三类，给不同的状态机使用，具体的后文马上会细说，data_origin 就是表示这个 ec_datagram_t 是属于哪类的。

`index`表示该数据报是 EtherCAT 数据区的第 index 个子报文，在 master 发送一个完整的 EtherCAT 数据报时，组装以太网数据帧时使用。

`data`这个指向子报文的数据内存区，由于每个子报文交换数据不同，其大小不同，所以数据区为动态分配,`mem_size`表示的就是分配的大小。

`data_size`表示数据报操作的数据大小，比如该数据报用于读从站的某个寄存器，该值就是这个寄存器的大小。

`jiffies_sent、jiffies_received、cycles_sent、cycles_received`使用不同时钟方式是记录数据帧发送接收时间的，用于统计。

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/datagram.svg)

`ec_datagram_state_t`表示数据报的状态，每个数据报（ec_datagram_t）也是基于状态来处理，有 6 种状态：

*   EC_DATAGRAM_INIT ：数据报已经初始化
*   EC_DATAGRAM_QUEUED ：插入发送队列准备发送
*   EC_DATAGRAM_SENT ：已经发送 (还存在队列中）
*   EC_DATAGRAM_RECEIVED：该数据报已接收，并从发送队列删除
*   EC_DATAGRAM_TIMED_OUT ：该数据报发送后，接收超时，从发送队列删除
*   EC_DATAGRAM_ERROR : 发送和接收过程中出错 (从队列删除)，校验错误、不匹配等。

M 位在 master 发送时组装 EtherCAT 数据帧时确定，接收时也根据该位判断后面还有没有子报文。

数据报对象初始化由函数`ec_datagram_init()`完成：

```
void ec_datagram_init(ec_datagram_t *datagram /**< EtherCAT datagram. */)
{
    INIT_LIST_HEAD(&datagram->queue); // mark as unqueued
    datagram->device_index = EC_DEVICE_MAIN;  /*默认主设备使用*/
    datagram->type = EC_DATAGRAM_NONE;		/*数据报类型*/
    memset(datagram->address, 0x00, EC_ADDR_LEN);  /*数据报地址清零*/
    datagram->data = NULL;
    datagram->data_origin = EC_ORIG_INTERNAL; 	/*默认内部数据*/
    datagram->mem_size = 0;
    datagram->data_size = 0;
    datagram->index = 0x00;
    datagram->working_counter = 0x0000;
    datagram->state = EC_DATAGRAM_INIT; /*初始状态*/
#ifdef EC_HAVE_CYCLES
    datagram->cycles_sent = 0;
#endif
    datagram->jiffies_sent = 0;
#ifdef EC_HAVE_CYCLES
    datagram->cycles_received = 0;
#endif
    datagram->jiffies_received = 0;
    datagram->skip_count = 0;
    datagram->stats_output_jiffies = 0;
    memset(datagram->name, 0x00, EC_DATAGRAM_NAME_SIZE);
}


```

#### 状态机

说完 IgH 数据报对象，我们来看有限状态机（fsm），有限状态机（fsm）：表示有限个状态以及在这些状态之间的转移和动作等行为的数学模型。说起状态机，相信大家大学时候都有用过吧，不管是单片机、FPGA，用它写按键、菜单、协议处理、控制器什么的爽的一塌糊涂。其实我们使用的计算机就是本就是基于状态机作为计算模型的，它对数字系统的设计具有十分重要的作用。另外 Linux TCP 协议也是由状态机实现。

同样 igh 主站内部机制使用有限状态机来实现，IgH 内状态机的基本表示如下：

```
struct ec_fsm_xxx {
    ec_datagram_t *datagram; /**< 主站状态机使用的数据报对象 */
    void (*state)(ec_fsm_master_t *); /**< 状态函数 */
    ....
};


```

IgH EtherCAT 协议栈几乎所有功能通过状态机实现，每个状态机管理着某个对象的状态、功能实现的状态装换，而这些状态转换是基于 EtherCAT 数据报来进行的，如状态机 A0 状态函数填充 datagram，经 EtherCAT 数据帧发出后，经过 slave ESC 处理，回到网卡接收端接收后，交给状态机 A1 状态的下一个状态函数解析处理。所以每个状态机内都包含有指向该状态机操作的数据报对象指针`datagram`和状态执行的状态函数`void (*state)(ec_fsm_master_t *)`；

总结一句话：状态机是根据数据报的状态来执行，**每个状态机都需要操作一个数据报对象**。

现在知道了状态机与数据报的关系，下面介绍 IgH EtherCAT 协议栈中有哪些状态机，及状态机使用的数据报对象是从哪里分配如何管理的。

**master 状态机**

前面说到主站具有的三个阶段，当主站与网卡设备 attach 后进入 Idle phase，处于 Idle phase 后，开始执行主站状态机。

主站状态机包含 1 个主状态机和许多子状态机，matser 状态机主要目的是：

*   **Bus monitoring** 监控 EtherCAT 总线拓扑结构，如果发生改变，则重新扫描。
    
*   **Slave confguration** 监视从站的应用程序层状态。如果从站未处于其应有的状态，则从站将被（重新）配置
    
*   **Request handling** 请求处理（源自应用程序或外部来源），主站任务应该处理异步请求，例如: SII 访问，SDO 访问或类似。
    

主状态机`ec_fsm_master_t`结构如下：

```
struct ec_fsm_master {
    ec_master_t *master; /**< master the FSM runs on */
    ec_datagram_t *datagram; /**< 主站状态机使用的数据报对象 */
    unsigned int retries; /**< retries on datagram timeout. */

    void (*state)(ec_fsm_master_t *); /**< master state function */
    ec_device_index_t dev_idx; /**< Current device index (for scanning etc.).
                                */
    int idle; /**< state machine is in idle phase */
    unsigned long scan_jiffies; /**< beginning of slave scanning */
    uint8_t link_state[EC_MAX_NUM_DEVICES]; /**< Last link state for every
                                              device. */
    unsigned int slaves_responding[EC_MAX_NUM_DEVICES]; /**<每个设备的响应从站数。*/
    unsigned int rescan_required; /**< A bus rescan is required. */
    ec_slave_state_t slave_states[EC_MAX_NUM_DEVICES]; /**< AL states of
                                                         responding slaves for
                                                         every device. */
    ec_slave_t *slave; /**< current slave */
    ec_sii_write_request_t *sii_request; /**< SII write request */
    off_t sii_index; /**< index to SII write request data */
    ec_sdo_request_t *sdo_request; /**< SDO request to process. */

    ec_fsm_coe_t fsm_coe; /**< CoE state machine */
    ec_fsm_soe_t fsm_soe; /**< SoE state machine */
    ec_fsm_pdo_t fsm_pdo; /**< PDO configuration state machine. */
    ec_fsm_change_t fsm_change; /**< State change state machine */
    ec_fsm_slave_config_t fsm_slave_config; /**< slave state machine */
    ec_fsm_slave_scan_t fsm_slave_scan; /**< slave state machine */
    ec_fsm_sii_t fsm_sii; /**< SII state machine */
};


```

可以看到，主站状态机结构下还有很多子状态机，想象一下如果主站的所有功能通过一个状态机来完成，那么这个状态机的状态数量、各状态之间的联系会有多恐怖，复杂性级别将会提高到无法管理的水平。为此，IgH 中，将 EtherCAT 主状态机的某些功能用子状态机完成。这有助于封装相关工作流，并且避免 “状态爆炸” 现象。这样当主站完成 coe 功能时，可以由子状态机 fsm_coe 去完成。具体各功能是如何通过状态机完成的，文章后面会介绍。

**slave 状态机**

slave 状态机管理着每个从站的状态，所以位于从站对象 (ec_slave_t) 内：

```
struct ec_slave
{
    ec_master_t *master; /**< Master owning the slave. */
.....
    ec_fsm_slave_t fsm; /**< Slave state machine. */
.....
};
struct ec_fsm_slave {
    ec_slave_t *slave; /**< slave the FSM runs on */
    struct list_head list; /**< Used for execution list. */
    ec_dict_request_t int_dict_request; /**< Internal dictionary request. */

    void (*state)(ec_fsm_slave_t *, ec_datagram_t *); /**< State function. */
    ec_datagram_t *datagram; /**< Previous state datagram. */
    ec_sdo_request_t *sdo_request; /**< SDO request to process. */
    ec_reg_request_t *reg_request; /**< Register request to process. */
    ec_foe_request_t *foe_request; /**< FoE request to process. */
    off_t foe_index; /**< Index to FoE write request data. */
    ec_soe_request_t *soe_request; /**< SoE request to process. */
    ec_eoe_request_t *eoe_request; /**< EoE request to process. */
    ec_mbg_request_t *mbg_request; /**< MBox Gateway request to process. */
    ec_dict_request_t *dict_request; /**< Dictionary request to process. */

    ec_fsm_coe_t fsm_coe; /**< CoE state machine. */
    ec_fsm_foe_t fsm_foe; /**< FoE state machine. */
    ec_fsm_soe_t fsm_soe; /**< SoE state machine. */
    ec_fsm_eoe_t fsm_eoe; /**< EoE state machine. */
    ec_fsm_mbg_t fsm_mbg; /**< MBox Gateway state machine. */
    ec_fsm_pdo_t fsm_pdo; /**< PDO configuration state machine. */
    ec_fsm_change_t fsm_change; /**< State change state machine */
    ec_fsm_slave_scan_t fsm_slave_scan; /**< slave scan state machine */
    ec_fsm_slave_config_t fsm_slave_config; /**< slave config state machine. */
};


```

slave 状态机和 master 状态机类似，slave 状态机内还包含许多子状态机。slave 状态机主要目的是：

*   主站管理从站状态
*   主站与从站应用层 (AL) 通讯。比如具有 EoE 功能的从站, 主站通过该从站下的子状态机 fsm_eoe 来管理主站与从站应用层的 EOE 通讯。

**数据报对象的管理**

上面简单介绍了 IgH 内的状态机，状态机输入输出的对象是`datagram`，fsm 对象内只有数据报对象的指针，那 fsm 工作过程中的数据报对象从哪里分配？

由于每个循环周期都需要操作数据报对象，IgH 为减少`datagram`的动态分配操作，提高主站性能，在 master 初始化的时候预分配了主站运行需要的所有 datagram 对象。在 master 实例我们可以看到下面的数据报对象：

```
struct ec_master {
    ...
    ec_datagram_t fsm_datagram; /**< Datagram used for state machines. */
    ...
    ec_datagram_t ref_sync_datagram; /**< Datagram used for synchronizing the
                                       reference clock to the master clock.*/
    ec_datagram_t sync_datagram; /**< Datagram used for DC drift
                                   compensation. */
    ec_datagram_t sync_mon_datagram; /**< Datagram used for DC synchronisation
                                       monitoring. */
    ...
    ec_datagram_t ext_datagram_ring[EC_EXT_RING_SIZE]; 
}


```

这些数据报对象都是已经分配内存的，但由于报文不同，报文操作的数据大小不同，所以 datagram 数据区大小随状态机的具体操作而变化，在具体使用时才分配数据区内存。  
以上数据报对象给状态机使用，别忘了还有过程数据也需要数据报对象，所以 IgH 中数据报类型分为以下四类：  
分为三类 (非常重要)：

<table><thead><tr><th><strong>数据报对象</strong></th><th><strong>用途</strong></th></tr></thead><tbody><tr><td>Datagram_pairs</td><td>过程数据报</td></tr><tr><td>fsm_datagram[]</td><td>Fsm_master 及子状态机专用的数据报对象。</td></tr><tr><td>ext_datagram_ring[]</td><td>动态分配给 fsm_slave 及其子 fsm。</td></tr><tr><td>ref_sync_datagram<br>sync_datagram<br>sync64_datagram<br>sync_mon_datagram</td><td>应用专用数据报用于时钟同步。</td></tr></tbody></table>

其中`fsm_datagram`为 master 状态机及 master 下的子状态机执行过程中操作的对象。

`ext_datagram_ring[]`是一个环形队列，当`fsm_slave`从站状态机处于 ready 状态，可以开始处理与 slave 相关请求，如配置、扫描、SDO、PDO 等，这时会从`ext_datagram_ring[]`中给该`fsm_slave`分配一个数据报，并运行`fsm_slave`状态机检查并处理请求。

应用专用数据报用于时钟同步，与时钟强相关，它们比较特殊，它们的数据区大小是恒定的，所以其数据区在主站初始化时就已分配内存，应用调用时直接填数据发送，避免 linux 的内存分配带来时钟的偏差。

数据报数据区（data）内存通过`ec_datagram_prealloc()`来分配.

```
int ec_datagram_prealloc(
        ec_datagram_t *datagram, /**< EtherCAT datagram. */
        size_t size /**< New payload size in bytes. */
        )
{
    if (datagram->data_origin == EC_ORIG_EXTERNAL
            || size <= datagram->mem_size)
        return 0;
	......

    if (!(datagram->data = kmalloc(size, GFP_KERNEL))) {
	......
    }

    datagram->mem_size = size;
    return 0;
}


```

数据区的大小为一个以太网帧中单个 Ethercat 数据报的最大数据大小`EC_MAX_DATA_SIZE`。

```
/** Size of an EtherCAT frame header. */
#define EC_FRAME_HEADER_SIZE 2

/** Size of an EtherCAT datagram header. */
#define EC_DATAGRAM_HEADER_SIZE 10

/** Size of an EtherCAT datagram footer. */
#define EC_DATAGRAM_FOOTER_SIZE 2

/** Size of the EtherCAT address field. */
#define EC_ADDR_LEN 4

/** Resulting maximum data size of a single datagram in a frame. */
#define EC_MAX_DATA_SIZE (ETH_DATA_LEN - EC_FRAME_HEADER_SIZE \
                          - EC_DATAGRAM_HEADER_SIZE - EC_DATAGRAM_FOOTER_SIZE)


```

由于以太网帧的大小有限，因此数据报的最大大小受到限制，即以太网帧长度 1500 - ethercat 头 2byte- ethercat 子数据报报头 10 字节 - WKC 2 字节，如图：

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20210222222428543.png)

如果过程数据镜像的大小超过该限制，就必须发送多个帧，并且必须对映像进行分区以使用多个数据报。 Domain 自动进行管理。

### 2.3 master 状态机及数据报初始化

对状态机及数据报对象有初步认识后，我们回到 ec_master.ko 模块入口函数`ec_init_module()`主站实例初始化`ec_master_init()`，主要完成主站状态机初始化及数据报：

```
    // init state machine datagram
    ec_datagram_init(&master->fsm_datagram); /*初始化数据报对象*/
    snprintf(master->fsm_datagram.name, EC_DATAGRAM_NAME_SIZE, "master-fsm");
    ret = ec_datagram_prealloc(&master->fsm_datagram, EC_MAX_DATA_SIZE);

    // create state machine object
    ec_fsm_master_init(&master->fsm, master, &master->fsm_datagram); /*初始化master fsm*/


```

其中`ec_fsm_master_init`初始化 master fsm 和子状态机，并指定了 master fsm 使用的数据报对象`fsm_datagram`。

```
void ec_fsm_master_init(
        ec_fsm_master_t *fsm, /**< Master state machine. */
        ec_master_t *master, /**< EtherCAT master. */
        ec_datagram_t *datagram /**< Datagram object to use. */
        )
{
    fsm->master = master;
    fsm->datagram = datagram;
    ec_fsm_master_reset(fsm);

    // init sub-state-machines
    ec_fsm_coe_init(&fsm->fsm_coe);
    ec_fsm_soe_init(&fsm->fsm_soe);
    ec_fsm_pdo_init(&fsm->fsm_pdo, &fsm->fsm_coe);
    ec_fsm_change_init(&fsm->fsm_change, fsm->datagram);
    ec_fsm_slave_config_init(&fsm->fsm_slave_config, fsm->datagram,
            &fsm->fsm_change, &fsm->fsm_coe, &fsm->fsm_soe, &fsm->fsm_pdo);
    ec_fsm_slave_scan_init(&fsm->fsm_slave_scan, fsm->datagram,
            &fsm->fsm_slave_config, &fsm->fsm_pdo);
    ec_fsm_sii_init(&fsm->fsm_sii, fsm->datagram);
}


```

**初始化外部数据报队列**

外部数据报队列用于从站状态机, 每个状态机执行期间使用的数据报从该区域分配，下面是初始化 ext_datagram_ring 中每个结构：

```
     for (i = 0; i < EC_EXT_RING_SIZE; i++) {
        ec_datagram_t *datagram = &master->ext_datagram_ring[i];
        ec_datagram_init(datagram);
        snprintf(datagram->name, EC_DATAGRAM_NAME_SIZE, "ext-%u", i);
    }


```

非应用数据报队列链表，如 EOE 数据报会插入该队列后发送。

```
INIT_LIST_HEAD(&master->ext_datagram_queue);


```

同样初始化几个时钟相关数据报对象，它们功能固定，所以数据区大小固定，就不贴代码了，比如 sync_mon_datagram，它的作用是用于同步监控，获取从站系统时间差，所以是一个 BRD 数据报，在此直接将数据报操作偏移地址初始化，使用时能快速填充发送。

```
    ec_datagram_init(&master->sync_mon_datagram);
	......
    ret = ec_datagram_brd(&master->sync_mon_datagram, 0x092c, 4);


```

<table><thead><tr><th>地址</th><th>位</th><th>名称</th><th>描述</th><th>复位值</th></tr></thead><tbody><tr><td>0x092c~0x092F</td><td>0~30</td><td>系统时间差</td><td>本地系统时间副本与参考时钟系统时间值之差</td><td>0</td></tr><tr><td></td><td>31</td><td>符号</td><td>0: 本地系统时间≥参考时钟时间<br>1: 本地系统时间＜参考时钟时间</td><td>0</td></tr></tbody></table>

另外比较重要的是将使用的网卡 MAC 地址放到 macs[] 中，在网卡驱动 probe 过程中根据 MAC 来匹配主站使用哪个网卡。

```
    for (dev_idx = EC_DEVICE_MAIN; dev_idx < EC_MAX_NUM_DEVICES; dev_idx++) {
        master->macs[dev_idx] = NULL;
    }

    master->macs[EC_DEVICE_MAIN] = main_mac;


```

### 2.4 初始化 EtherCAT device

master 协议栈主要完成 EtherCAT 数据报的解析和组装，然后需要再添加 EtherNet 报头和 FCS 组成一个完整的以太网帧，最后通过网卡设备发送出去。为与以太网设备驱动层解耦，igh 使用 ec_device_t 来封装底层以太网设备，一般来说每个 master 只有一个 ec_device_t，这个编译时配置决定，若启用线缆冗余功能，可指定多个网卡设备：

```
struct ec_device
{
    ec_master_t *master; /**< EtherCAT master */
    struct net_device *dev; /**< 使用的网络设备 */
    ec_pollfunc_t poll; /**< pointer to the device's poll function */
    struct module *module; /**< pointer to the device's owning module */
    uint8_t open; /**< true, if the net_device has been opened */
    uint8_t link_state; /**< device link state */
    struct sk_buff *tx_skb[EC_TX_RING_SIZE]; /**< transmit skb ring */
    unsigned int tx_ring_index; /**< last ring entry used to transmit */
#ifdef EC_HAVE_CYCLES
    cycles_t cycles_poll; /**< cycles of last poll */
#endif
#ifdef EC_DEBUG_RING
    struct timeval timeval_poll;
#endif
    unsigned long jiffies_poll; /**< jiffies of last poll */

    // Frame statistics
    u64 tx_count; /**< 发送的帧数 */
    u64 last_tx_count; /**<上次统计周期发送的帧数。 */
    u64 rx_count; /**< 接收的帧数 */
    u64 last_rx_count; /**< 上一个统计周期收到的帧数。 */
	
    u64 tx_bytes; /**< 发送的字节数 */
    u64 last_tx_bytes; /**< 上一个统计周期发送的字节数。 */
    u64 rx_bytes; /**< Number of bytes received. */
    u64 last_rx_bytes; /**< Number of bytes received of last statistics cycle.
                        */
    u64 tx_errors; /**< Number of transmit errors. */
    s32 tx_frame_rates[EC_RATE_COUNT]; /**< Transmit rates in frames/s for
                                         different statistics cycle periods.
                                        */
    s32 rx_frame_rates[EC_RATE_COUNT]; /**< Receive rates in frames/s for
                                         different statistics cycle periods.
                                        */
    s32 tx_byte_rates[EC_RATE_COUNT]; /**< Transmit rates in byte/s for
                                        different statistics cycle periods. */
    s32 rx_byte_rates[EC_RATE_COUNT]; /**< Receive rates in byte/s for
                                        different statistics cycle periods. */

......
};


```

成员`*master`表示改对象属于哪个 master，`*dev`指向使用的以太网设备 net_device，`poll`该网络设备 poll 函数，`tx_skb[]`以太网帧发送缓冲区队列，需要发送的以太网帧会先放到该队里，`tx_ring_index`管理`tx_skb[]`, 以及一些网络统计变量，下面初始化 ec_device_t 对象：

```
/*\master\master.c*/
    for (dev_idx = EC_DEVICE_MAIN; dev_idx < ec_master_num_devices(master);
            dev_idx++) {
        ret = ec_device_init(&master->devices[dev_idx], master);
        if (ret < 0) {
            goto out_clear_devices;
        }
    }
/*\master\device.c*/
int ec_device_init(
        ec_device_t *device, /**< EtherCAT device */
        ec_master_t *master /**< master owning the device */
        )
{
    int ret;
    unsigned int i;
    struct ethhdr *eth;
....

    device->master = master;
    device->dev = NULL;
    device->poll = NULL;
    device->module = NULL;
    device->open = 0;
    device->link_state = 0;
    for (i = 0; i < EC_TX_RING_SIZE; i++) {
        device->tx_skb[i] = NULL;
    }
......

    ec_device_clear_stats(device);
......
    for (i = 0; i < EC_TX_RING_SIZE; i++) {
        if (!(device->tx_skb[i] = dev_alloc_skb(ETH_FRAME_LEN))) {
            ......
        }

        // add Ethernet-II-header
        skb_reserve(device->tx_skb[i], ETH_HLEN);
        eth = (struct ethhdr *) skb_push(device->tx_skb[i], ETH_HLEN);
        eth->h_proto = htons(0x88A4);
        memset(eth->h_dest, 0xFF, ETH_ALEN);
    }
.....
}


```

主要关注分配以太网帧发送队列内存 tx_skb[]，并填充 Ethernet 报头中的以太网类型字段为 0x88A4，目标 MAC 地址 0xFFFFFFFF FFFF，对于源 MAC 地址、sk_buff 所属网络设备、ec_device_t 对象使用的网络设备 net_device，将在网卡驱动初始化与 master 建立联系过程中设置。

### 2.5 设置 IDLE 线程的发送间隔:

```
ec_master_set_send_interval(master, 1000000 / HZ);


```

根据网卡速率计算：

```
void ec_master_set_send_interval(
        ec_master_t *master, /**< EtherCAT master */
        unsigned int send_interval /**< Send interval */
        )
{
    master->send_interval = send_interval;  //发送间隔 us
    master->max_queue_size =
        (send_interval * 1000) / EC_BYTE_TRANSMISSION_TIME_NS;
    master->max_queue_size -= master->max_queue_size / 10;
}


```

100Mbps 网卡发送一字节数据需要的时间 EC_BYTE_TRANSMISSION_TIME_NS： `1/(100 MBit/s / 8 bit/byte) = 80 ns/byte`.

### 2.6 初始化字符设备

由于主站位于内核空间，用户空间应用与主站交互通过字符设备来交互；  
创建普通字符设备，给普通 linux 应用和 Ethercat tool 使用。若使用 xenomai 或 RTAI，则再创建实时字符设备，提供给实时应用使用。

```
	......
	master->class_device = device_create(class, NULL,
            MKDEV(MAJOR(device_number), master->index), NULL,
            "EtherCAT%u", master->index);
    ......
#ifdef EC_RTDM
    // init RTDM device
    ret = ec_rtdm_dev_init(&master->rtdm_dev, master);
   ...
#endif


```

到这里明白了 IgH 中的状态机与数据报之间的关系，主站对象也创建好了，但是主站还没有网卡设备与之关联，主站也还没有工作，下面简单看一下 ecdev_offer 流程。

关于网卡驱动代码详细解析推荐这两篇文章：

[Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/)

[Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/)

## 3 网卡

1.  网卡 probe
    
    ```
    static int igb_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
    {	
        ......
    	adapter->ecdev = ecdev_offer(netdev, ec_poll, THIS_MODULE);
    	if (adapter->ecdev) {	/*注册打开ec_net设备*/
    		err = ecdev_open(adapter->ecdev);
    		.....
    		adapter->ec_watchdog_jiffies = jiffies;
    	} else { /*注册普通网络设备*/
    		......
    		err = register_netdev(netdev);
    		......
    	}
        ......
    }
    
    
    ```
    
2.  给主站提供网络设备：`ecdev_offer`
    
    根据 MAC 地址找到 master 下的`ec_device_t`对象
    
    ```
        device->dev = net_dev;
        device->poll = poll;
        device->module = module;
    
    
    ```
    
    上面我们只设置了`ec_device_t->tx_skb[]`中`sk_buff`的以太网类型和目的地址，现在继续填充源 MAC 地址为网卡的 MAC 地址、sk_buff 所属的 net_device：
    
    ```
        for (i = 0; i < EC_TX_RING_SIZE; i++) {
            device->tx_skb[i]->dev = net_dev;
            eth = (struct ethhdr *) (device->tx_skb[i]->data);
            memcpy(eth->h_source, net_dev->dev_addr, ETH_ALEN);
        }
    
    
    ```
    
3.  调用网络设备接口打开网络设备
    

```
int ec_device_open(
        ec_device_t *device /**< EtherCAT device */
        )
{
    int ret;
.....
    ret = device->dev->open(device->dev);
    if (!ret)
        device->open = 1;
....
    return ret;
}


```

4.  当 master 下的所有的网络设备都`open`后，master 从 **ORPHANED** 转到 **IDLE** 阶段

```
int ec_master_enter_idle_phase(
        ec_master_t *master /**< EtherCAT master */
        )
{
    int ret;
    ec_device_index_t dev_idx;
	......
    master->send_cb = ec_master_internal_send_cb;
    master->receive_cb = ec_master_internal_receive_cb;
    master->cb_data = master;

    master->phase = EC_IDLE;  /*更新master状态*/

    // reset number of responding slaves to trigger scanning
    for (dev_idx = EC_DEVICE_MAIN; dev_idx < ec_master_num_devices(master);
            dev_idx++) {
        master->fsm.slaves_responding[dev_idx] = 0;
    }

    ret = ec_master_nrthread_start(master, ec_master_idle_thread,
            "EtherCAT-IDLE");
    ....
    return ret;
}


```

其中主要设置 master 发送和接收回调函数，应用通过发送和接收数据时，将通过这两接口直接发送和接收。创建 master **idle** 线程`ec_master_idle_thread`。

## 4 IDLE 阶段内核线程

综上，状态机操作对象是 datagram，datagram 发送出去后回到主站交给状态机的下一个状态处理，所以主站需要循环地执行状态机、发送 EtherCAT 数据帧、接收 EtherCAT 数据帧、执行状态机、发送 EtherCAT 数据帧、…… 来驱动状态机运行，这个循环由内核线程来完成。

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20210523100034721.png)

当主站与网卡绑定后，应用还没有请求主站，主站处于 IDLE 状态，这时循环由内核线程`ec_master_idle_thread`来完成，主要完成从站拓扑扫描、配置站点地址等工作。

```
static int ec_master_idle_thread(void *priv_data)
{
    ec_master_t *master = (ec_master_t *) priv_data;
    int fsm_exec;
#ifdef EC_USE_HRTIMER
    size_t sent_bytes;
#endif

    // send interval in IDLE phase
    ec_master_set_send_interval(master, 250000 / HZ);
    
    while (!kthread_should_stop()) {
        // receive
        ecrt_master_receive(master);  /*接收上个循环发送的数据帧*/
		......
        // execute master & slave state machines
        ......
        fsm_exec = ec_fsm_master_exec(&master->fsm); /*执行master状态机*/

        ec_master_exec_slave_fsms(master); /*为从站状态机分配datagram，并执行从站状态机*/
		......
        if (fsm_exec) {
            ec_master_queue_datagram(master, &master->fsm_datagram); /*将master状态机处理的datagram插入发送链表*/
        }
        // send
        ecrt_master_send(master); /*组装以太网帧并调用网卡发送*/
        
        sent_bytes = master->devices[EC_DEVICE_MAIN].tx_skb[
            master->devices[EC_DEVICE_MAIN].tx_ring_index]->len;
        up(&master->io_sem);

        if (ec_fsm_master_idle(&master->fsm)) {
            ec_master_nanosleep(master->send_interval * 1000);
            set_current_state(TASK_INTERRUPTIBLE);
            schedule_timeout(1);
        } else {
            ec_master_nanosleep(sent_bytes * EC_BYTE_TRANSMISSION_TIME_NS);
        }
    }

    EC_MASTER_DBG(master, 1, "Master IDLE thread exiting...\n");

    return 0;
}



```

整个过程简单概述如下。

### 4.1 数据报发送

下面介绍 IgH 中状态机处理后数据报的发送流程（`ecrt_master_send()`）。

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20210523100210574.png)

master 使用一个链表`datagram_queue`来管理要发送的子报文对象 datagram，需要发送的子报文对象会先插入该链表中，统一发送时，分配一个`sock_buff`，从`datagram_queue`上取出报文对象，设置`index`（`index`是发送后接收回来与原报文对应的标识之一），将一个个报文对象按 EtherCAT 数据帧结构填充到`sock_buff`中，最后通过网卡设备驱动函数`hard_start_xmit`，将`sock_buff`从网卡发送出去。

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20210523100225964.png)

### 4.2 数据报接收

![](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/image-20210523100246660.png)

接收数据时，通过网卡设备驱动`ec_poll`函数取出 Packet 得到以太网数据，然后解析其中的 EtherCAT 数据帧，解析流程如下：

1.  得到子报文`index`，遍历发送链表`datagram_queue`，**找到`index`对应的 datagram**。
    
2.  将子报文数据拷贝到`datagram`数据区。
    
3.  将以太网帧内子报文中的 WKC 值复制到`datagram`中的 WKC。
    
4.  将`datagram`从链表`datagram_queue`删除。
    
5.  根据子报文头 M 位判断还有没有子报文，有则跳转 1 继续处理下一个子报文，否则完成接收。
    

接收完成后，进入下一个循环，内核线程运行状态机或周期应用进行下一个周期，处理接收的 Ethercat 报文。

先简单介绍到这，敬请关注后续文章。。。。