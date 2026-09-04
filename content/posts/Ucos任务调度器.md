---
title: "Ucos任务调度器"
date: 2025-09-01T10:12:39+08:00
description: "讲解调度器类型并逐步实现一个合作式调度器"
categories: ["操作系统"]
tags: ["Ucos", "RTOS", "调度器", "合作式调度"]
---

## 调度器类型

调度器的类型：合作式调度、抢占式调度、时间片调度。

调度器可以区分就绪态任务和挂起任务，调度器可以根据任务延时、信号量等待、邮箱等待、事件组等待等原因将任务调度，然后选择就绪态的一个任务，激活它。不同调度器的区别就是如何分配就绪态任务间的完成时间。

合作式调度：每一个时刻只有一个任务可以运行，任务之间不支持抢占，直到该任务自愿放弃CPU的控制权。
抢占式调度：当前任务运行时，当有更高优先级任务进入到就绪态，当前任务会被挂起，更高优先级任务被执行。
时间片调度：

## 设计一个合作式调度器

设计一个合作式调度器

1. 调度器的数据结构 ：用作存储信息
2. 初始化函数          ：初始化滴答定时器
3. 滴答定时器中断   ：定时刷新任务的信息，包括Delay Period RunMe
4. 向调度器增加任务函数 ：将任务添加到任务队列中
5. 使任务在应当运行的时候被执行的调度函数 ：任务调度中的任务切换函数
6. 从调度器删除任务的函数 (此案例中没有设计)

### 1.调度器的数据结构

```C
typedef unsigned char    tByte;
typedef unsigned int     tWord;
typedef  struct
{
	void  (*pTask)();        /*指向任务的指针必须是一个*void(void)*函数；*/
	tWord  Delay;          /*延时（时标）知道下一个函数的运行*/
	tWord  Period;        /*连续运行之间的间隔*/
	tByte RunMe;          /*当任务需要运行的时候由调度器加1*/
}sTask;

任务队列的大小通过下面进行定义：
#define  SCH_MAX_TASKS     5
sTask SCH_task_G[SCH_MAX_TASKS];  /*建立的任务数*/
```

### 2.初始化函数

清零所有的软件定时器，初始化滴答定时器的参数和配置，设置中断周期

```C
/*
*******************************************************************************************
* 函 数 名: bsp_InitTimer
* 功能说明: 配置 systick 中断，并初始化软件定时器变量
* 形 参: 无
* 返 回 值: 无
*******************************************************************************************
*/
void bsp_InitTimer(void)
{
    uint8_t i;
    /* 清零所有的软件定时器 */
    for (i = 0; i < TMR_COUNT; i++)
    {
    s_tTmr[i].Count = 0;
    s_tTmr[i].PreLoad = 0;
    s_tTmr[i].Flag = 0;
    s_tTmr[i].Mode = TMR_ONCE_MODE; /* 缺省是 1 次性工作模式 */
}
/*
配置 systic 中断周期为 1ms，并启动 systick 中断。
 SystemCoreClock 是固件中定义的系统内核时钟，对于 STM32F4XX,一般为 168MHz
 SysTick_Config() 函数的形参表示内核时钟多少个周期后触发一次 Systick 定时中断.
 -- SystemCoreClock / 1000 表示定时频率为 1000Hz， 也就是定时周期为 1ms
 -- SystemCoreClock / 500 表示定时频率为 500Hz， 也就是定时周期为 2ms
 -- SystemCoreClock / 2000 表示定时频率为 2000Hz， 也就是定时周期为 500us
 对于常规的应用，我们一般取定时周期 1ms。对于低速 CPU 或者低功耗应用，可以设置定时周期为 10ms
 */
	SysTick_Config(SystemCoreClock / 1000);
}
```

### 3.滴答定时器中断

刷新任务的信息。遍历所有任务，延时Delay自减，减到0将间隔Period赋值给延时Delay

```c
/*
*******************************************************************************************
* 函 数 名: SCH_Update(void)
* 功能说明: 调度器的刷新函数，每个时标中断执行一次。在嘀嗒定时器中断里面执行。
* 当刷新函数确定某个任务要执行的时候，将 RunMe 加 1，要注意的是刷新任务
* 不执行任何函数，需要运行的任务有调度函数激活。
* 形 参：无
* 返 回 值: 无
*******************************************************************************************
*/
void SCH_Update(void)
{
    tByte index;
    /*注意计数单位是时标，不是毫秒*/
    for(index = 0; index < SCH_MAX_TASKS; index++)
    {
        /*检测这里是否有任务*/
        if(SCH_task_G[index].pTask)
        {
            if(SCH_task_G[index].Delay == 0)
            {
                /*任务需要运行 将 RunMe 置 1*/
                SCH_task_G[index].RunMe += 1;
                if(SCH_task_G[index].Period)
                {
                    /*调度周期性的任务再次执行*/
                    SCH_task_G[index].Delay = SCH_task_G[index].Period;
                }
			}
         	else
            {
                /*还有准备好运行*/
                SCH_task_G[index].Delay -= 1;
            }
        }
    }
}
```

### 4.添加任务函数

将任务添加到任务队列中

tByte SCH_Add_Task(void (*pFuntion)(void), tWord DELAY, tWord PERIOD)

1. void (*pFuntion)(void) ：表示函数的地址，也就是将函数名填进去就行了。
2. DELAY ：表示函数第一次运行需要等待的时间。
3. PERIOD：表示函数周期性执行的时间间隔

```c
/*
*******************************************************************************************
* 函 数 名: SCH_Add_Task
* 功能说明: 添加任务。
* 形 参：void (*pFuntion)(void) tWord DELAY tWord PERIOD
* 返 回 值: 返回任务的 ID 号
*******************************************************************************************
*/
tByte SCH_Add_Task(void (*pFuntion)(void), tWord DELAY, tWord PERIOD)
{
    tByte index = 0; /*首先在队列中找到一个空隙，（如果有的话）*/
    while((SCH_task_G[index].pTask != 0) && (index <SCH_MAX_TASKS))
    {
    	index ++;
    }
    if(index == SCH_MAX_TASKS)/*超过最大的任务数目 则返错误信息*/
    {
        Error_code_G = ERROR_SCH_TOO_MANY_TASKS;/*设置全局错误变量*/
        return SCH_MAX_TASKS;
    }
    SCH_task_G[index].pTask = pFuntion; /*运行到这里说明申请的任务块成功*/
    SCH_task_G[index].Delay = DELAY;
    SCH_task_G[index].Period = PERIOD;
    SCH_task_G[index].RunMe =0;
	return index; /*返回任务的位置，以便于以后删除*/
}
```

### 5.调度函数

激活执行任务

```C
/*
*******************************************************************************************
* 函 数 名: SCH_Dispatch_Tasks
* 功能说明: 在主任务里面执行的调度函数。
* 形 参：无
* 返 回 值: 无
*******************************************************************************************
*/
void SCH_Dispatch_Tasks(void)
{
    tByte index;
    /*运行下一个任务，如果下一个任务准备就绪的话*/
    for(index = 0; index < SCH_MAX_TASKS; index++)
    {
    	if(SCH_task_G[index].RunMe >0)
    	{
			/*执行任务 */
			(*SCH_task_G[index].pTask)();
            /* 执行任务完成后，将 RunMe 减一 */
            SCH_task_G[index].RunMe -= 1;
            /*如果是单次任务的话，则将任务删除 */
            if(SCH_task_G[index].Period == 0)
            {
                SCH_Task_Delete(index);
			}
		}
    }
}
```

### 6.总结

这种合作式调度器有三个缺点

1. 只有一个中断

2. 需要满足任务运行时间<任务周期间隔时间

3. 可能会有任务运行重叠，一任务时间延时会影响后一任务
