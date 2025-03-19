---
url: https://blog.csdn.net/a1347065101/article/details/131944246
title: EtherCAT——PDO/SDO_ethercat pdo-CSDN 博客
date: 2025-03-18 11:52:04
tag: 
summary: 
---
**目录**

[数据传输方式：PDO/SDO](#t0)

[应用层协议](#t1)

## [数据传输](https://so.csdn.net/so/search?q=%E6%95%B0%E6%8D%AE%E4%BC%A0%E8%BE%93&spm=1001.2101.3001.7020)方式：PDO/SDO

主站与从站进行数据交互的方式主要通过 [PDO](https://so.csdn.net/so/search?q=PDO&spm=1001.2101.3001.7020) 和 SDO，即过程数据和邮箱数据其概念与 CANOpen 中的概念相同。

*   **PDO（过程数据对象）**：过程数据用来传输周期性的数据，PDO 由三个数据缓冲区组成，类似于一个 FIFO，从站写入第一个缓冲区，主站从第三个缓冲区读走。注意第二个缓冲区不可操作。从站发送 PDO 和接受 PDO 各自采用两个独立的数据缓冲区。有同步管理器来控制缓冲区，每一个同步管理器只负责一种功能，例如同步管理器 2 负责发送 PDO，同步管理器 3 负责接受 PDO。
*   **SDO（服务数据对象）**：邮箱通信用来发送非周期性的数据，邮箱通信只有一个数据缓冲区，通信方式采用握手的机制确保主从之间的数据交互不丢失，而 PDO 由于采用 FIFO 的机制，可能会出现新值覆盖旧值或旧值被多次读走的情况。SDO 也由同步管理器来进行管理，发送和接受邮箱独立控制，例如同步管理器 0 控制发送邮箱，同步管理器 1 控制接受邮箱。

## [应用层协议](https://so.csdn.net/so/search?q=%E5%BA%94%E7%94%A8%E5%B1%82%E5%8D%8F%E8%AE%AE&spm=1001.2101.3001.7020)

*   **COE** (CANopen over EtherCAT) : 该协议是一个具有开放性的标准应用层协议。其中 CANopen 协议是基于 CAN 协议的链路层上实现的。EtherCAT 协议在应用层支持 CANopen 协议，因此支持 CANopen 协议的从站可以被运用在 EtherCAT 协议上。
*   **SOE** (Sercos over EtherCAT) : SERCOS 是世界首个应用于伺服控制的协议。EtherCAT 协议在应用层接口上兼容了这个协议，简称为 SOE。SERCOS 应用层协议为主站设计了信息接口，可以通过配置 EtherCAT 过程数据报文，实现周期性传递伺服驱动器的数据。
*   **EOE** (EtherNet over EtherCAT) : 该协议支持 EtherCAT 能分段传递标准的以太网数据报文，使得 EtherCAT 协议同样能支持 TCP/IP、UDP/IP 协议。
*   **FOE** (File Access over EtherCAT) : 该协议可以使用 EtherCAT 总线上传、下载固件，刷新从站的固件。并且可以通过命令行工具加载或存储文件。