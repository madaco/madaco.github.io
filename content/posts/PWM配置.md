---
title: "MiniFOC的PWM外设配置"
date: 2025-09-01T10:12:39+08:00
description: "MiniFOC 中 GD32 PWM 外设的配置与占空比更新。"
categories: ["项目笔记"]
tags: ["PWM", "GD32", "CAPWM", "定时器"]
---

## 1. 背景

BLDC 需要用 3 个独立的 PWM 通道控制三相电压的大小。

## 2. 引脚选择

由于该方案选用的是基于 SVPWM 的 FOC 控制策略，因此 PWM 要使用中央对齐模式的，并且还需要带 3 个通道的，查阅 GD32F130 用户手册，这里选用通用定时器 L0 里的 TIMER 1。

![GD32F1x0 定时器分类图](/images/PWM配置/GD32定时器分类图.png)

选择定时器后，再选择定时器通道，根据 GD32F130 数据手册，这里选用 TIMER1_CH0、TIMER1_CH1、TIMER1_CH2，对应引脚 PA0、PA1、PA2。

![GD32F130Gx 系列引脚定义](/images/PWM配置/GD32F130Gx系列引脚定义.png)

## 3. PWM 时钟

通过查询 GD32F130 用户手册的时钟树，可以看到 TIMER1 挂载在 APB1 外设总线上，我这里 APB1 外设总线时钟设置的是最大频率 72MHz。

![GD32F130 时钟树](/images/PWM配置/GD32F130时钟树.png)

TIMER1 的时钟源有四种选择：内部时钟 CK_TIMER、内部输入触发 ITIx、外部输入捕获 CHx_IN、外部触发输入 ETI。这里采用内部时钟 CK_TIMER，而内部时钟来源就是 APB1，因此定时器时钟 TIMER_CK = 72MHz。TIMER_CK 之后的预分频器设置为 71，得到预分频器时钟 PSC_CLK = TIMER_CK / (PSC + 1) = 72MHz / (71 + 1) = 1MHz。同样计数器频率为 1MHz，1us 计数一次。

![通用定时器 L0 结构框图](/images/PWM配置/通用定时器L0结构框图.png)

## 4. PWM 参数

GD32 的 PWM 有两种类型，EAPWM（边沿对齐 PWM）和 CAPWM（中央对齐 PWM），实际上 EAPWM 就是向上计数或向下计数模式，CAPWM 就是中央对齐模式，我这里使用的是 CAPWM。

PWM 设置里的两个重要参数是自动重装载值 CAR 和捕获比较值 CHxCV，CAPWM 模式下，PWM 的周期 = 2×自动重装载值，占空比也由 (2×捕获比较值) 决定。

PWM0 和 PWM1 模式决定了在超过捕获比较值时输出电压为高电平还是低电平。

PWM 可以配置极性为高电平还是低电平，代表了哪种为有效电平。

PWM 可以配置定时器时钟（CK_TIMER）与死区时间和数字滤波器采样时钟（DTS）之间的分频系数，这个是用于设置 DTS 的频率。

![GD32 的 CAPWM 时序图](/images/PWM配置/GD32的CAPWM时序图.png)

PWM 可以选择是否使能影子寄存器缓存功能，该功能决定了在 CAR 或 PSC 值发生变化时，是否立刻变化，如果开启缓存功能，会在下一个更新事件发生时变化，下图中展示了影子寄存器的缓存功能。我这里选择关闭缓存，立刻生效。

![影子寄存器功能展示](/images/PWM配置/GD32定时器的影子寄存器功能.png)

## 5. 更新占空比

电机在运转过程中 PWM 需要经常变动的值就是占空比，这里可以直接调用相应的函数改变捕获比较值 CHx_CV。

## 6. 示例代码

```c
/*!
    \brief configure timer1 periph and its gpios
*/
void pwm_config(void) {
    timer_oc_parameter_struct timer_ocintpara;
    timer_parameter_struct timer_initpara;

    /* enable GPIO clock and TIMER1 clock*/
    rcu_periph_clock_enable(RCU_GPIOA);
    rcu_periph_clock_enable(RCU_TIMER1);
    timer_deinit(TIMER1);

    /* configure TIMER1_CH0 as alternate function push-pull */
    gpio_mode_set(GPIOA, GPIO_MODE_AF, GPIO_PUPD_NONE, GPIO_PIN_0);
    gpio_output_options_set(GPIOA, GPIO_OTYPE_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_0);

    /* configure TIMER1_CH1 as alternate function push-pull */
    gpio_mode_set(GPIOA, GPIO_MODE_AF, GPIO_PUPD_NONE, GPIO_PIN_1);
    gpio_output_options_set(GPIOA, GPIO_OTYPE_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_1);

    /* configure TIMER1_CH2 as alternate function push-pull */
    gpio_mode_set(GPIOA, GPIO_MODE_AF, GPIO_PUPD_NONE, GPIO_PIN_2);
    gpio_output_options_set(GPIOA, GPIO_OTYPE_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_2);

    /* connect port to TIMER1_CH0 1 2 */
    gpio_af_set(GPIOA, GPIO_AF_2, GPIO_PIN_0);
    gpio_af_set(GPIOA, GPIO_AF_2, GPIO_PIN_1);
    gpio_af_set(GPIOA, GPIO_AF_2, GPIO_PIN_2);

    /* TIMER1CLK = SystemCoreClock / 2 = 18MHz */
    timer_initpara.prescaler = 1;

    /* timer center alignment up count mode */
    timer_initpara.alignedmode = TIMER_COUNTER_CENTER_UP;
    timer_initpara.counterdirection = TIMER_COUNTER_UP;

    /* Period = 36000 / PWM_FREQUENCY */
    timer_initpara.period = PWM_PERIOD;
    timer_initpara.clockdivision = TIMER_CKDIV_DIV1;
    timer_initpara.repetitioncounter = 0;
    timer_init(TIMER1, &timer_initpara);

    /* CH1, CH2 and CH3 configuration in PWM0 mode */
    timer_ocintpara.ocpolarity = TIMER_OC_POLARITY_HIGH;
    timer_ocintpara.outputstate = TIMER_CCX_ENABLE;
    timer_channel_output_config(TIMER1, TIMER_CH_0, &timer_ocintpara);
    timer_channel_output_config(TIMER1, TIMER_CH_1, &timer_ocintpara);
    timer_channel_output_config(TIMER1, TIMER_CH_2, &timer_ocintpara);

    /* CH0 configuration in PWM mode1,duty cycle 50% */
    timer_channel_output_pulse_value_config(TIMER1, TIMER_CH_0, PWM_PERIOD / 2);
    timer_channel_output_mode_config(TIMER1, TIMER_CH_0, TIMER_OC_MODE_PWM0);
    timer_channel_output_shadow_config(TIMER1, TIMER_CH_0, TIMER_OC_SHADOW_DISABLE);

    /* CH1 configuration in PWM mode1,duty cycle 25% */
    timer_channel_output_pulse_value_config(TIMER1, TIMER_CH_1, PWM_PERIOD / 2);
    timer_channel_output_mode_config(TIMER1, TIMER_CH_1, TIMER_OC_MODE_PWM0);
    timer_channel_output_shadow_config(TIMER1, TIMER_CH_1, TIMER_OC_SHADOW_DISABLE);

    /* CH2 configuration in PWM mode1,duty cycle 12.5% */
    timer_channel_output_pulse_value_config(TIMER1, TIMER_CH_2, PWM_PERIOD / 2);
    timer_channel_output_mode_config(TIMER1, TIMER_CH_2, TIMER_OC_MODE_PWM0);
    timer_channel_output_shadow_config(TIMER1, TIMER_CH_2, TIMER_OC_SHADOW_DISABLE);

    /* auto-reload preload and timer enable */
    timer_auto_reload_shadow_enable(TIMER1);
    timer_enable(TIMER1);
}

/*!
    \brief     update timer1 ch0 1 2 duty-cycle
    \param[in] ch0: duty-cycle of channel0, 0 ~ 1.0f
    \param[in] ch1: duty-cycle of channel1, 0 ~ 1.0f
    \param[in] ch2: duty-cycle of channel2, 0 ~ 1.0f
*/
void update_pwm_dutycycle(float ch0, float ch1, float ch2) {
    /* update the comparison register of timer1 */
    TIMER_CH0CV(TIMER1) = (uint32_t) ((float) PWM_PERIOD * ch0);
    TIMER_CH1CV(TIMER1) = (uint32_t) ((float) PWM_PERIOD * ch1);
    TIMER_CH2CV(TIMER1) = (uint32_t) ((float) PWM_PERIOD * ch2);
}
```
