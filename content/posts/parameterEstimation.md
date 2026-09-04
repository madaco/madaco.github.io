---
title: "Parameter Estimation模型临摹笔记"
date: 2025-09-01T10:14:48+08:00
description: "Simulink 求解器类型与 ode 算法选择笔记"
categories: ["Matlab"]
tags: ["Simulink", "求解器", "Parameter Estimation"]
---

## 1. 模型配置参数的求解器设置

我自己在临摹时模型配置的求解器默认选择的是 **“自动(自动求解器选择)”**，原模型选择的是 **“ode 15s(stiff/NDF)”**，这里的不同会导致我自己的模型仿真结果和原模型不同，那么这里的选择的求解器是什么意思呢？

![](/images/parameterEstimation/变步长求解器.png)

![](/images/parameterEstimation/定步长求解器.png)

- Simulink 的求解器分为两类，**变步长求解器**和**定步长求解器**。变步长求解器会自动调整步长，在变化快时用小步长，在变化慢时用大步长，以平衡计算的精度和时间。定步长求解器步长是固定的，用于实时仿真和**生成代码**的情况。
- 对于变步长求解器和定步长求解器都有多种类型，ode 的英文全称是 **Ordinary Differential Equation**，中文含义是**常微分方程**，因为 Simulink 模型中的动态系统通常是用常微分方程描述的，因此**求解器的作用**就是通过数值方法**求解这些微分方程**，从而模拟系统的行为。
- 求解器 ode 名称后的数字表示算法的阶数或组合，后缀字母表示特定用法或算法类型。类如 ode45 中的 4 和 5 表示该算法同时使用 4 阶和 5 阶的龙格-库塔法；ode23 中的 2 和 3 代表 2 阶和 3 阶的组合，原理类似；ode15s 中的 15 是算法版本标识，s 表示适用于刚性系统(stiff systems)；ode1/ode2/ode3/ode4/ode5 中的数字直接表示算法的阶数。阶数越高精度越高，计算量也更大；阶数越低，计算速度会快，但是误差可能累加。

更详细的解释有博主已经写了篇帖子：https://blog.csdn.net/yangren123456789/article/details/127976713
