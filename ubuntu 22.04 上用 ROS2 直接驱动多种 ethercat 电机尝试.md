> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/695674236)

硬件
--

使用研华工控机， EI-53-S7， 有 2 个网口， 2 对 CAN 口， 3 个 D9 口，4 个 USB 口

电机使用钛虎的， 基于 CAN 总线的一体化关节电机，

### EI-53 系列研华工控机开启 CAN 接口

开机按 del 键进入 BIOS

![](https://pic3.zhimg.com/v2-55fead7b175af2bbe1043f3dbe8c3500_r.jpg)

操作系统安装
------

ubuntu 22.04 server ， [https://releases.ubuntu.com/22.04.4/ubuntu-22.04.4-live-server-amd64.iso](https://link.zhihu.com/?target=https%3A//releases.ubuntu.com/22.04.4/ubuntu-22.04.4-live-server-amd64.iso)

ip 为 192.168.1.103, 账号 fudanrobotuser, 密码 1

使用 balenaEtcher 烧录 U 盘， 然后去重装工控机， 不连网线，

一些东西需要裁剪

```sh
#禁用 cloud-init
sudo touch /etc/cloud/cloud-init.disabled
#开机时间长
sudo mkdir /etc/systemd/system/network-online.target.wants/
sudo vi /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service

```

```
[Service]
Type=oneshot
ExecStart=/lib/systemd/systemd-networkd-wait-online
RemainAfterExit=yes
TimeoutStartSec=2sec

```

检查开机时间占用

```
systemd-analyze blame

```

![](https://pic4.zhimg.com/v2-6562f6dee33b58fb81d295a9044d7d33_r.jpg)

修改 IP 地址

```sh
sudo vi /etc/netplan/00-installer-config.yaml 

```

改为静态地址

```sh
network:
  ethernets:
    enp1s0:
      dhcp4: false
      addresses:
        - "192.168.1.105/24"
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: 
          - 114.114.114.114
          - 8.8.8.8
    eno0:
      dhcp4: false
      addresses:
        - "10.10.10.2/24"
  version: 2

```

改了以后没有直接生效，需要：

```
sudo netplan generate
sudo netplan apply

```

由牟同学帮忙裁剪的系统功能：

```
sudo systemctl stop apt-daily-upgrade.service
sudo systemctl disable apt-daily-upgrade.service
sudo systemctl disable apt-daily.timer
sudo systemctl stop apt-daily.timer

```

### 安装实时补丁包

22.04 以后的 ubuntu 安装实时补丁包会非常简单， 之前的版本， 都需要自己下载，编译， 很折腾人。

![](https://picx.zhimg.com/v2-bb82c986700e57f345b56fa922b6f3dd_r.jpg)

需要到 ubuntu 官网注册账号， 登录， 然后申请专业版（ubuntu pro）的授权， 每个账号可以免费激活 5 个点，

激活成功后：

![](https://pic1.zhimg.com/v2-ead6ea30e308d6e71f32d5244ace959e_r.jpg)

```
sudo pro disable livepatch
sudo pro enable realtime-kernel

```

然后重启，可以看到实时补丁包已经打好了

![](https://pic2.zhimg.com/v2-c9257fd69ee51171c3a7c29d5ab20173_r.jpg)

奇怪的是， ubuntu 官网上， 判定我已经使用了 4 台，

![](https://picx.zhimg.com/v2-462d41eacba473ddb918eaa0368ecb91_r.jpg)

我用这个 token 只激活了一个呀， 好无辜

### 安装 ROS（没苦硬吃）

清华镜像：

```sh
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse

deb http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse

```

```sh
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
sudo rm /etc/apt/sources.list
sudo vi /etc/apt/sources.list
#使用清华镜像 https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/
sudo apt-get update
sudo apt-get install  openssh-server

#ROS2 HUMBLE
locale  # check for UTF-8
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
locale  # verify settings
sudo apt install software-properties-common
sudo add-apt-repository universe
sudo apt update && sudo apt install curl -y

sudo curl -x http://192.168.1.103:10809 -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
sudo apt update && sudo apt install -y \
  python3-flake8-docstrings \
  python3-pip \
  python3-pytest-cov \
  ros-dev-tools
sudo apt upgrade
sudo apt install ros-humble-ros-base
sudo apt install ros-dev-tools

echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
echo "export ROS_DOMAIN_ID=1" >> ~/.bashrc
echo "export ROS_LOCALHOST_ONLY=1" >> ~/.bashrc


export https_proxy=http://192.168.1.103:10809
export http_proxy=http://192.168.1.103:10809
sudo -E  rosdep init
rosdep update

```

### ROS 安装，简单模式

使用鱼香 ros

```
wget http://fishros.com/install -O fishros && . fishros

```

### EI-53 研华工控机安装 CAN 驱动

向研华那边要到 Linux_Driver_AHC_20230901.zip 文件

```
sudo apt install -y make gcc net-tools can-utils unzip
cd ~
unzip Linux_Driver_AHC_20230901.zip
cd Linux_Driver_AHC_20230901/AHC_drivers/CAN
sudo make
sudo make install
sudo bash insmod_module_platform.sh
sudo cp can-ahc0512.ko /lib/modules/$(uname -r)/kernel/drivers/net/can/
sudo depmod -a
sudo vi /etc/modprobe.d/can-ahc0512.conf
#can-ach0512

```

接线  

![](https://pic4.zhimg.com/v2-9244ee3705291d17b9a049800f194395_1440w.jpg)

### igh 安装

```sh
cd ~
sudo apt-get install mokutil
mokutil --sb-state
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install git autoconf libtool pkg-config make build-essential net-tools
git clone https://gitlab.com/etherlab.org/ethercat.git
# 如果 gitlab 被墙了， 可以用国内 gitee 转一下
# git clone https://gitee.com/wei1224hf/ethercat.git
cd ethercat
git checkout stable-1.5
sudo rm /usr/bin/ethercat
sudo rm /etc/init.d/ethercat
./bootstrap  # to create the configure script
./configure --prefix=/usr/local/etherlab  --disable-8139too --disable-eoe --enable-generic --with-linux-dir=/usr/src/linux-headers-$(uname -r)


```

注意要禁用掉本机的 ubuntu 自动升级内核的功能

```sh
make all modules
sudo make modules_install install
sudo depmod
sudo ln -s /usr/local/etherlab/bin/ethercat /usr/bin/
sudo ln -s /usr/local/etherlab/etc/init.d/ethercat /etc/init.d/ethercat
sudo mkdir -p /etc/sysconfig
sudo cp /usr/local/etherlab/etc/sysconfig/ethercat /etc/sysconfig/ethercat

```

修改文件

```
sudo vi /etc/udev/rules.d/99-EtherCAT.rules

```

内容为：

```sh
KERNEL=="EtherCAT[0-9]*", MODE="0666"

```

修改另一个文件，网卡硬件地址要参考 ifconfig 命令的输出来，如果是 intel 网卡， 使用 igb 类型

```sh
sudo vi /etc/sysconfig/ethercat
```

内容为：

```
MASTER0_DEVICE="00:0c:29:ad:73:22"
DEVICE_MODULES="generic"

```

启动 ethercat，这部分代码要设置为开机自启动

```
sudo /etc/init.d/ethercat start

```

检查节点，如果网口上接入了伺服驱动器的话，可以扫到

```
ethercat slaves

```

### ROS2 CAN-OPEN 库安装

```
cd ~
mkdir ws_canopen
cd ws_canopen
mkdir src
cd src
git config --global http.proxy http://192.168.1.103:10809
git config --global https.proxy http://192.168.1.103:10809
git clone https://github.com/ros-industrial/ros2_canopen.git -b $ROS_DISTRO
cd ..
rosdep install --from-paths src/ros2_canopen --ignore-src -r -y
colcon build

```

ROS2-CANOPEN 库依赖 .eds 格式的 CANOPEN 类设备的描述文件，类似于 ethercat 设备都有 xml 格式的设备描述文件。

下载并安装 .eds 文件编辑器

[https://github.com/robincornelius/libedssharp/releases](https://link.zhihu.com/?target=https%3A//github.com/robincornelius/libedssharp/releases)

![](https://pica.zhimg.com/v2-ec8f1d552c2c615570156dbab972c374_r.jpg)

### ROS2 ethercat 库安装

硬件上， 接上开璇的 2 个电机，一个低压伺服电机，一个一体化关节

```
#下载安装 ethercat_driver_ros2
cd ~
mkdir ws_ethercat
cd ws_ethercat
mkdir src
cd src
git clone https://github.com/ICube-Robotics/ethercat_driver_ros2.git 
git clone https://github.com/ICube-Robotics/ethercat_driver_ros2_examples.git
cd ../
rosdep install --ignore-src --from-paths . -y -r
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release --symlink-install
source install/setup.bash

```

添加开璇电机的 ROS2 包配置

/home/gene/ws_ethercat/src/ethercat_driver_ros2_examples/ethercat_motor_drive/config/kaiser_kde_config.yaml

```yaml
vendor_id:  0x00010203
product_id: 0x00000402
assign_activate: 0x0300  # DC Synch register
period: 125  # Hz
auto_fault_reset: false  # true = automatic fault reset, false = fault reset on rising edge command interface "reset_fault"
sdo: # sdo data to be transferred at drive startup
  - { index: 0x1C32, sub_index: 1, type: int16, value: 2 } 
  - { index: 0x1C33, sub_index: 1, type: int16, value: 2 } 
rpdo:  # RxPDO = receive PDO Mapping
  - index: 0x1601
    channels:
      - {index: 0x6040, sub_index: 0, type: uint16, default: 0}  # Control word
      - {index: 0x607a, sub_index: 0, type: int32, command_interface: position, default: .nan}  # Target position
      - {index: 0x60ff, sub_index: 0, type: int32, default: 0, command_interface: velocity, default: .nan}  # Target velocity
      - {index: 0x6071, sub_index: 0, type: int16, default: 0, command_interface: torque, default: .nan}  # Target torque
      - {index: 0x6060, sub_index: 0, type: int8, default: 8}  # Mode of operation
      - {index: 0x2078, sub_index: 1, type: uint16, default: 0}  # Digital Output Functionalities
tpdo:  # TxPDO = transmit PDO Mapping
  - index: 0x1a01
    channels:
      - {index: 0x6041, sub_index: 0, type: uint16}  # Status word
      - {index: 0x6064, sub_index: 0, type: int32, state_interface: position}  # Position actual value
      - {index: 0x606c, sub_index: 0, type: int32, state_interface: velocity}  # Velocity actual value
      - {index: 0x6077, sub_index: 0, type: int16, state_interface: effort}  # Torque actual value
      - {index: 0x6061, sub_index: 0, type: int8}  # Mode of operation display


```

controller.yaml， 调成 2 个电机， 10 毫秒的周期

```yaml
controller_manager:
  ros__parameters:
    update_rate: 100  # Hz
    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster
    trajectory_controller:
      type: joint_trajectory_controller/JointTrajectoryController
    velocity_controller:
      type: velocity_controllers/JointGroupVelocityController
    effort_controller:
      type: effort_controllers/JointGroupEffortController

trajectory_controller:
  ros__parameters:
    command_interfaces:
      - position
    state_interfaces:
      - position
    joints:
      - joint_1
      - joint_2

velocity_controller:
  ros__parameters:
    command_interfaces:
      - velocity
    state_interfaces:
      - velocity  
    joints:
      - joint_1
      - joint_2

effort_controller:
  ros__parameters:
    joints:
      - joint_1
      - joint_2


```

/home/gene/ws_ethercat/src/ethercat_driver_ros2_examples/ethercat_motor_drive/description/ros2_control/motor_drive.ros2_control.xacro

定义好电机设备描述文件及其他接口

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
  <xacro:macro >

    <ros2_control >
      <hardware>
        <plugin>ethercat_driver/EthercatDriver</plugin>
        <param >0</param>
        <param >100</param>
      </hardware>

      <joint >
        <state_interface />
        <state_interface />
        <state_interface />
        <command_interface />
        <command_interface />
        <command_interface />
        <ec_module >
          <plugin>ethercat_generic_plugins/EcCiA402Drive</plugin>
          <param >0</param>
          <param >0</param>
          <param >3</param>
          <param >$(find ethercat_motor_drive)/config/kaiser_kde_config.yaml</param>
        </ec_module>
      </joint>
      
      <joint >
        <state_interface />
        <state_interface />
        <state_interface />
        <command_interface />
        <command_interface />
        <command_interface />
        <ec_module >
          <plugin>ethercat_generic_plugins/EcCiA402Drive</plugin>
          <param >0</param>
          <param >1</param>
          <param >3</param>
          <param >$(find ethercat_motor_drive)/config/kaiser_kde_config.yaml</param>
        </ec_module>
      </joint>      
      
    </ros2_control>
  </xacro:macro>
</robot>


```

安全一些额外的包

```sh
sudo apt install ros-humble-hardware-interface \
ros-humble-xacro \
ros-humble-imu-sensor-broadcaster \
ros-humble-diff-drive-controller  \
ros-humble-position-controllers  \
ros-humble-gripper-controllers  \
ros-humble-joint-state-broadcaster  \
ros-humble-joint-trajectory-controller  \
ros-humble-controller-manager \
ros-humble-velocity-controllers \
ros-humble-ros2-control

```

```sh
ros2 launch ethercat_motor_drive motor_drive.launch.py

```

### 让电机转起来

```sh
ros2 topic pub -r 0.2 /trajectory_controller/joint_trajectory trajectory_msgs/msg/JointTrajectory '{header: {stamp: {sec: 0, nanosec: 0}, frame_id: ""}, joint_names: ["joint_1"], points: [{positions: [100.0], velocities: [0.0], accelerations: [0.0], time_from_start: {sec: 1, nanosec: 0}},{positions: [10000.0], velocities: [0.0], accelerations: [0.0],time_from_start: {sec: 5, nanosec: 0}}]}'

```

观察角度变化

```sh
ros2 topic echo /joint_states
```

### 允许 root 登录 ubuntu 22.04

```sh
sudo passwd root
sudo nano /etc/gdm3/custom.conf

[security]
AllowRoot=true

sudo nano /etc/pam.d/gdm-password

#auth    required        pam_succeed_if.so user != root quiet_success


sudo nano /etc/ssh/sshd_config

PermitRootLogin yes


sudo systemctl restart ssh

```

### 提高 ROS 节点实时性

使用 root 账号来运行， 否则无法让 ethercat_ros2 节点在 RT 模式下运行，若成功，则：

```
Spawning controller_manager RT thread with scheduler priority: 50

```

否则就会报错

```
Could not enable FIFO RT scheduling policy: with error number <1>(Operation not permitted). See [https://control.ros.org/master/doc/ros2_control/controller_manager/doc/userdoc.html] for details on how to enable realtime scheduling

```

然后做 CPU 控核，

```
sudo apt install htop
htop

sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash isolcpus=26,27"
sudo update-grub
sudo reboot
cat /sys/devices/system/cpu/isolated
htop

```

然后，将 ros2 节点启动的程序， 写成批处理代码

```
cd ~/ws/
nano cpu26.sh
ros2 launch ethercat_motor_drive motor_drive.launch.py
chmod +x cpu26.sh
taskset -c 26 ./cpu26.sh

```

这样，就可以让 ros 节点程序， 在 26 号 CPU 上， 单独运行

钛虎电机调试
------

![](https://pic1.zhimg.com/v2-a6a75ffa3d70460195c06059a416677a_1440w.jpg)

windows pc 端， 安装 canalyst-ii 驱动， 然后连线电机， 黑 L, 黄 G，红 H，

用上位机搜索，can 模式， 非 can-open 模式，

![](https://pic2.zhimg.com/v2-1ef1cd6fb4ebe0ed83e1e3deed180911_r.jpg)

选择速度模式， 400 的速度， 启动电机， 然后电机会转动

工控机上， 发送的报文格式为：

配置好 can0

```
sudo ip link set can0 type can bitrate 1000000
sudo ip link set can0 up

```

### 普通 can 电机

发送的查询报文，长度为 1， 命令报文， 长度为 5

```
#速度模式转动
cansend can0 001#1d34030000
#速度模式停止
cansend can0 001#1d00000000

```

### canopen 电机

报文长度固定为 8

![](https://pic1.zhimg.com/v2-0b0ce96720e64a8efed1e849f0b507d8_1440w.jpg)

```
#查询报文

cansend can0 603#40.3F.60.00.00.00.00.00

cansend can0 603#43.78.60.00.00.00.00.00

cansend can0 603#43.64.60.00.00.00.00.00
cansend can0 603#43.6C.60.00.00.00.00.00


cansend can0 603#43.0E.20.00.00.00.00.00
cansend can0 603#43.0D.20.00.00.00.00.00
cansend can0 603#43.0C.20.00.00.00.00.00

#控制报文
cansend can1 603#2B.40.60.00.1F.00.00.00
cansend can1 603#2F.60.60.00.00.00.00.00
cansend can1 603#2B.40.60.00.06.00.00.00
cansend can1 603#2B.40.60.00.1F.00.00.00
cansend can1 603#2F.60.60.00.03.00.00.00
cansend can1 603#23.FF.60.00.14.00.00.00
cansend can1 603#2F.60.60.00.03.00.00.00

```

根据钛虎电机厂家的 canopen 报文， 编辑 .eds 设备描述文件

开璇电机， 支持 ethercat 和 canopen 两种模式
--------------------------------

有两种电机， 一种是一体化关节， 另一种是驱动板 + 小电机

![](https://pic1.zhimg.com/v2-07e8ac9d1e9dec8ee25d352f0cc8a690_1440w.jpg)

### 串口调试设备

对于低压伺服板 + 电机，需要用基于 RS232 的串口线，

上面需要接入 编码器电池， 电机 UVW 三线， 棕 u，黄 v，蓝 w，

使用 TTL 转 USB ， 虽然可以接通电机，但 online 一下后， 就 offline 了

![](https://pic1.zhimg.com/v2-2da93d32ab7ba3655933f75542779902_r.jpg)

必须保证只有一个串口，需要关闭其他串口

电机的同步周期只接受 0.5/1/2/4/8 毫秒

### ethercat 调试

```
ethercat upload --position 0 --type int8 0x6060 0x00 #查看模式
ethercat upload --position 0 --type int16 0x6041 0x00 #查看状态
ethercat upload --position 0 --type int16 0x603f 0x00 #查看故障

#改模式
ethercat download --position 0 --type int8 0x6060 0x00 0x08
#改状态机
ethercat download --position 0 --type int16 0x6040 0x00 0x06
ethercat download --position 0 --type int16 0x6040 0x00 0x07
ethercat download --position 0 --type int16 0x6040 0x00 0x0f
#故障清除
ethercat download -p 0 -t uint16 0x6040 0 128 

#修改速度
ethercat download --position 0 --type int32 0x60ff 0x00 50
#查看目标速度
ethercat upload --position 0 --type int32 0x60ff 0x00 
#查看真实速度
ethercat upload --position 0 --type int32 0x606c 0x00 
#查看位置
ethercat upload --position 0 --type int32 0x60ff 0x00 

```

现在不停的报错

```
[ros2_control_node-1] STATE: Switch on Disabled with status word :2640
[ros2_control_node-1] STATE: Ready to Switch On with status word :2609
[ros2_control_node-1] STATE: Switch On with status word :2611
[ros2_control_node-1] STATE: Fault with status word :2584
[ros2_control_node-1] STATE: Switch on Disabled with status word :2640
[ros2_control_node-1] STATE: Ready to Switch On with status word :2609
[ros2_control_node-1] STATE: Switch On with status word :2611
[ros2_control_node-1] STATE: Fault with status word :2584
[ros2_control_node-1] STATE: Switch on Disabled with status word :2640
[ros2_control_node-1] STATE: Ready to Switch On with status word :2609

```

ethercat 内容, 使用 ethercat xml

```
  <EtherCATInfo>
    <!-- Slave 2 -->
    <Vendor>
      <Id>66051</Id>
    </Vendor>
    <Descriptions>
      <Devices>
        <Device>
          <Type ProductCode="#x00000402" RevisionNo="#x00050012">Kaiserdrive_ECAT</Type>
          <Name><![CDATA[KDE EtherCAT Drive (CoE)]]></Name>
          <Sm Enable="1" StartAddress="#x1000" ControlByte="#x26" DefaultSize="128" />
          <Sm Enable="1" StartAddress="#x1080" ControlByte="#x22" DefaultSize="128" />
          <Sm Enable="1" StartAddress="#x1100" ControlByte="#x64" DefaultSize="12" />
          <Sm Enable="1" StartAddress="#x1d00" ControlByte="#x20" DefaultSize="18" />
          <RxPdo Sm="2" Fixed="1" Mandatory="1">
            <Index>#x1601</Index>
            <Name>2</Name>
            <Entry>
              <Index>#x6040</Index>
              <SubIndex>0</SubIndex>
              <BitLen>16</BitLen>
              <Name>C</Name>
              <DataType>UINT16</DataType>
            </Entry>
            <Entry>
              <Index>#x6071</Index>
              <SubIndex>0</SubIndex>
              <BitLen>16</BitLen>
              <Name>T</Name>
              <DataType>UINT16</DataType>
            </Entry>
            <Entry>
              <Index>#x60ff</Index>
              <SubIndex>0</SubIndex>
              <BitLen>32</BitLen>
              <Name>T</Name>
              <DataType>UINT32</DataType>
            </Entry>
            <Entry>
              <Index>#x6060</Index>
              <SubIndex>0</SubIndex>
              <BitLen>8</BitLen>
              <Name>M</Name>
              <DataType>UINT8</DataType>
            </Entry>
          </RxPdo>
          <TxPdo Sm="3" Fixed="1" Mandatory="1">
            <Index>#x1a01</Index>
            <Name>2</Name>
            <Entry>
              <Index>#x6041</Index>
              <SubIndex>0</SubIndex>
              <BitLen>16</BitLen>
              <Name>S</Name>
              <DataType>UINT16</DataType>
            </Entry>
            <Entry>
              <Index>#x6064</Index>
              <SubIndex>0</SubIndex>
              <BitLen>32</BitLen>
              <Name>P</Name>
              <DataType>UINT32</DataType>
            </Entry>
            <Entry>
              <Index>#x606c</Index>
              <SubIndex>0</SubIndex>
              <BitLen>32</BitLen>
              <Name>V</Name>
              <DataType>UINT32</DataType>
            </Entry>
            <Entry>
              <Index>#x6077</Index>
              <SubIndex>0</SubIndex>
              <BitLen>16</BitLen>
              <Name>T</Name>
              <DataType>UINT16</DataType>
            </Entry>
            <Entry>
              <Index>#x603f</Index>
              <SubIndex>0</SubIndex>
              <BitLen>16</BitLen>
              <Name>E</Name>
              <DataType>UINT16</DataType>
            </Entry>
          </TxPdo>
        </Device>
      </Devices>
    </Descriptions>
  </EtherCATInfo>
</EtherCATInfoList>


```

```
vendor_id:  0x00010203
product_id: 0x00000402
assign_activate: 0x0300  # DC Synch register
auto_fault_reset: false  # true = automatic fault reset, false = fault reset on rising edge command interface "reset_fault"
#sdo:  # sdo data to be transferred at drive startup
#  - {index: 0x60C2, sub_index: 1, type: int8, value: 10} # Set interpolation time for cyclic modes to 10 ms
#  - {index: 0x60C2, sub_index: 2, type: int8, value: -3} # Set base 10-3s
rpdo:  # RxPDO = receive PDO Mapping
  - index: 0x1601
    channels:
      - {index: 0x6040, sub_index: 0, type: uint16, default: 0}  # Control word
      - {index: 0x607a, sub_index: 0, type: int32, command_interface: position, default: .nan}  # Target position
      - {index: 0x60ff, sub_index: 0, type: int32, command_interface: velocity, default: .nan}  # Target velocity
      - {index: 0x6071, sub_index: 0, type: int16, command_interface: effort, default: .nan}  # Target torque
      - {index: 0x6060, sub_index: 0, type: int8, default: 8}  # Mode of operation
tpdo:  # TxPDO = transmit PDO Mapping
  - index: 0x1a01
    channels:
      - {index: 0x6041, sub_index: 0, type: uint16}  # Status word
      - {index: 0x6064, sub_index: 0, type: int32, state_interface: position}  # Position actual value
      - {index: 0x606c, sub_index: 0, type: int32, state_interface: velocity}  # Velocity actual value
      - {index: 0x6077, sub_index: 0, type: int16, state_interface: effort}  # Torque actual value
      - {index: 0x6061, sub_index: 0, type: int8}  # Mode of operation display


```

### 尝试使用纯速度模式：

```
ros2 topic pub /velocity_controller/commands std_msgs/msg/Float64MultiArray "data: 
- 110.5"

```

尝试模式要改成 3

```
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
  <xacro:macro >
    <ros2_control >
      <hardware>
        <plugin>ethercat_driver/EthercatDriver</plugin>
        <param >0</param>
        <param >125</param>
      </hardware>
      <joint >
        <state_interface />
        <state_interface />
        <state_interface />
        <command_interface />
        <command_interface />
        <command_interface />
        <ec_module >
          <plugin>ethercat_generic_plugins/EcCiA402Drive</plugin>
          <param >0</param>
          <param >0</param>
          <param >3</param>
          <param >$(find ethercat_motor_drive)/config/kaiser_1_config.yaml</param>
        </ec_module>
      </joint>
    </ros2_control>
  </xacro:macro>
</robot>

```

launch 文件中要修改默认启动控制器

```
    nodes = [
        control_node,
        robot_state_pub_node,
        joint_state_broadcaster_spawner,
        velocity_controller_spawner,
        # trajectory_controller_spawner,
        # effort_controller_spawner,
    ]

```

这样，总算可以正常启动并运行了：

![](https://pic4.zhimg.com/v2-cf9b0f2fb79078495fbd84b222465ca9_r.jpg)

### 使用同步周期位置模式

```
ros2 topic pub -r 0.2 /trajectory_controller/joint_trajectory trajectory_msgs/msg/JointTrajectory '{header: {stamp: {sec: 0, nanosec: 0}, frame_id: ""}, joint_names: ["joint_1"], points: [{positions: [100.0], velocities: [0.0], accelerations: [0.0], time_from_start: {sec: 1, nanosec: 0}},{positions: [10000000.0], velocities: [0.0], accelerations: [0.0],time_from_start: {sec: 5, nanosec: 0}}]}'

```

注意 controllers.yaml 里的 upate_rate 跟 motor_drive.ros2_control.xacro 里的 control_frequency 两个必须一样， 我设置成 125

伺服总是故障，说位置偏差过大， 就需要手动调整编码器值，清零，

如果继续故障 ，就手动发送清故障码

```
 ethercat download -p 0 -t uint16 0x6040 0 128 

```

小象电机 + 外置伺服驱动板
--------------

小象电机的编码器线序

![](https://pic1.zhimg.com/v2-a4cecc6123ec13d3dde5dade240103a4_r.jpg)

elmo 稍小一点的伺服驱动器跟编码器接线：

红 - 红

白 - 黑

黄 - 黄

蓝 - 紫

ethercat 网线接线

![](https://pic3.zhimg.com/v2-894d8f0920e74f1f63a70e84f7e87272_r.jpg)

大一点的伺服驱动器接线：

紫 - 3

黄 - 5

红 - 27

黑 - 26

### 小功率 elmo 伺服驱动板配置

```
ethercat xml

```

得到

```
<?xml version="1.0" ?>
<EtherCATInfo>
  <!-- Slave 0 -->
  <Vendor>
    <Id>154</Id>
  </Vendor>
  <Descriptions>
    <Devices>
      <Device>
        <Type ProductCode="#x00030924" RevisionNo="#x00010420"></Type>
        <Sm Enable="1" StartAddress="#x1800" ControlByte="#x26" DefaultSize="140" />
        <Sm Enable="1" StartAddress="#x1900" ControlByte="#x22" DefaultSize="140" />
        <Sm Enable="1" StartAddress="#x1100" ControlByte="#x64" DefaultSize="32" />
        <Sm Enable="1" StartAddress="#x1180" ControlByte="#x20" DefaultSize="32" />
        <RxPdo Sm="2" Fixed="1" Mandatory="1">
          <Index>#x1600</Index>
          <Name>RPDO1 Mapping</Name>
          <Entry>
            <Index>#x607a</Index>
            <SubIndex>0</SubIndex>
            <BitLen>32</BitLen>
            <Name>Target position</Name>
            <DataType>UINT32</DataType>
          </Entry>
          <Entry>
            <Index>#x60fe</Index>
            <SubIndex>1</SubIndex>
            <BitLen>32</BitLen>
            <Name>SubIndex 001</Name>
            <DataType>UINT32</DataType>
          </Entry>
          <Entry>
            <Index>#x6040</Index>
            <SubIndex>0</SubIndex>
            <BitLen>16</BitLen>
            <Name>Controlword</Name>
            <DataType>UINT16</DataType>
          </Entry>
        </RxPdo>
        <TxPdo Sm="3" Fixed="1" Mandatory="1">
          <Index>#x1a00</Index>
          <Name>TPDO1 Mapping</Name>
          <Entry>
            <Index>#x6064</Index>
            <SubIndex>0</SubIndex>
            <BitLen>32</BitLen>
            <Name>Position actual value</Name>
            <DataType>UINT32</DataType>
          </Entry>
          <Entry>
            <Index>#x60fd</Index>
            <SubIndex>0</SubIndex>
            <BitLen>32</BitLen>
            <Name>Digital inputs</Name>
            <DataType>UINT32</DataType>
          </Entry>
          <Entry>
            <Index>#x6041</Index>
            <SubIndex>0</SubIndex>
            <BitLen>16</BitLen>
            <Name>Statusword</Name>
            <DataType>UINT16</DataType>
          </Entry>
        </TxPdo>
      </Device>
    </Devices>
  </Descriptions>
</EtherCATInfo>


```

因此，配置 ros2 ethercat 包

```
vendor_id:  0x0000009a
product_id: x00030924
assign_activate: 0x0300  # DC Synch register
auto_fault_reset: false  # true = automatic fault reset, false = fault reset on rising edge command interface "reset_fault"
#sdo:  # sdo data to be transferred at drive startup
#  - {index: 0x60C2, sub_index: 1, type: int8, value: 10} # Set interpolation time for cyclic modes to 10 ms
#  - {index: 0x60C2, sub_index: 2, type: int8, value: -3} # Set base 10-3s
rpdo:  # RxPDO = receive PDO Mapping
  - index: 0x1600
    channels:
      - {index: 0x6040, sub_index: 0, type: uint16, default: 0}  # Control word
      - {index: 0x607a, sub_index: 0, type: int32, command_interface: position, default: .nan}  # Target position
      - {index: 0x60ff, sub_index: 0, type: int32, command_interface: velocity, default: .nan}  # Target velocity
      - {index: 0x6071, sub_index: 0, type: int16, command_interface: effort, default: .nan}  # Target torque
      - {index: 0x6060, sub_index: 0, type: int8, default: 8}  # Mode of operation
tpdo:  # TxPDO = transmit PDO Mapping
  - index: 0x1a00
    channels:
      - {index: 0x6041, sub_index: 0, type: uint16}  # Status word
      - {index: 0x6064, sub_index: 0, type: int32, state_interface: position}  # Position actual value
      - {index: 0x606c, sub_index: 0, type: int32, state_interface: velocity}  # Velocity actual value
      - {index: 0x6077, sub_index: 0, type: int16, state_interface: effort}  # Torque actual value
      - {index: 0x6061, sub_index: 0, type: int8}  # Mode of operation display


```

文件为

/home/fudanrobotuser/ws_ethercat/src/ethercat_driver_ros2_examples/ethercat_motor_drive/config/elmo_1_config.yaml

### elmo 上位机使用

使用普通的 micro-usb （扁口串口线）连接电脑， 打开上位机，使用 open driver locater

![](https://pic1.zhimg.com/v2-8470eee9f5e376238fff5a49301190ec_r.jpg)

选择 USB ， 然后点 locate ，会找到对应的串口

![](https://pic2.zhimg.com/v2-e512d36190bc2aa83014d12b3b529f5f_r.jpg)

如果找不到串口，要确认是否安装了驱动，要保证电脑的设备管理器可以识别到串口通道

![](https://pic2.zhimg.com/v2-f620b23f0d6f647d39f15528fdca2c55_r.jpg)

需要进行配置

![](https://pic3.zhimg.com/v2-0689ae48cc22d10b67b23f6e6f214b76_r.jpg)![](https://picx.zhimg.com/v2-6d1955258b6df6012e623d666b18ca8f_r.jpg)

电机减速比为 1：16 ，电机端单编码器，

![](https://pic1.zhimg.com/v2-dd581f2d10a508ecd5caca9c10ff6906_1440w.jpg)![](https://pic4.zhimg.com/v2-075a55a3cdac6571c57482fcb5e280d7_r.jpg)![](https://pic2.zhimg.com/v2-24f42700dfc2ff714eacfb4a3ca55759_r.jpg)![](https://pic4.zhimg.com/v2-1f787d68193d1d653cc2caa1b68e4ef1_r.jpg)

报错， STO 接口异常， 需要将 STO 的 1 2 短接到 3 上

![](https://pica.zhimg.com/v2-cf75ef959d0880b62a5699a3d0c27ade_r.jpg)![](https://pic1.zhimg.com/v2-0c83840105e8b9f12d50b01dce5dea86_r.jpg)

报错说编码器读不到值，认为电机没有转动

全部配置完成：

![](https://pic1.zhimg.com/v2-7c0fbcf64a3b66c04d5e413ac2f3de48_r.jpg)

然后，就可以通过上位机来控制电机转动了

![](https://pic4.zhimg.com/v2-d4a187ddacdfaf429686cafe51f3fc9f_r.jpg)

### 使用 igh 命令直接控制电机转动

只能使用 PP PV 模式， 无法使用 CSV CSP 模式

```
#改模式，速度模式 , PV
ethercat download --position 0 --type int8 0x6060 0x00 0x03
#改状态机
ethercat download --position 0 --type int16 0x6040 0x00 0x06
ethercat download --position 0 --type int16 0x6040 0x00 0x07
ethercat download --position 0 --type int16 0x6040 0x00 0x0f
#改速度
ethercat download --position 0 --type int32 0x60ff 0x00 4800
#查看速度
ethercat upload --position 0 --type int32 0x606c 0x00
#查看位置
ethercat upload --position 0 --type int32 0x6064 0x00

```