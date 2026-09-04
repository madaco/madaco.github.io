---
title: "simulink和stateflow设计soc预估算法"
date: 2025-09-01T10:14:48+08:00
description: "SOC 末端修正算法与安时积分误差"
categories: ["Matlab"]
tags: ["Simulink", "Stateflow", "SOC预估"]
---

## 末端修正算法

- **目的**：消除安时积分法带来的累计误差
- **逻辑**：当电池静置一段时间后，采用开路电压法计算较为准确的 SOC，计算与安时积分法的误差，为防止跳变，在下一次充放电过程中采用安时积分校准系数，将误差补偿回来。

视频见链接：https://www.bilibili.com/video/BV19H4y1D7Yw/?spm_id_from=333.337.search-card.all.click&vd_source=af1a9d4adba029707439882bbe78ebb1

## 安时积分法的两个误差

1. 初始 SOC 不准确
2. 积分造成的累计误差
