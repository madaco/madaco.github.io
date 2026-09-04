---
title: "OVWatch"
date: 2025-09-01T10:12:39+08:00
description: "OVWatch 项目的软件层、硬件层与 LVGL 层设计笔记。"
categories: ["项目笔记"]
tags: ["OVWatch", "FreeRTOS", "LVGL", "嵌入式"]
---

## 软件层

### 1. FreeRTOS 中消息队列的应用

消息队列与信号量的区别：消息队列可以传递数据。

### 2. FreeRTOS 定时器的应用

创建定时器、设置时间，每次定时器达到计时时间，在回调函数中执行相应动作。

### 3. FreeRTOS 关闭所有任务调度

相应 API：`vTaskSuspendAll()`、`xTaskResumeAll()`。

### 4. FreeRTOS 初始化思路

使用一个最高优先级任务执行初始化函数，并且执行中关闭其他所有任务调度，然后任务执行完后删除任务 `vTaskDelete(NULL);`。

### 5. CMSIS Cortex-M 系列微处理器软件标准接口

（Cortex Microcontroller Software Interface Standard）

### 6. 软件分层

应用层、中间层、驱动层：

- 应用层：task、bootloader
- 中间层：freertos 库文件、lvgl 库文件
- 驱动层：封装 MCU 引脚控制函数

## 硬件层

### 1. MCU 供电方案

锂电池经升压降压芯片转换供电，用锂电池充电芯片给供电源充电。

### 2. 多个传感器

多个设备挂载到 IIC 总线上，且 SDA 和 SCL 采用开漏输出 + 上拉电阻。

### 3. 需要存储用户数据

使用 EEPROM 掉电保存数据。

### 4. 看门狗

模拟开关 + 看门狗芯片，隔离作用，自由选择是否启用看门狗。

## LVGL 层

### 1. 设计思路

square line studio 设计界面 → code block 上 LVGL 仿真 → 移植到 keil。

### 2. square line studio 添加自定义图标和自定义字体

a. 阿里巴巴矢量图库上下载文件，移到工程上的 assert 文件夹下

b. square line 添加新字体或新图标，选择对应文件，设置大小，图标另需添加十六进制数字（下载时有对应代码）配置，并将代码用 unicode 转换工具转换图标存到配置中。
