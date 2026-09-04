---
title: "e2 studio将代码封装成lib库并应用"
date: 2025-09-01T10:10:48+08:00
description: "在 e2 studio 中将代码封装成 lib 库并应用"
categories: ["软件技巧"]
tags: ["e2 studio", "lib库", "静态库", "Renesas"]
---

# e2 studio将代码封装成lib库并应用

## 一.软件版本

e2 studio：2021-01 (21.1.0)

fsp：2.3.0

## 二.操作步骤

### 1.创建静态库工程

![](/images/e2-studio将代码封装成lib库并应用/创建工程-1.png)

![](/images/e2-studio将代码封装成lib库并应用/创建工程-2.png)

![](/images/e2-studio将代码封装成lib库并应用/创建工程-3.png)

![](/images/e2-studio将代码封装成lib库并应用/创建工程-4.png)

![](/images/e2-studio将代码封装成lib库并应用/创建工程-5.png)

![](/images/e2-studio将代码封装成lib库并应用/创建工程-6.png)

### 2.添加源代码，源文件实现头文件声明

![](/images/e2-studio将代码封装成lib库并应用/库源代码.png)

![](/images/e2-studio将代码封装成lib库并应用/库头文件.png)

### 3.编译生成.a静态库文件，库文件改名为lib开头文件，这里改成libtest.a

![](/images/e2-studio将代码封装成lib库并应用/库文件.png)

![](/images/e2-studio将代码封装成lib库并应用/库文件-改名.png)

### 4.将.a文件和.h头文件放到应用工程下

![](/images/e2-studio将代码封装成lib库并应用/导入应用.png)

### 5.设置工程属性，在编译器交叉链接属性中选择库文件，输入文件名默认去除lib前缀，

![](/images/e2-studio将代码封装成lib库并应用/应用工程设置.png)

### 6.编译应用程序，生成可执行文件
