---
title: "配置S32K1xx的MBD开发环境"
date: 2025-10-08T21:43:20+08:00
description: "S32K1xx 的 MBD 开发环境配置与编译测试"
categories: ["Matlab"]
tags: ["S32K1xx", "MBD", "NXP", "Matlab"]
---

## 1. 下载 NXP_MBDToolbox_S32K1xx_4.3.0_20220913.mltbx

### 1.1 进入官网 [www.nxp.com/mctoolbox](http://www.nxp.com/mctoolbox)，登录 NXP 账号，按照下面步骤下载软件

![](/images/配置S32K1xx的MBD开发环境/下载页面1.png)

![](/images/配置S32K1xx的MBD开发环境/下载页面2.png)

![](/images/配置S32K1xx的MBD开发环境/下载页面3.png)

![](/images/配置S32K1xx的MBD开发环境/下载页面4.png)

![](/images/配置S32K1xx的MBD开发环境/下载页面5.png)

### 1.2 下载许可证

![](/images/配置S32K1xx的MBD开发环境/配置许可证1.png)

![](/images/配置S32K1xx的MBD开发环境/配置许可证2.png)

![](/images/配置S32K1xx的MBD开发环境/配置许可证3.png)

![](/images/配置S32K1xx的MBD开发环境/配置许可证4.png)

![](/images/配置S32K1xx的MBD开发环境/配置许可证5.png)

## 2. 安装 NXP_MBDToolbox_S32K1xx_4.3.0_20220913.mltbx

### 2.1 安装 Toolbox

下载完成后会获得下面两个文件，zip 文件重命名为 .mltbx 类型，双击打开后自动跳转到 matlab 附加功能窗口，按照提示栏安装。

![](/images/配置S32K1xx的MBD开发环境/下载图片1.png)

![](/images/配置S32K1xx的MBD开发环境/下载图片2.png)

### 2.2 复制许可证文件

将许可证文件放到刚安装的 Toolbox 工具目录下 `{..\MATLAB\Add Ons\Toolboxes\NXP_MBDToolbox_S32K1xx\lic}`。

## 3. 配置环境

### 3.1 配置编译器环境变量

配置系统环境变量，配置如下所示，配置完后重启电脑。

![](/images/配置S32K1xx的MBD开发环境/配置环境变量.png)

```
GCC_S32K_TOOL = {Toolbox安装路径}/tools/gcc-6.3-arm32-eabi

IAR_TOOL = {IAR安装路径}/IAR Systems/Embedded Workbench 7.3

GHS_TOOL = {GHS安装路径}/multi517
```

### 3.2 设置 MBD 工具路径

直接在 matlab 命令行输入 `mbd_s32k_path`，软件会自动配置。

![](/images/配置S32K1xx的MBD开发环境/配置.png)

## 4. 测试模型案例，编译烧录

1. 找到测试模型

![](/images/配置S32K1xx的MBD开发环境/测试1.png)

2. 按照如下步骤打开 USART 的测试模型

![](/images/配置S32K1xx的MBD开发环境/测试2.png)

![](/images/配置S32K1xx的MBD开发环境/测试3.png)

![](/images/配置S32K1xx的MBD开发环境/测试4.png)

![](/images/配置S32K1xx的MBD开发环境/测试5.png)

3. 打开后配置 MCU 的工具链和连接方式

![](/images/配置S32K1xx的MBD开发环境/测试x.png)

![](/images/配置S32K1xx的MBD开发环境/测试6.png)

![](/images/配置S32K1xx的MBD开发环境/测试7.png)

4. 编译模型

![](/images/配置S32K1xx的MBD开发环境/测试8.png)

![](/images/配置S32K1xx的MBD开发环境/测试9.png)

5. 输出窗口显示如图代表编译成功

![](/images/配置S32K1xx的MBD开发环境/测试10.png)
