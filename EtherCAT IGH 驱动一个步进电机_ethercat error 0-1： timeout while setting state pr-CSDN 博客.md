---
url: https://blog.csdn.net/u014077947/article/details/127465402
title: EtherCAT IGH 驱动一个步进电机_ethercat error 0-1: timeout while setting state pr-CSDN 博客
date: 2025-03-18 12:48:39
tag: 
summary: 
---
## EtherCAT IGH 驱动一个步进电机


在学习 EtherCAT 的过程主要是参照官方的一些文档，以及网上的一些资料。在把主站编译好之后，心情就很激动就想去驱动一个电机。然后就去上网想去买一套伺服驱动器和电机，看了看价格，想了想还是买一套步进驱动器和步进电机吧，最后选了一套杰美康的步进驱动器和步进电机，以及明纬的开关电源。

**参考例程**

*   EtherCAT 主站源码中的 example/user
*   EtherCAT 主站源码中的 example/dc_user

### 1、程序源代码

```c
#include <errno.h>
#include <malloc.h>
#include <sched.h> /* sched_setscheduler() */
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/resource.h>
#include <sys/time.h>
#include <sys/types.h>
#include <time.h>
#include <unistd.h>

/****************************************************************************/

#include "ecrt.h"

/****************************************************************************/

// Application parameters
#define FREQUENCY 1000
#define CLOCK_TO_USE CLOCK_MONOTONIC
#define MEASURE_TIMING

#define TARGET_POSITION 0 /*target position*/
// 杰美康 2DM522-EC 步进驱动器的 CSP模式的 控制模式为 8
#define PROFILE_CSP 8/*Operation mode for 0x6060:0*/

/****************************************************************************/

#define NSEC_PER_SEC (1000000000L)
#define PERIOD_NS (NSEC_PER_SEC / FREQUENCY) /*本次设置周期PERIOD_NS为1ms*/

#define DIFF_NS(A, B)                                                          \
  (((B).tv_sec - (A).tv_sec) * NSEC_PER_SEC + (B).tv_nsec - (A).tv_nsec)

#define TIMESPEC2NS(T) ((uint64_t)(T).tv_sec * NSEC_PER_SEC + (T).tv_nsec)

/****************************************************************************/

// EtherCAT
static ec_master_t *master = NULL;
static ec_master_state_t master_state = {};

static ec_domain_t *domain1 = NULL;
static ec_domain_state_t domain1_state = {};

static ec_slave_config_t *sc = NULL;
static ec_slave_config_state_t sc_state = {};

/****************************************************************************/

// process data
static uint8_t *domain1_pd = NULL;

#define DM_522EC 0, 0 /*EtherCAT address on the bus*/
#define VID_PID 0x66786328, 0x20190303 /*Vendor ID, product code*/

/*Offsets for PDO entries*/
static struct {
  unsigned int operation_mode;
  unsigned int ctrl_word;
  unsigned int target_position;
  unsigned int target_velocity;
  // 单位 %，额定转矩的百分比
  unsigned int target_torque;
  // 转矩斜率，该值单位为 ‰，该参数描述了转矩变化率，单位为每秒额定转矩矩的千分之一。
  unsigned int torque_slope;
  // 加速度
  unsigned int target_p_acc;
  // 减速度
  unsigned int target_n_acc;
  unsigned int status_word;
  unsigned int current_velocity;
  unsigned int current_position;
  // 模式显示
  unsigned int operation_mode_display;
} offset;

const static ec_pdo_entry_reg_t domain1_regs[] = {
    {DM_522EC, VID_PID, 0x6040, 0, &offset.ctrl_word},
    {DM_522EC, VID_PID, 0x6060, 0, &offset.operation_mode},
    {DM_522EC, VID_PID, 0x607A, 0, &offset.target_position},
    {DM_522EC, VID_PID, 0x6041, 0, &offset.status_word},
    {DM_522EC, VID_PID, 0x606C, 0, &offset.current_velocity},
    {DM_522EC, VID_PID, 0x6064, 0, &offset.current_position},
    {}};

/***************************************************************************/
/*Config PDOs*/
static ec_pdo_entry_info_t device_pdo_entries[] = {
    /*RxPdo 0x1600*/
    {0x6040, 0x00, 16},
    {0x6060, 0x00, 8},
    {0x607A, 0x00, 32},
    /*TxPdo 0x1A00*/
    {0x6041, 0x00, 16},
    {0x606C, 0x00, 32},
    {0x6064, 0x00, 32}};


static ec_pdo_info_t device_pdos[] = {
    // RxPdo，指示有多少个 RXPdo，从device_pdo_entries 可以看到有 3个 RXPdo
    {0x1600, 3, device_pdo_entries + 0},
    // TxPdo，指示有多少个 TXPdo，从device_pdo_entries 可以看到有 3个 TXPdo，第二个 3 是指示 RxPdo 的数量
    {0x1A00, 3, device_pdo_entries + 3}};

// 关闭关门狗
// static ec_sync_info_t device_syncs[] = {
//     {0, EC_DIR_OUTPUT, 0, NULL, EC_WD_DISABLE},
//     {1, EC_DIR_INPUT, 0, NULL, EC_WD_DISABLE},
//     {2, EC_DIR_OUTPUT, 1, device_pdos + 0, EC_WD_DISABLE},
//     {3, EC_DIR_INPUT, 1, device_pdos + 1, EC_WD_DISABLE},
//     {0xFF}};

// 开启看门狗
static ec_sync_info_t device_syncs[] = {
    {0, EC_DIR_OUTPUT, 0, NULL, EC_WD_ENABLE},
    {1, EC_DIR_INPUT, 0, NULL, EC_WD_ENABLE},
    {2, EC_DIR_OUTPUT, 1, device_pdos + 0, EC_WD_ENABLE},
    {3, EC_DIR_INPUT, 1, device_pdos + 1, EC_WD_ENABLE},
    {0xFF}};


static unsigned int counter = 0;
static unsigned int blink = 0;
static unsigned int sync_ref_counter = 0;
const struct timespec cycletime = {0, PERIOD_NS};

/*****************************************************************************/

// 两个时间相加
struct timespec timespec_add(struct timespec time1, struct timespec time2) {
  struct timespec result;

  if ((time1.tv_nsec + time2.tv_nsec) >= NSEC_PER_SEC) {
    result.tv_sec = time1.tv_sec + time2.tv_sec + 1;
    result.tv_nsec = time1.tv_nsec + time2.tv_nsec - NSEC_PER_SEC;
  } else {
    result.tv_sec = time1.tv_sec + time2.tv_sec;
    result.tv_nsec = time1.tv_nsec + time2.tv_nsec;
  }

  return result;
}

/*****************************************************************************/

void check_domain1_state(void) {
  ec_domain_state_t ds;

  ecrt_domain_state(domain1, &ds);

  if (ds.working_counter != domain1_state.working_counter)
    printf("Domain1: WC %u.\n", ds.working_counter);
  if (ds.wc_state != domain1_state.wc_state)
    printf("Domain1: State %u.\n", ds.wc_state);

  domain1_state = ds;
}

/*****************************************************************************/

void check_master_state(void) {
  ec_master_state_t ms;

  ecrt_master_state(master, &ms);

  if (ms.slaves_responding != master_state.slaves_responding)
    printf("%u slave(s).\n", ms.slaves_responding);
  if (ms.al_states != master_state.al_states)
    printf("AL states: 0x%02X.\n", ms.al_states);
  if (ms.link_up != master_state.link_up)
    printf("Link is %s.\n", ms.link_up ? "up" : "down");

  master_state = ms;
}

/****************************************************************************/

void check_slave_config_states(void) {
  ec_slave_config_state_t s;
  ecrt_slave_config_state(sc, &s);
  if (s.al_state != sc_state.al_state) {
    printf("slave: State 0x%02X.\n", s.al_state);
  }
  if (s.online != sc_state.online) {
    printf("slave: %s.\n", s.online ? "online" : "offline");
  }
  if (s.operational != sc_state.operational) {
    printf("slave: %soperational.\n", s.operational ? "" : "Not ");
  }
  sc_state = s;
}

/****************************************************************************/

void cyclic_task() {
  static uint16_t command = 0x004F; //用来帮助判断状态字的值
  uint16_t status;                  //用以存放当前读取到的伺服状态

  struct timespec wakeupTime, time;
#ifdef MEASURE_TIMING
  struct timespec startTime, endTime, lastStartTime = {};
  uint32_t period_ns = 0, exec_ns = 0, latency_ns = 0, latency_min_ns = 0,
           latency_max_ns = 0, period_min_ns = 0, period_max_ns = 0,
           exec_min_ns = 0, exec_max_ns = 0;
#endif

  // 获取当前的系统时间
  // CLOCK_TO_USE = CLOCK_MONOTONIC
  // monotonic time 的字面意思是单调时间，实际上指的是系统启动之后所流逝的时间，这是由变量 jiffies 来记录的，当系统每次启动时，jiffies 被初始化为 0，在每一个 timer interrupt 到来时，变量 jiffies 就加上 1，因此这个变量代表着系统启动后的流逝 tick 数。jiffies 一定是单调增加的，因为时间不可逆。
  clock_gettime(CLOCK_TO_USE, &wakeupTime);

  // 每 1 ms 进行一次循环
  while (1) {

    // 因为下面的 clock_nanosleep 延时函数需要用到一个绝对时间，因此需要用上面的的clock_gettime 获取当前时间 ,然后在添加睡眠间隔.
    // 将 wakeupTime 加上 cycletime（当前值为 1ms），然后再赋值给 wakeupTime
    wakeupTime = timespec_add(wakeupTime, cycletime);

    // TIMER_ABSTIME： 代表是采用的绝对时间
    // 最后一个参数 rept的意思： 该函数将会使得调用进程处于挂起状态，直到请求的时间到达或者是被信号中断，参数reqtp指定了需要睡眠的秒数与纳秒数。
    // 如果在睡眠中途被信号中断，且进程没有终止的话，timespec结构指针remtp指向的结构将保存剩余的睡眠时间，如果我们对于未睡眠时间不感兴趣的话，我们可以把这一时间设置为NULL.
    // 如果系统不支持纳秒时间精度的话，请求时间将被向上取整，因为函数nanosleep并不涉及任何信号的产生，我们可以放心大胆地使用它而不用担心与其他函数相互影响。
    clock_nanosleep(CLOCK_TO_USE, TIMER_ABSTIME, &wakeupTime, NULL);

    // Write application time to master
    // It is a good idea to use the target time (not the measured time) as
    // application time, because it is more stable.
    ecrt_master_application_time(master, TIMESPEC2NS(wakeupTime));

#ifdef MEASURE_TIMING
    clock_gettime(CLOCK_TO_USE, &startTime);
    latency_ns = DIFF_NS(wakeupTime, startTime);
    period_ns = DIFF_NS(lastStartTime, startTime);
    exec_ns = DIFF_NS(lastStartTime, endTime);
    lastStartTime = startTime;

    if (latency_ns > latency_max_ns) {
      latency_max_ns = latency_ns;
    }
    if (latency_ns < latency_min_ns) {
      latency_min_ns = latency_ns;
    }
    if (period_ns > period_max_ns) {
      period_max_ns = period_ns;
    }
    if (period_ns < period_min_ns) {
      period_min_ns = period_ns;
    }
    if (exec_ns > exec_max_ns) {
      exec_max_ns = exec_ns;
    }
    if (exec_ns < exec_min_ns) {
      exec_min_ns = exec_ns;
    }
#endif

    // receive process data
    ecrt_master_receive(master);
    ecrt_domain_process(domain1);

    // check process data state (optional)
    check_domain1_state();

    // 1s 去更新一次状态
    if (counter) {
      counter--;
    } else { // do this at 1 Hz
      // FREQUENCY = 1000
      counter = FREQUENCY;

      // check for master state (optional)
      check_master_state();
      // check for slave configuration state(s)
      check_slave_config_states();

#ifdef MEASURE_TIMING
      // output timing stats
      printf("period     %10u ... %10u\n", period_min_ns, period_max_ns);
      printf("exec       %10u ... %10u\n", exec_min_ns, exec_max_ns);
      printf("latency    %10u ... %10u\n", latency_min_ns, latency_max_ns);
      period_max_ns = 0;
      period_min_ns = 0xffffffff;
      exec_max_ns = 0;
      exec_min_ns = 0xffffffff;
      latency_max_ns = 0;
      latency_min_ns = 0xffffffff;
#endif

      // calculate new process data
      // TODO: 这个变量当前并没有被使用
      blink = !blink;
    }
    /*Read state*/
    status = EC_READ_U16(domain1_pd + offset.status_word); //读取状态字

    // write process data
    // DS402 CANOpen over EtherCAT status machine
    // 控制字的切换用来设置不同的电机状态
    if ((status & command) == 0x0040) {
      EC_WRITE_U16(domain1_pd + offset.ctrl_word, 0x0006);
      EC_WRITE_S8(domain1_pd + offset.operation_mode, PROFILE_CSP);
      // set control mode
      command = 0x006F;
    }

    else if ((status & command) == 0x0021) {
      EC_WRITE_U16(domain1_pd + offset.ctrl_word, 0x0007);
      command = 0x006F;
    }

    else if ((status & command) == 0x0023) {
      EC_WRITE_U16(domain1_pd + offset.ctrl_word, 0x000f);
      command = 0x006F;
    }
    // operation enabled

    else if ((status & command) == 0x0027) {
      EC_WRITE_U16(domain1_pd + offset.ctrl_word, 0x001f);
    }

    printf("slave: (status & command) 0x%04X.\n", (status & command));
    printf("sync_ref_counter %d.\n", sync_ref_counter);

    // 每两个循环周期会去同步一次数据，填充 PDO 数据
    if (sync_ref_counter) {
      sync_ref_counter--;
    } else {
      sync_ref_counter = 1; // sync every cycle

      clock_gettime(CLOCK_TO_USE, &time);

      // ecrt_master_sync_reference_clock: 这个函数将最近一次从ecrt_master_application_time传入的时间戳发送给参考时钟。
      // 这就是官方example中除了rtai_rtdm_dc这个例子使用的默认dc同步方式，即所谓的以主站作为参考时钟的方式。

      // ecrt_master_sync_reference_clock_to: 将指定的时间发送给参考时钟
      ecrt_master_sync_reference_clock_to(master, TIMESPEC2NS(time));

      // 让电机运动的代码
      // 说明：定义一个当前位置actual_position 和目标位置target_position
      // 在周期任务里，判断当电机切换为使能状态后，读取电机当前位置，并把这个数值加10作为目标位置写入电机。
      if ((status & command) == 0x0027) //确认为OP状态，电机使能
      {
        // 不读取当前的电机位置的话，直接发送电机位置，可能会离电机当前的位置很远，电机需要一个很大的力才能到达那个位置。
        // 可能电机容易报过流故障。
        int actual_position = EC_READ_S32(domain1_pd + offset.current_position); //读取当前位置
        printf("slave: position 0x%04X.\n", EC_READ_U16(domain1_pd + offset.current_position));
        int target_position = actual_position + 10; //把当前位置+10 赋值为下一目标位置
        EC_WRITE_S32(domain1_pd + offset.target_position, target_position); //把目标位置写入对象字典
      }
    }

    //将DC时钟漂移补偿数据报排队发送，让所有从站时钟与基准时钟同步。 
    ecrt_master_sync_slave_clocks(master);

    // send process data
    // 将所有的需要发送的数据报先存放在数据域中
    // 然后在每一个时钟周期中发送数据
    ecrt_domain_queue(domain1);
    ecrt_master_send(master);

#ifdef MEASURE_TIMING
    clock_gettime(CLOCK_TO_USE, &endTime);
#endif
  }
}

/****************************************************************************/

int main(int argc, char **argv) {
  if (mlockall(MCL_CURRENT | MCL_FUTURE) == -1) {
    perror("mlockall failed");
    return -1;
  }

  master = ecrt_request_master(0);
  if (!master)
    return -1;

  domain1 = ecrt_master_create_domain(master);
  if (!domain1)
    return -1;

  // Create configuration for bus coupler
  sc = ecrt_master_slave_config(master, DM_522EC, VID_PID);
  if (!sc)
    return -1;

  printf("Configuring PDOs...\n");
  if (ecrt_slave_config_pdos(sc, EC_END, device_syncs)) {
    fprintf(stderr, "Failed to configure slave PDOs!\n");
    exit(EXIT_FAILURE);
  } else {
    printf("*Success to configuring slave PDOs*\n");
  }

  if (ecrt_domain_reg_pdo_entry_list(domain1, domain1_regs)) {
    fprintf(stderr, "PDO entry registration failed!\n");
    exit(EXIT_FAILURE);
  }

  // 配置从站 sync0 和 sync1 信号
  // 最后四个参数表示设置 sync 0和1 的周期和相应的偏移量，单位都是 ns。
  // sync0_cycle即为sync0的循环周期，和主栈的周期任务的循环周期保持一致。这个是很重要的一点，需要注意。

  // 1、查看 esi 文件，不支持 sync1 同步，所以需要设置成 0x0300
  // 0x300: 0x0981的 0,1 位 置 1,其他位 置 0,表示激活运行周期， 激活 sync0
  // 0x700: 0x0981的 0,1,2 位 置 1,其他位 置 0,表示激活运行周期， 激活 sync0，和 sync1 
  // 2、查看 ethercat upload 0x1c32 0x0004 中 bit5-6 为零，代表不支持 shift time，因此第三个参数设置为 0
  // 3、一般不使用sync1同步信号，最后两个参数可设置为0。
  // 同步周期设置成 1ms
  ecrt_slave_config_dc(sc, 0x0300, 1000000, 0, 0, 0);

  printf("Activating master...\n");
  if (ecrt_master_activate(master))
    return -1;

  if (!(domain1_pd = ecrt_domain_data(domain1))) {
    return -1;
  }

  /* Set priority */

  struct sched_param param = {};
  param.sched_priority = sched_get_priority_max(SCHED_FIFO);

  printf("Using priority %i.", param.sched_priority);
  if (sched_setscheduler(0, SCHED_FIFO, ¶m) == -1) {
    perror("sched_setscheduler failed");
  }

  printf("Starting cyclic function.\n");
  cyclic_task();

  return 0;
}


```

### 2、调试过程中遇到的问题

在调试过程中遇到了一些问题，然后和群里的一些大神同行交流了一下，现总结记录一下。

#### 2.1、如何更好更快的定位 EtherCAT 的问题

我们在驱动电机的时候，可能会遇到各种问题，那么如何更好更快的定位遇到的问题，以及如何更好的向他人进行请教呢？那么就下面这些指令了。


```sh
sudo ehtercat debug 1
```


这个是 EtherCAT 的命令行工具，用以设置显示 EtherCAT 主站程序运行过程中的 debug 日志信息。这个指令设置之后，不会有任何信息输出的，需要和 **sudo dmesg -w** 进行配合使用，才能看到需要的日志信息。

**这个指令后面的参数不建议设置成 2，因为设置成 2 的时候会在终端打印出每一个 EtherAT 的数据帧，而这有可能会造成 PC 的界面卡死的，需要慎用。**


```sh
sudo dmesg -w
```


在下载和安装 EtherCAT 的时候，我们知道 EtherCAT 是安装在 [Liunx](https://so.csdn.net/so/search?q=Liunx&spm=1001.2101.3001.7020) 内核中的。Linux 内核是操作系统的核心，它控制对系统资源（例如：CPU、I/O 设备、物理内存和文件系统）的访问。在引导过程中以及系统运行时，内核会将各种消息写入内核环形缓冲区。这些消息包括有关系统操作的各种信息。内核环形缓冲区是物理内存的一部分，用于保存内核的日志消息。它具有固定的大小，这意味着一旦缓冲区已满，较旧的日志记录将被覆盖。[dmesg](https://so.csdn.net/so/search?q=dmesg&spm=1001.2101.3001.7020) 命令行用于 Linux 和其他类似 Unix 的操作系统中打印和控制内核环形缓冲区。对于检查内核启动消息和调试与硬件相关的问题很有用。

```sh
sudo dmesg

```

默认情况下，所有用户都可以运行 dmesg 命令。但是，在某些系统上，非 root 用户可能会限制对 dmesg 的访问。在这种情况下，调用 dmesg` 时您将收到如下错误消息：  
dmesg: read kernel buffer failed: Operation not permitted。  
内核参数 kernel.dmesg_restrict 指定非特权用户是否可以使用 dmesg 查看来自内核日志缓冲区的消息。要删除限制，请将其设置为零：

```sh
$ sudo dmesg -w

```

用以设置实时观看 `dmesg` 命令的输出。

```sh
$ sudo dmesg -H

```

用来设置输出更容易读的结果。

```sh
$ sudo dmesg -T

```

用以设置输出易读的时间戳。

```sh
$ sudo dmesg -H -T

```

用来设置输出更容易读的结果和易读的时间戳。

#### 2.2、Failed to open `/dev/EtherCAT0`

如果在运行程序的时候，报错如下的话：

```
Requesting master.....
Failed to open /dev/EtherCAT0: No such file or directory

```

那么应该就是你没有启动 EtherCAT 主站，这个时候需要在另外一个终端开启 EtherCAT 主站。

```sh
sudo /etc/init.d/ethercat start
Starting EtherCAT master 1.6.0-rc1  done

```

#### 2.3、Failed to set SAFEOP state, slave refused state chang (PREOP + ERROR)

```
EtherCAT ERROR 0-0: Failed to set SAFEOP state, slave refused state chang (PREOP + ERROR).
EhterCAT ERROR 0-0: AL status messgae 0x001F: "Invalid watchdog configuration".
EtherCAT 0-0： Acknowlwdged state PREOP.

```

群里的大神说是因为有的驱动器必须开启看门狗，但是 EtherCAT 里面的 user 和 dc_user 里面的例子都是默认关闭看门狗的。

这边直接在程序的参数定义的时候使用 **EC_WD_ENABLE** 代替 **EC_WD_DISABLE** 就可以了。我这边开启看门狗之后就可以进入到 SAFEOP 状态了。

**群里也有人说要设置时针模式和频率，要和设备说明书一致。然后我就去问厂家，厂家说这个和主站设置的一致。也有可能有的驱动器设备会说明这个参数，暂时还不清楚。当时开启看门狗之后没有出现这个问题，我就没去设置了。嗯嗯，最后还是因为这个造成的。**

```c
static ec_sync_info_t device_syncs[] = {
    {0, EC_DIR_OUTPUT, 0, NULL, EC_WD_ENABLE},
    {1, EC_DIR_INPUT, 0, NULL, EC_WD_ENABLE},
    {2, EC_DIR_OUTPUT, 1, device_pdos + 0, EC_WD_ENABLE},
    {3, EC_DIR_INPUT, 1, device_pdos + 1, EC_WD_ENABLE},
    {0xFF}};

```

#### 2.4、驱动器报过流故障

^40650b

这边在开启看门狗之后，利用 **EtherCAT slaves** 查看到驱动器已经进入到了 OP 状态，但是驱动器会报过流故障。后面问过了厂家的技术，说是没有设置同步周期时间导致的，没有设置同步周期时间，当控制字 0x6040 从 0x0007 变换为 0x000f 的时候驱动器就会报过流故障，也就是给电机上电使能的就会报过流故障。

#### 2.5、Timeout while setting state OP

这边在听取厂家的反馈之后，开始设置同步周期时间，然后网上找了一些例程，也有可能是我没找到，或者没理解大神的意图，然后我就加了这么一句话，运行的时候就一直报错进行 OP 状态超时。

```c
  ecrt_slave_config_dc(sc, 0x0300, 1000000, 0, 0, 0);
```

群里有人说是因为**有的驱动器的 0x1600 的 PDO 需要配置两个 8 位的 PDO，不然可能就会出现这种超时的问题**。额外增加的那个 8 位 PDO 可以不用发数据，然后我就配了一个 0x607B：0x0000，结果发现还是报这个错。

**最后还是参考 EtherCAT `example/dc_user/main.c` 中的例程，才没有报这个错误。**

**关于 ecrt_slave_config_dc 想再说两点**：

*   有的驱动器只支持 sync0，不支持 sync1，因此 中的第二个参数只能设置成 0x0300。
*   有的驱动器同步模式不支持 shift time, 因此第三个参数也只能设置为 0。  
    查看 ethercat upload 0x1c32 0x0004 中 bit5-6 为零，代表不支持 shift time。
*   如果不用 sync1 的话，那么最后两个参数设置为 0。

同步周期的设置一般为 500us、1ms、2ms、4ms 等。

#### 2.6、如何添加自己需要的打印信息

在学习 EtherCAT 或者调试的时候，想看一些函数的运行过程中的参数值，或者调用顺序什么的，一般第一想到的就是加上打印消息，然后重新编译。

**但是 EtherCAT 主站在编译之后，一般需要再次配置西相关文件，填入网卡设备信息。具体可参照我的《EtherCAT IGH 的下载和编译》 里面的 [7、修改源代码之后如何重新编译](https://blog.csdn.net/u014077947/article/details/127402412?spm=1001.2014.3001.5501)**

#### 2.7 如何获取 Vendor ID 和 product code

在配置从站的时候调用 ecrt_master_slave_config 这个函数的时候需要填入 Vendor ID 和 product code 参数，那么这个参数又是从哪里获取到的呢？

这个参数在从站的 ESI .xml 从站描述文件中是有写的。所以就有如下两种方法进行获取了。

*   当然就是直接从 .xml 文件中查找获取得到了。
    
*   利用 EtherCAT 命令行 **sudo ethercat xml** 进行获取。
    
    如下图中 Vendor/ID 就是 Vendor ID 参数，但是这个是 10 进制表示。ProductCode 就是 product code 参数，这个是 16 进制表示的。
    
    ```
    @:~$ ethercat xml
    <?xml version="1.0" ?>
    <EtherCATInfo>
    <!-- Slave 0 -->
    <Vendor>
        <Id>1717995656</Id>
    </Vendor>
    <Descriptions>
        <Devices>
        <Device>
            <Type ProductCode="#x20190303" RevisionNo="#x20190620">2DM522-EC</Type>
            <Name><![CDATA[2DM522-EC]]></Name>
    ....
    ....
    
    
    ```
    

#### 2.8 同步周期时间和通信周期时间

^2ef7d2

在 EtherCAT 程序中，我们一般会设置一个 while 循环去发送数据帧，那么每一次发送数据帧的数据是多少呢？这个和我们设置的同步周期时间有什么关系呢？

同步周期需要和主栈的周期任务的循环周期保持一致。这个是很重要的一点，需要注意。

在这个例程中同步周期设置的是 1ms, while 中的等待时间设置的也是 1ms。

同步周期时间： 1000000ns = 1ms

```
ecrt_slave_config_dc(sc, 0x0300, 1000000, 0, 0, 0);

```

周期任务循环时间： cycletime = 1000000ns = 1ms

```
wakeupTime = timespec_add(wakeupTime, cycletime);
clock_nanosleep(CLOCK_TO_USE, TIMER_ABSTIME, &wakeupTime, NULL);

```

这个设置的不匹配也导致 **### 2.5、Timeout while setting state OP** 的问题。