---
title: "你好，世界：博客开张了"
date: 2026-09-05T00:00:00+08:00
description: "第一篇博文，聊聊这个博客要记什么，顺便验证代码高亮"
tags: ["随笔", "嵌入式"]
categories: ["随笔"]
draft: false
---

这是我的个人技术博客，主要用来沉淀自己折腾硬件的记录，后续会慢慢覆盖这些方向：

- 单片机开发（STM32 等）
- Linux 板子、Linux 驱动、Linux 内核
- C / C++ 开发
- AUTOSAR 架构
- 机器人开发
- 自己画硬件、写代码、调试、画结构，做一些小玩具

## 为什么开这个博客

很多踩坑过程，如果当时不记下来，过几个月就忘了。写下来既能给自己留档，也能帮到遇到同样问题的人。

## 顺便验证一下代码高亮

下面是一段常见的单片机点灯代码：

```c
#include "main.h"

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    /* 使能 GPIO 时钟，初始化 LED 引脚 */
    MX_GPIO_Init();

    while (1)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);  // 翻转 LED
        HAL_Delay(500);                          // 延时 500ms
    }
}
```

以后的文章会以「**问题 → 思路 → 代码/电路 → 结果**」的方式来写，方便复现和回顾。

欢迎交流 👋
