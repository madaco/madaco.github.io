---
title: "关于Ucos-II任务调度的理解"
date: 2025-09-01T10:12:39+08:00
description: "解析Ucos-II任务调度机制、PendSV上下文切换与临界区管理"
categories: ["操作系统"]
tags: ["Ucos", "RTOS", "任务调度", "PendSV"]
---

理解来源：https://www.cnblogs.com/han-bing/p/9015134.html

## Ucos-II是如何执行任务调度的？

1. Ucos-II的systick里会更新所有优先级任务的就绪状态，更新挂起任务的延时时间，当低优先级的任务运行中，如果有高优先级的任务变为就绪状态，那么触发一次滴答中断后，低优先级任务会挂起，执行任务切换，运行高优先级的任务。
2. 任务延时函数OSTimeDly()会记下当前任务的延时时间，并挂起任务，寻找就绪任务中优先级最高的任务，执行任务切换。
3. 通过OSTaskSuspend()函数挂起指定任务。

## 通过OSTimeDly()函数进行任务切换的过程

当前任务变成非就绪态->寻找就绪态中优先级最高的任务->触发PendSV异常中断->任务切换

## PendSV异常中断详解：

PendSV是可挂起的系统调用，系统级别的异常，支持缓期执行。
Ucos-II是在systick中断中进行任务切换的，而如果程序在其他中断中运行，此时触发了systick中断，systick因为优先级会抢占其他中断运行，如果systick中又执行了任务切换，会使得其他中断执行延后。并且如果systick进行任务切换后操作系统试图返回线程模式，此时仍有活跃的其他中断，会触发UsageFault。

![](/images/Ucos-II/ISR执行期间的上下文切换回延迟中断服务.png)

解决方法：在有其他中断触发时，Ucos-II不进行任务切换。任务切换放在PendSV异常中处理，将PendSV的优先级设置成最小，这样当systick中断打断了其他中断后，systick中断处理函数中将任务切换交给了PendSV处理，由于PendSV优先级低，它会等待其他中断执行后再执行任务切换。

![](/images/Ucos-II/PendSV上下文切换.png)

## 几个重要的任务变量

OSTCBCur->OSTCBY    :当前任务的优先级组编号
OSTCBCur->OSTCBX    :当前任务的优先级组内编号
OSTCBCur->OSTCBBitY :用于逻辑运算的二进制变量，优先级组编号的bit偏移量
OSTCBCur->OSTCBBitX :用于逻辑运算的二进制变量，优先级组内编号的bit偏移量

OSRdyTbl[OS_RDY_TBL_SIZE] :任务就绪状态，代表每个优先级组的组内任务就绪状态，每个bit代表某一个任务的就绪状态
OSRdyGrp                  :组就绪状态，代表组任务就绪状态，每个bit代表这个一个任务组的就绪状态
OSUnMapTbl[256]           :索引表，通过查表法找到最高优先级的任务

## OS_ENTER_CRITICAL()和OS_EXIT_CRITICAL()函数作用

OS_ENTER_CRITICAL()是进入临界区，禁止中断。

OS_EXIT_CRITICAL()是退出临界区。

具体原理如下：

```C
/*
*********************************************************************************************************
*                                 Cortex-M3
*                        Critical Section Management
									   临界区管理
*
* Method #1:  Disable/Enable interrupts using simple instructions.  After critical section, interrupts will be enabled even if they were disabled before entering the critical section.
方法1：使用简单的指令禁用/使能中断。进入临界区之后，即便是之前被禁用的中断也会被使能。
*             NOT IMPLEMENTED
					未实施
*
* Method #2:  Disable/Enable interrupts by preserving the state of interrupts.  In other words, if interrupts were disabled before entering the critical section, they will be disabled when leaving the critical section.
方法2：通过保存中断状态禁用/使能中断。换句话说，如果中断在进入临界区前被禁用了，当他们离开临界区后仍然会被禁用。
*             NOT IMPLEMENTED
					未实施
*
* Method #3:  Disable/Enable interrupts by preserving the state of interrupts.  Generally speaking you would store the state of the interrupt disable flag in the local variable 'cpu_sr' and then disable interrupts.  'cpu_sr' is allocated in all of uC/OS-II's functions that need to disable interrupts.  You would restore the interrupt disable state by copying back 'cpu_sr' into the CPU's status register.
方法3：通过保存中断状态禁用/使能中断。通常来说使用局部变量cpu_sr保存中断的禁用标志，然后禁用中断。cpu_sr用于所有需要禁用中断的所有ucos-II函数，你将通过将cpu_sr复制到CPU状态寄存器来恢复中断禁用状态。
*********************************************************************************************************
*/
#define  OS_CRITICAL_METHOD   3u //使用第三种临界区方法

#if OS_CRITICAL_METHOD == 3u
#define  OS_ENTER_CRITICAL()  {cpu_sr = OS_CPU_SR_Save();}
#define  OS_EXIT_CRITICAL()   {OS_CPU_SR_Restore(cpu_sr);}
#endif
```

可以看到Ucos-II提供了三种临界区管理的方法，这里使用的第三种，使用局部变量cpu_sr保存中断状态，进入临界区前保存状态，然后禁用中断，执行用户自定义代码后，将cpu_sr的值再复制到CPU的状态寄存器恢复进入前的中断状态。

OS_ENTER_CRITICAL() 和OS_EXIT_CRITICAL()函数分别代表进入临界区和退出临界区的函数，他们是通过汇编函数实现的，可以找到函数定义的位置。

```assembly
OS_CPU_SR_Save
    MRS     R0, PRIMASK    ; Set prio int mask to mask all (except faults)
    CPSID   I			   ;禁用中断
    BX      LR
OS_CPU_SR_Restore
    MSR     PRIMASK, R0
    BX      LR
```

PRIMASK寄存器是Cortex-M架构下的特殊寄存器，用于禁用除NMI和HardFault异常之外的中断，实际上是将当前优先级设置成0(最高的可编程等级)。
CPSID的英文全称是change processor state and interrupt disable，改变处理器状态、禁用中断
CPSIE的英文全称是change processor state and interrupt enable，改变处理器状态、使能中断
OS_CPU_SR_Save()函数进入临界区后先将PRIMASK的值赋值给R0通用寄存器，禁用中断，然后返回调用点，将PRIMASK的值返回给了cpu_sr。

OS_CPU_SR_Restore()函数退出临界区是将输入的变量cpu_sr保存到R0寄存器，然后将R0的值再赋值给PRIMASK寄存器，恢复到进入前的状态。

这里就有个疑问，为什么退出的时候不使用CPSIE呢？这是考虑了临界区嵌套的缘故，如果有多层进入临界区，这时候最内层如果退出使用CPSIE会将直接将中断使能，那么外层临界区代码执行时就可能产生中断，就会出问题。
