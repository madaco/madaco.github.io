---
title: "捕获HardFault触发前状态"
date: 2025-09-01T10:10:48+08:00
description: "捕获 HardFault 触发前寄存器状态"
categories: ["外设深究"]
tags: ["HardFault", "Cortex-M", "异常处理"]
---

# 捕获HardFault触发前状态

## 1.可行性分析

Cortex-M系列单片机会在进入HardFault异常时将多个**内核寄存器**压入栈，这些寄存器中包括了程序计数寄存器PC、链接寄存器LR，通过这两个可以定位触发HardFault异常的指令地址。

同时Cortex-M还提供了多个**错误状态寄存器**，HardFault状态寄存器SCB->HFSR、BusFault状态寄存器SCB->BFSR、MemManageFault状态寄存器SCB->MMFAR、UsageFault状态寄存器SCB->CFSR、DebugFault状态寄存器SCB->DFSR，这些寄存器代表了Fault系列异常的原因。

因此可以**在HardFault的异常处理函数中捕获内核寄存器和错误状态寄存器**，然后**写入到Flash**中，在上电后再读取Flash内存的值，获取捕获的寄存器值，最后通过数据分析触发HardFault前的状态。

## 2.实现过程

### 2.1重定义启动文件中的HardFault

压栈操作中使用的是MSP（主栈指针）或PSP（进程栈指针），可以通过它获取压栈的地址，而具体使用的是哪个栈需要依据EXC_RETURN的数值判断。

EXC_RETURN是一个特殊的异常返回值，当处理器进入异常处理时，链接寄存器LR的数值会被更新为EXC_RETURN数值，而EXC_RETURN的bit2代表了使用的栈类型。

![](/images/捕获HardFault触发前状态/EXC_RETURN位域.png)

因此可以先判断LR寄存器的bit2决定使用MSP还是PSP，然后将栈指针传给R0寄存器，跳转到异常处理函数中访问第一个传参就可以得到压栈的数据。

重定义启动文件startup.s中的HardFault汇编代码

```assembly
HardFault_Handler\
                PROC
                EXPORT  HardFault_Handler  [WEAK]
                IMPORT  HardFault_Handler_C  ;HardFault的异常处理函数
                
                TSR LR, #4    ;LR&4 
                ITE EQ        ;判断LR&4==0
                MRSEQ R0, MSP ;EQ条件满足，将MSP寄存器传给R0
                MRSNE R0, PSP ;EQ条件不满足，将PSP寄存器传给R0
               
                B HardFault_Handler_C  ;跳转到异常处理函数中
                B	.                  ;死循环。也可以在这里省略，在异常处理函数最后死循环
                ENDP
```

如果没有上操作系统，一直用MSP，不会使用PSP，代码可以简化成

```assembly
HardFault_Handler\
                PROC
                EXPORT  HardFault_Handler  [WEAK]
                IMPORT  HardFault_Handler_C  ;HardFault的异常处理函数
                
                MRS R0, MSP  		         ;将MSP寄存器传给R0
                B HardFault_Handler_C
                B	.                  ;死循环。也可以在这里省略，在异常处理函数最后死循环
                ENDP
```

### 2.2重定义HardFault异常处理函数

因为汇编代码中已经将栈指针传给了R0寄存器，所以第一个传参就是栈指针。直接访问栈指针获取压栈时的内核寄存器，然后再读取错误状态寄存器的值，一并写入到Flash中保存起来。

在获取压栈的寄存器时需要注意先后顺序，因此入栈时是按照xPSR、PC、LR、R12、R3-R0顺序入栈的，所以出栈要按照相反顺序。

![](/images/捕获HardFault触发前状态/入栈顺序.png)

```C
typedef struct {
    // 核心寄存器
    uint32_t r0, r1, r2, r3;
    uint32_t r12, lr, pc, psr;
    
    // 系统控制块寄存器
    uint32_t cfsr;   // 可配置故障状态寄存器
    uint32_t hfsr;   // 硬件故障状态寄存器
    uint32_t mmfar;  // 存储器管理地址寄存器
    uint32_t bfar;   // 总线故障地址寄存器
} HardFaultInfo;

volatile HardFaultInfo g_hardFaultInfo = {0};

void HardFault_Handler_C(uint32_t* stack_pointer) {
    // 1. 获取异常现场寄存器
    volatile HardFaultInfo* frame = (HardFaultInfo*)stack_pointer;
    
    // 2. 将关键信息存储到全局结构体
    g_hardFaultInfo.r0   = frame->r0;
    g_hardFaultInfo.r1   = frame->r1;
    g_hardFaultInfo.r2   = frame->r2;
    g_hardFaultInfo.r3   = frame->r3;
    g_hardFaultInfo.r12  = frame->r12;
    g_hardFaultInfo.lr   = frame->lr;
    g_hardFaultInfo.pc   = frame->pc; 
    g_hardFaultInfo.psr  = frame->psr;
    
    // 3. 存储系统控制块寄存器
    g_hardFaultInfo.cfsr  = SCB->CFSR;
    g_hardFaultInfo.hfsr  = SCB->HFSR;
    g_hardFaultInfo.mmfar = SCB->MMFAR;
    g_hardFaultInfo.bfar  = SCB->BFAR;
    
    //4.写入Flash
    HardFault_WriteTime++;     //刷入次数累加
    SramTest();
    while(1);
}
```
