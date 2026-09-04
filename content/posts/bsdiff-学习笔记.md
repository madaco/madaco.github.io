---
title: "bsdiff 差分升级学习笔记"
date: 2025-09-01T10:12:39+08:00
description: "bsdiff/bspatch 差分升级思路与基础文件操作复习。"
categories: ["项目笔记"]
tags: ["bsdiff", "bspatch", "差分升级", "LZMA"]
---

bsdiff 官方源码 GitHub 地址：https://github.com/mendsley/bsdiff

## bsdiff + lzma 实现 MCU 差分升级的高 star 移植案例

- GitHub：https://github.com/ruiwarn/mcu_bsdiff_upgrade
- Gitee：https://gitee.com/qq791314247/mcu_bsdiff_upgrade
- CSDN 博客：https://blog.csdn.net/qq_35333978/article/details/128211763

## bsdiff 思路

1. 打开新文件和旧文件，并分别分配内存
2. 创建二进制差分文件，写入文件头（标志 + 新文件长度）
3. 准备好 lzma 压缩写入的条件
4. 利用 bsdiff 从旧文件和新文件提取出差异部分，lzma 压缩写入到二进制文件中
5. 关闭 lzma 压缩写入，释放新文件和旧文件的内存

## bspatch 思路

1. 打开差分文件
2. 从差分文件中获取文件头，判断头信息是否和预设的一致（特定），获取长度
3. 打开旧文件，为旧文件和新文件分配内存
4. 用 lzma 解压算法提取，边解压边和旧文件合并，存放到新文件中
5. 关闭 lzma 解压读取，释放新文件和旧文件的内存

## 基础文件操作（复习）

| POSIX | C 标准库 |
| :--- | :--- |
| open() | fopen() |
| read() | fread() |
| write() | fwrite() |
| close() | fclose() |
| lseek() | |
| fstat() | |

内存操作：`malloc()`、`free()`

字符操作：`memset()`、`memcmp()`、`memcpy()`

## 函数功能统一化的一种思路

结构体类型中定义用到的都会用到的变量指针和函数指针：

```c
struct bspatch_stream
{
    void* opaque;
    int (*read)(const struct bspatch_stream* stream, void* buffer, int length);
};
```

然后定义需要的类型后，直接在初始化时设置结构体成员，即变量指针和函数指针：

```c
BZFILE* bz2;                    // 专用于 Bzip2 压缩解压的类型
struct bspatch_stream stream;   // 创建一个结构体变量 stream 数据流
stream.read = bz2_read;         // 将 read 函数设置为 Bzip2 专用的解压读函数
stream.opaque = bz2;            // 这个结构体变量的特征，初始化之后就不会再改变
```

上面这种就是可以专用于 Bzip2 解压，同样的初始化不同类型，可以用作 lzma 等多种解压方法。
