---
url: https://zhuanlan.zhihu.com/p/687129318
title: ROS2 使用 icub 的 ethercat_driver_ros2 驱动汇川伺服尝试
date: 2025-03-15 23:00:33
tag: 
summary: 
---
## 1 [SV660N](https://zhida.zhihu.com/search?content_id=240825478&content_type=Article&match_order=1&q=SV660N&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDIyMjI1NDUsInEiOiJTVjY2ME4iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNDA4MjU0NzgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Pn0X9UcKdVb067f0MZQXMGkvREJeBF8QYWLVp98axBE&zhida_source=entity) 配置修改

H02.31 改 1 , 恢复出厂状态

H02.01 改 1, 设置为绝对值编码器， 接上编码器电池， 默认 为增量编码器， 不支持断电位置记忆

H0E.32 改成 4000

## 2 ethercat igh 安装

```
sudo apt-get install mokutil
mokutil --sb-statesudo apt-get update
sudo apt-get upgrade
sudo apt-get install git autoconf libtool pkg-config make build-essential net-tools
git clone https://gitlab.com/etherlab.org/ethercat.git
cd ethercat
git checkout stable-1.5
sudo rm /usr/bin/ethercat
sudo rm /etc/init.d/ethercat
./bootstrap  # to create the configure script
./configure --prefix=/usr/local/etherlab  --disable-8139too --disable-eoe --enable-generic --with-linux-dir=/usr/src/linux-headers-5.13.0-28-generic/

#--with-linux-dir=/usr/src/linux-headers-5.13.0-28-generic/  这部分要根据 uname -a 里的反馈结果来修改
#注意要禁用掉本机的 ubuntu 自动升级内核的功能

make all modules
sudo make modules_install install
sudo depmod

sudo ln -s /usr/local/etherlab/bin/ethercat /usr/bin/
sudo ln -s /usr/local/etherlab/etc/init.d/ethercat /etc/init.d/ethercat
sudo mkdir -p /etc/sysconfig
sudo cp /usr/local/etherlab/etc/sysconfig/ethercat /etc/sysconfig/ethercat
sudo gedit /etc/udev/rules.d/99-EtherCAT.rules
#KERNEL=="EtherCAT[0-9]*", MODE="0666"

sudo gedit /etc/sysconfig/ethercat
#MASTER0_DEVICE="00:0c:29:ad:73:22"
#DEVICE_MODULES="generic"


sudo /etc/init.d/ethercat start

ethercat slaves

```

编译时有 warning：

```
warning: the compiler differs from the one used to build the kernel
  The kernel was built by: x86_64-linux-gnu-gcc-12 (Ubuntu 12.3.0-1ubuntu1~22.04) 12.3.0
  You are using:           gcc-12 (Ubuntu 12.3.0-1ubuntu1~22.04) 12.3.0
make[1]: Leaving directory '/usr/src/linux-headers-6.5.0-25-generic'


```

## 3 [ROS2](https://zhida.zhihu.com/search?content_id=240825478&content_type=Article&match_order=1&q=ROS2&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDIyMjI1NDUsInEiOiJST1MyIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjQwODI1NDc4LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.YTdm75oFSsXN4vdXOKZNzOy2gHeAqAr73qoPjzBJP1c&zhida_source=entity) 安装 [ethercat_driver_ros2](https://link.zhihu.com/?target=https%3A//github.com/ICube-Robotics/ethercat_driver_ros2)

```sh
#下载安装 ethercat_driver_ros2
cd ~/ros2_ws
git clone https://github.com/ICube-Robotics/ethercat_driver_ros2.git src/ethercat_driver_ros2
rosdep install --ignore-src --from-paths . -y -r
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release --symlink-install
source install/setup.bash

```

igh 安装在 虚拟机 里的时候， 似乎有些问题，有时会识别不了伺服

## 4 ethercat_driver_ros2 使用

下载官方的例子文件：

```sh
cd ~/ros2_ws/src/
git clone https://github.com/ICube-Robotics/ethercat_driver_ros2_examples.git
cd ../
rosdep install --ignore-src --from-paths . -y -r
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release --symlink-install 

```

  
可以看到里面有驱动伺服电机的例子， 修改 ~/ros2_ws/src/ethercat_driver_ros2_examples/ethercat_motor_drive/config

新建一个汇川 SV660N 的配置：

![](https://pic3.zhimg.com/v2-3cb116c30986fb61c4f55d21b172aa52_r.jpg)

```
# Configuration file for Maxon EPOS3 drive
vendor_id: 0x100000
product_id: 0x000c010d
assign_activate: 0x0300  # DC Synch register
auto_fault_reset: false  # true = automatic fault reset, false = fault reset on rising edge command interface "reset_fault"

rpdo:  # RxPDO = receive PDO Mapping
  - index: 0x1600
    channels:
      - {index: 0x6040, sub_index: 0, type: uint16, default: 0}  # Control word
      - {index: 0x607a, sub_index: 0, type: int32, command_interface: position, default: .nan}  # Target position
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

再修改 /home/gene/ros2_ws/src/ethercat_driver_ros2_examples/ethercat_motor_drive/description/ros2_control/motor_drive.ros2_control.xacro

![](https://picx.zhimg.com/v2-7161156caa98859e57bdb8bb0138d77f_r.jpg)

先启动 ethercat

```
sudo /etc/init.d/ethercat start

```

然后运行：

```
ros2 launch ethercat_motor_drive motor_drive.launch.py

```

![](https://pic4.zhimg.com/v2-d67c434997c4dcbffea49d2606e4c8cf_r.jpg)

观察 ROS 里监听到的角度值：

```
ros2 topic echo /joint_states

```

![](https://pica.zhimg.com/v2-b51fc6d5638c1b7aded1eeae5090e686_1440w.jpg)

让电机转动

```
ros2 topic pub -r 0.2 /trajectory_controller/joint_trajectory trajectory_msgs/msg/JointTrajectory '{header: {stamp: {sec: 0, nanosec: 0}, frame_id: ""}, joint_names: ["joint_1"], points: [{positions: [19000000.0], velocities: [0.0], accelerations: [0.0], time_from_start: {sec: 1, nanosec: 0}}]}'

ros2 topic pub -r 0.2 /trajectory_controller/joint_trajectory trajectory_msgs/msg/JointTrajectory '{header: {stamp: {sec: 0, nanosec: 0}, frame_id: ""}, joint_names: ["joint_1"], points: [{positions: [10.0], velocities: [0.0], accelerations: [0.0], time_from_start: {sec: 1, nanosec: 0}}]}'



```

这样就成功的让电机转动了

### 故障处理 SV660N 报错 EE15.0

![](https://picx.zhimg.com/v2-20750cc24f56d3b7d7bd5b54cdee5091_r.jpg)

修改 H0E.32 ， 默认 3000，改为 4000

![](https://picx.zhimg.com/v2-d0a63b555acd47b5a8791803f25a09e5_r.jpg)

原因是同步周期误差太大导致的

![](https://pic1.zhimg.com/v2-4eaa1ce8d9176f4a8d03c593b4d497a6_r.jpg)

## 5 [IS620N](https://zhida.zhihu.com/search?content_id=240825478&content_type=Article&match_order=1&q=IS620N&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDIyMjI1NDUsInEiOiJJUzYyME4iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNDA4MjU0NzgsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.zv-hIGs43Y0mom0yMUfpNe6pV-1HAY2T79tFVqkcmXI&zhida_source=entity) 伺服

```
# Configuration file for Maxon EPOS3 drive
vendor_id: 0x100000
product_id: 0x000c0108
assign_activate: 0x0300  # DC Synch register
auto_fault_reset: false  # true = automatic fault reset, false = fault reset on rising edge command interface "reset_fault"

rpdo:  # RxPDO = receive PDO Mapping
  - index: 0x1600
    channels:
      - {index: 0x6040, sub_index: 0, type: uint16, default: 0}  # Control word
      - {index: 0x607a, sub_index: 0, type: int32, command_interface: position, default: .nan}  # Target position
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

注意 RX 跟 TX 必须是 1600 跟 1A00 才对

启动服务：

```
 ros2 run ethercat_manager ethercat_sdo_srv_server

```

启动 rqt

尝试读取位置，失败，报错

![](https://pica.zhimg.com/v2-92c6f2b622b487f1d90937a800eba556_r.jpg)

```
[ethercat_manager]: ioctl() version magic is differing: /dev/EtherCAT0: 33, ethercat tool: 31

```

故障代码来自 /home/gene/ros2_ws/src/ethercat_driver_ros2/ethercat_manager/include/ethercat_manager

ec_master_async.hpp 84 行附近 ：

```
      getModule(&module_data);
      /* WXF
      if (module_data.ioctl_version_magic != EC_IOCTL_VERSION_MAGIC) {
        std::stringstream err;
        err << "ioctl() version magic is differing: "
            << deviceName.str() << ": " << module_data.ioctl_version_magic
            << ", ethercat tool: " << EC_IOCTL_VERSION_MAGIC;
        throw MasterException(err.str());
      }
      */
      mcount_ = module_data.master_count;


```

直接注释掉这部分故障判断代码，虽然可以读取到 SDO 的值，写入也是成功，但是电机并没有转动

![](https://pic1.zhimg.com/v2-1a4f3483e793fab95cda7f52aeb54b58_r.jpg)

电机转起来需要通过 PDO ，而不是 SDO

### 配置 4 个电机：

修改 motor_drive.ros2_control.xacro 文件

```
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

  <xacro:macro name="motor_drive">

    <ros2_control name="motor_drive" type="system">
      <hardware>
        <plugin>ethercat_driver/EthercatDriver</plugin>
        <param name="master_id">0</param>
        <param name="control_frequency">100</param>
      </hardware>

      <joint name="joint_1">
        <state_interface name="position"/>
        <state_interface name="velocity"/>
        <state_interface name="effort"/>
        <command_interface name="position"/>
        <command_interface name="reset_fault"/>
        <ec_module name="Maxon">
          <plugin>ethercat_generic_plugins/EcCiA402Drive</plugin>
          <param name="alias">0</param>
          <param name="position">0</param>
          <param name="mode_of_operation">8</param>
          <param name="slave_config">$(find ethercat_motor_drive)/config/inovance_IS620N_config.yaml</param>
        </ec_module>
      </joint>
      
      <joint name="joint_2">
        <state_interface name="position"/>
        <state_interface name="velocity"/>
        <state_interface name="effort"/>
        <command_interface name="position"/>
        <command_interface name="reset_fault"/>
        <ec_module name="Maxon">
          <plugin>ethercat_generic_plugins/EcCiA402Drive</plugin>
          <param name="alias">0</param>
          <param name="position">1</param>
          <param name="mode_of_operation">8</param>
          <param name="slave_config">$(find ethercat_motor_drive)/config/inovance_IS620N_config.yaml</param>
        </ec_module>
      </joint>      
      
      <joint name="joint_3">
        <state_interface name="position"/>
        <state_interface name="velocity"/>
        <state_interface name="effort"/>
        <command_interface name="position"/>
        <command_interface name="reset_fault"/>
        <ec_module name="Maxon">
          <plugin>ethercat_generic_plugins/EcCiA402Drive</plugin>
          <param name="alias">0</param>
          <param name="position">2</param>
          <param name="mode_of_operation">8</param>
          <param name="slave_config">$(find ethercat_motor_drive)/config/inovance_IS620N_config.yaml</param>
        </ec_module>
      </joint>    

      <joint name="joint_4">
        <state_interface name="position"/>
        <state_interface name="velocity"/>
        <state_interface name="effort"/>
        <command_interface name="position"/>
        <command_interface name="reset_fault"/>
        <ec_module name="Maxon">
          <plugin>ethercat_generic_plugins/EcCiA402Drive</plugin>
          <param name="alias">0</param>
          <param name="position">3</param>
          <param name="mode_of_operation">8</param>
          <param name="slave_config">$(find ethercat_motor_drive)/config/inovance_IS620N_config.yaml</param>
        </ec_module>
      </joint>           
    </ros2_control>

  </xacro:macro>

</robot>

```

修改 controllers.yaml 文件

```
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
      - joint_3
      - joint_4

velocity_controller:
  ros__parameters:
    joints:
      - joint_1
      - joint_2
      - joint_3
      - joint_4

effort_controller:
  ros__parameters:
    joints:
      - joint_1
      - joint_2
      - joint_3
      - joint_4

```

命令行方式让电机转起来：

```
ros2 topic pub -r 1 /trajectory_controller/joint_trajectory trajectory_msgs/msg/JointTrajectory '{header: {stamp: {sec: 0, nanosec: 0}, frame_id: ""}, joint_names: ["joint_1","joint_2","joint_3","joint_4"], points: [{positions: [1.0,1.0,1.0,1.0], velocities: [0.0,0.0,0.0,0.0], accelerations: [0.0,0.0,0.0,0.0], time_from_start: {sec: 1, nanosec: 0}},{positions: [19000000.0,19000000.0,19000000.0,19000000.0], velocities: [0.0,0.0,0.0,0.0], accelerations: [0.0,0.0,0.0,0.0], time_from_start: {sec: 4, nanosec: 0}}]}'


```

## 台达 ASDA-A2 伺服控制

配置代码：

```
# Configuration file for Maxon EPOS3 drive
vendor_id: 0x000001dd
product_id: 0x10305070
assign_activate: 0x0300  # DC Synch register
auto_fault_reset: false  # true = automatic fault reset, false = fault reset on rising edge command interface "reset_fault"

rpdo:  # RxPDO = receive PDO Mapping
  - index: 0x1601
    channels:
      - {index: 0x6040, sub_index: 0, type: uint16, default: 0}  # Control word
      - {index: 0x607a, sub_index: 0, type: int32, command_interface: position, default: .nan}  # Target position
      - {index: 0x60ff, sub_index: 0, type: int32, command_interface: velocity, default: .nan}  # Target position
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

然后报错 AL3E1， 时钟同步错误，

## 6 moveit2 servo 库的底层接口修改

moveit 库启动逻辑顺序：

1 启动 rviz，因为要做界面显示，如果是 RT 平台， 可以不用 rviz

2 启动 controller mananger ，这是 ROS 运行环境所必须的节点，用于监管其他节点的启动运行，运行方式为 ros2_control_node

3 启动 boardcaster 节点， 广播机械臂各轴状态的， 运行方式为 spawner ，被 controller mananger 加载完之后， 广播节点的启动过程会结束， 日志上会输出 process has finised cleanly

4 启动 机械臂的 fake 节点 ， 用来实现机械臂关节变化控制的恶， 方式为 spawner ，记载完之后也会显示 process has finised cleanly

节点的启动程序在 .launch.py 里一般这样写：

```
    return launch.LaunchDescription(
        [
            rviz_node,
            ros2_control_node,
            joint_state_broadcaster_spawner,
            panda_arm_controller_spawner,
            servo_node,
            container,
        ]
    )

```

## 7 ros_humble_base 上安装

阻止系统开机自升级

```
cd /etc/systemd/system/network-online.target.wants/
sudo nano systemd-networkd-wait-online.service

[Service]
Type=oneshot
ExecStart=/lib/systemd/systemd-networkd-wait-online
RemainAfterExit=yes
TimeoutStartSec=2sec

```

### 故障 launch.substitutions.substitution_failure.SubstitutionFailure: executable

'[<launch.substitutions.text_substitution.TextSubstitution object at 0x7f72314f7eb0>]' not found on the PATH

是因为没有安装包， 把其他包安装上就好了

### 'Failed to get logging directory, at ./src/rcl_logging_spdlog.cpp:83'

```
export HOME=/root
export ROS_LOG_DIR=/root/.ros/log

```

### 报错 ROS_DOMAIN_ID is not an integral number

```
vi ~/.bashrc

export ROS_DOMAIN_ID=10

```

### 后台运行

```
ros2 launch ethercat_motor_drive motor_drive.launch.py >> ~/log.txt 2>&1 &

ros2 topic pub -r 0.1 /trajectory_controller/joint_trajectory trajectory_msgs/msg/JointTrajectory '{header: {stamp: {sec: 0, nanosec: 0}, frame_id: ""}, joint_names: ["joint_1","joint_2","joint_3","joint_4"], points: [{positions: [1.0,1.0,1.0,1.0], velocities: [0.0,0.0,0.0,0.0], accelerations: [0.0,0.0,0.0,0.0], time_from_start: {sec: 1, nanosec: 0}},{positions: [19000000.0,19000000.0,19000000.0,19000000.0], velocities: [0.0,0.0,0.0,0.0], accelerations: [0.0,0.0,0.0,0.0], time_from_start: {sec: 4, nanosec: 0}}]}' >> ~/log2.txt  2>&1 &



```

### 补充安装其他的包

```
sudo apt install ros-humble-hardware-interface
sudo apt install ros-humble-xacro
sudo apt install ros-humble-imu-sensor-broadcaster
sudo apt install ros-humble-diff-drive-controller
sudo apt install ros-humble-position-controllers
sudo apt install ros-humble-gripper-controllers
sudo apt install ros-humble joint-state-broadcaster
sudo apt install ros-humble-joint-trajectory-controller
sudo apt install ros-humble-controller-manager

```

### 开机自启动

ethercat 启动必须用 root 账号，

rc.local

```
#!/bin/bash
source /opt/ros/humble/setup.bash
source /home/user1/ros2_ws/install/setup.bash
export ROS_DOMAIN_ID=10
export ROS_LOCALHOST_ONLY=1
export HOME=/root
export ROS_LOG_DIR=/root/.ros/log
/etc/init.d/ethercat start
sleep 2
bash /home/user1/nodered.sh
/home/user1/l.sh
/home/user1/l2.sh
#exit 1


```

l.sh

```
source /home/user1/.bashrc
ros2 launch ethercat_motor_drive motor_drive.launch.py >> ~/log.txt 2>&1 &


```

l2.sh

```
source /home/user1/.bashrc
ros2 topic pub -r 0.1 /trajectory_controller/joint_trajectory trajectory_msgs/msg/JointTrajectory '{header: {stamp: {sec: 0, nanosec: 0}, frame_id: ""}, joint_
names: ["joint_1","joint_2","joint_3","joint_4"], points: [{positions: [1.0,1.0,1.0,1.0], velocities: [0.0,0.0,0.0,0.0], accelerations: [0.0,0.0,0.0,0.0], time
_from_start: {sec: 1, nanosec: 0}},{positions: [19000000.0,19000000.0,19000000.0,19000000.0], velocities: [0.0,0.0,0.0,0.0], accelerations: [0.0,0.0,0.0,0.0],
time_from_start: {sec: 4, nanosec: 0}}]}' >> ~/log2.txt  2>&1 &


```

nodered.sh

```
more nodered.sh
export PATH=$PATH:/home/user1/downloads/node-v16.17.0-linux-x64/bin
node-red --setting /home/user1/settings.js >> /home/user1/nodered.log 2>&1 &

```

## 使用其他 ROS controller 来驱动电机

轨迹运动模式，既支持位置控制， 也支持速度控制

### 使用 纯位置模式

先修改 controller.yaml , 增加 forward_position_controller

```
controller_manager:
  ros__parameters:
    update_rate: 100  # Hz

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    trajectory_controller:
      type: joint_trajectory_controller/JointTrajectoryController
      
    forward_position_controller:
      type: forward_command_controller/ForwardCommandController 

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
      - joint_3
      - joint_4
      
forward_position_controller:
  ros__parameters:
    joints:
      - joint_1
      - joint_2
      - joint_3
      - joint_4
    interface_name: position      

velocity_controller:
  ros__parameters:
    joints:
      - joint_1
      - joint_2
      - joint_3
      - joint_4

effort_controller:
  ros__parameters:
    joints:
      - joint_1
      - joint_2
      - joint_3
      - joint_4


```

由于在文档内部， forward_position_controller 比 trajectory_controller 靠后，所以默认启动 trajectory_controller， forward_position_controller 没有启动， 启动方法：

```
ros2 control load_controller forward_position_controller 

ros2 control set_controller_state forward_position_controller inactive
ros2 control set_controller_state trajectory_controller inactive
ros2 control set_controller_state forward_position_controller active

```

然后就可以发命令了：

```
ros2 topic pub /forward_position_controller/commands std_msgs/msg/Float64MultiArray "data: 
- 11110.5 
- 11110.5 
- 11110.5 
- 11110.5"

```

### 使用 纯速度模式

在 controllers.yaml 里增加

```
velocity_controller:
  ros__parameters:
    command_interfaces:
      - velocity
    state_interfaces:
      - velocity  
    joints:
      - joint_1
      - joint_2
      - joint_3
      - joint_4

```

使用时：

```
ros2 topic pub /velocity_controller/commands std_msgs/msg/Float64MultiArray "data: 
- 1111110.5 
- 1111110.5 
- 1111110.5 
- 1111110.5"

```

xacro 文件里也要增加 velocity 的命令，并设置初始化模式为 CSV 模式

```
     <joint name="joint_1">
        <state_interface name="position"/>
        <state_interface name="velocity"/>
        <state_interface name="effort"/>
        <command_interface name="position"/>
        <command_interface name="velocity"/>
        <command_interface name="reset_fault"/>
        <ec_module name="Maxon">
          <plugin>ethercat_generic_plugins/EcCiA402Drive</plugin>
          <param name="alias">0</param>
          <param name="position">0</param>
          <param name="mode_of_operation">9</param>
          <param name="slave_config">$(find ethercat_motor_drive)/config/inovance_is620n_config.yaml</param>
        </ec_module>
      </joint>

```

切换各个控制器模式时，伺服电机的状态不会一起更改，就很麻烦

伺服电机配置文件要增加速度

```
- {index: 0x60ff, sub_index: 0, type: int32, command_interface: velocity, default: .nan}  

```