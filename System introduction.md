# System introduction

系统的逻辑功能图如下所示

![LoigcFunction](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Image/LoigcFunction.PNG?raw=true)

<center>图0.1 系统逻辑功能图</center>
整个系统主要可以分为以下四个部分：

1. 时钟复位：连接外部时钟和复位端口，为各个模块产生时钟驱动以及复位
2. 通信：主要分为与上位机通信的以太网模块，与外设通信的串口
3. 外围：主要分为AD驱动，DDS驱动
4. 数据处理：处理通信以及AD来的数据

## 时钟复位

![ClockDesign](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Image/ClockDesign.PNG?raw=true)

<center>图1.1 时钟设计图</center>
对于FPGA来说，整个芯片的外围时钟主要分为三部分：

1. 系统时钟SYS_CLK ： 由此产生系统主时钟 以及需要和主时钟同步的时钟
2. AD数据时钟ADC_CLK：这个时钟由AD给出用于同步AD数据，同时也用来作为DDS的源时钟
3. 以太网数据时钟ETH_CLK：这个时钟经由RGMII接口而来，用于同步以太网接收数据

## 通信

![CommunicationBlock](https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Image/CommunicationBlock.PNG?raw=true)

通信最主要分为两个方向：以太网通信和串口通信

以太网通信主要作用：

1. 上传AD采样的数据
2. 设置光模块参数 DDS参数 AD参数
3. 查询系统状态

串口通信主要作用：

1. 设置光模块参数
2. 查询光模块状态

## 外围

<img src="https://github.com/Lvwings/FiberAcquisitionCard/blob/feature/Image_update/Image/PeripheralCircuit.PNG?raw=true" alt="PeripheralCircuit" style="zoom:67%;" />

主要的外围电路联系为：AD驱动和DDS驱动

在AD驱动中：

1. AD的时钟由外部晶振给出，不需要FPGA提供
2. AD返回的时钟不与系统时钟同步

在DDS驱动中：

1. DDS的开启与AD采样的开启密切相关，因此时钟源为ADC_CLK产生
2. 在DDS输出连接ADG901用于开关DDS信号，需要控制