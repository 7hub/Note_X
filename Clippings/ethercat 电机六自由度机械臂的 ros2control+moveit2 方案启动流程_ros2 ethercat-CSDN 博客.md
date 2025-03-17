---
url: https://blog.csdn.net/weixin_41652700/article/details/143746141
title: ethercat 电机六自由度机械臂的 ros2control+moveit2 方案启动流程_ros2 ethercat-CSDN 博客
date: 2025-03-17 10:12:06
tag: 
summary: 
---
## 零、启动过程

### 终端 1

```
sudo /etc/init.d/ethercat start

```

*   启动 ethercat 的主站服务，ethercat 不向 tcp 那种常见的网络服务被写进了操作系统内部，所以需要单独启动这个服务。

```
ethercat slaves

```

*   查看从机状态，确保硬件正常

### 终端 2

```
ros2 launch moveit_test demo.launch.py

```

*   启动 ros2control 和 moveit2

## 一、ros2 launch moveit_test demo.launch.py

### 1.1 MoveItConfigsBuilder(“erobot”, package_name=“moveit_test”)

*   这个函数定义机器人的名字是 “erobot”，软件包是 moveit_test。这个函数只是生成配置条件。
*   这个函数主要是利用机器人相关的配置文件生成了 "robot_description"，包括：**erobot.urdf.xacro 和 erobot.srdf、erobot.ros2_control.xacro**

### 1.2 generate_demo_launch(moveit_config)

![](https://i-blog.csdnimg.cn/direct/238e23f40a8a4d8589218fdb31ee34ba.png)

*   这是一个 python 文件，通过脚本的形式启动需要的 launch 文件。所有 launch 文件也是通过 py 文件启动的，就是上面这个图中的 py 文件，都是围绕 demo.launch.py 建立的，其他的都是为了启动不通的部分、
*   需要启动的包括以下文件：

```
    Includes
     * static_virtual_joint_tfs
     * robot_state_publisher
     * move_group
     * moveit_rviz
     * warehouse_db (optional)
     * ros2_control_node + controller spawners
    """

```

也就是需要的所有文件了，就是除了 **erobot.urdf.xacro 和 erobot.srdf、erobot.ros2_control.xacro** 以外下面这张图中的所有[配置文件](https://so.csdn.net/so/search?q=%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6&spm=1001.2101.3001.7020)（以机器人名开头的文件）  

![](https://i-blog.csdnimg.cn/direct/a8860a76f87a49d08197644cc5bbfe71.png)

*   这两节的区别在于一个是生成机器人描述，一个是启动 ros2control 和 moveit，之后将分别详述

## 二、机器人描述文件

![](https://i-blog.csdnimg.cn/direct/8789e8ea71db477396e0467925acf85f.png)

### 1、erobot.urdf.xacro

这里值得一说的就是 erobot.urdf.xacro 这个文件，调用了 erobot.ros2_control.xacro 文件，这个文件所需的文件生成了名为 FakeSystem 的硬件接口，这里还叫 FakeSystem 的原因是控制器直接调用的 moveit 自带的，自带的那个是没有接实际硬件的，所以叫 “虚假系统”  

![](https://i-blog.csdnimg.cn/direct/ce1e3e5c3a7b41a8a3774c570baa7fcd.png)

### 2、erobot.ros2_control.xacro

这个调用了[硬件接口](https://so.csdn.net/so/search?q=%E7%A1%AC%E4%BB%B6%E6%8E%A5%E5%8F%A3&spm=1001.2101.3001.7020)那个包的编译结果，连接了两个软件包

## 三、启动配置文件

配置文件中比较重要的就是 moveit_contorllers.yaml 和 ros2_controllers.yaml，分别配置了 moveit 中控制器管理者和 ros2control 的控制器管理者，moveit 中的管理者实际就是调用了 ros2control 的，只是增加了一部分内容。

```
# This config file is used by ros2_control
controller_manager:
  ros__parameters:
    update_rate: 100  # Hz

    p1_controller:   #定义控制器
      type: joint_trajectory_controller/JointTrajectoryController


    joint_state_broadcaster:  #定义广播类型
      type: joint_state_broadcaster/JointStateBroadcaster

p1_controller:      #给与参数
  ros__parameters:
    joints:
      - shoulder_joint
      - upperArm_joint
      - foreArm_joint
      - wrist1_joint
      - wrist2_joint
      - wrist3_joint
    command_interfaces:
      - position
    state_interfaces:
      - position
      - velocity


```