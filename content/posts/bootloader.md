---
title: "bootloader"
date: 2025-09-01T10:10:48+08:00
description: "Bootloader 实现与跳转方法总结"
categories: ["外设深究"]
tags: ["Bootloader", "FLASH", "CAN"]
---

# bootloader

## 1. Bootloader 的实现方法

- 存储介质：NAND、eMMC、U盘、SD卡
- 通讯方式：USB、CAN、232、485、SPI、I2C、网络更新

## 2. 实际案例

- **BMU**：将 bootloader 程序存放到内部 FLASH 里，然后通过 CAN 通讯更新 app 层。
- **BC**：由 bootloader 层、备份层、app 层组成。
- **OV 手表**：同样将 bootloader 层和 app 层放到内部 FLASH 中，通过蓝牙更新 app 层，并且加入了一层 FLAG 区域，只有这一段区域数据正确时，才会从 bootloader 层跳转到 app 层。

## 3. 上电启动流程

1. 首先根据 boot0 和 boot1 引脚电平选择从哪个区域启动（选择将哪个存储区映射到 0 地址区）。
2. 读取栈顶地址初始化 MSP。
3. 读取中断服务程序首地址，进入复位中断服务程序。

## 4. 从 app 跳转回 bootloader

NVIC_SystemReset

## 5. boot 跳转 app 方法

1. 关闭全局中断、关闭所有中断、将所有时钟设置成默认状态、清除中断挂起标志位。
2. 关闭滴答定时器，复位到默认值。
3. 设置 MSP、设置 PSP；如果用到了 RTOS，需要加一行 `__set_CONTROL(0)`，设置成特权级模式，使用 MSP 指针。
4. 这里 app 层程序要重新设置中断向量表的偏移地址，跳转到 APP 层的复位中断服务程序。

## 针对一些疑问做实验

1. 如果采用用户 FLASH 启动程序，必须要以 `0x8000 0000` 地址启动，如果以 `0x8100 0000` 启动就会出问题。
   - 猜测是因为 ARM 默认将 `0x8000 0000` 地址开启的数据映射到了 0 地址，因为首个和次个地址存放着栈顶地址和复位中断服务程序地址，如果以 `0x8100 0000` 启动程序就找不到栈顶地址，且找不到正确的复位中断服务程序，就会出问题。
2. 验证：https://blog.csdn.net/qq_20312079/article/details/108283594
3. 看 `__set_CONTROL(0)` 执行了什么。
