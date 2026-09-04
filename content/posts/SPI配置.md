---
title: "MiniFOC的SPI外设配置"
date: 2025-09-01T10:12:39+08:00
description: "MiniFOC 中 GD32 SPI 外设与 SC60228 磁编码器配置。"
categories: ["项目笔记"]
tags: ["SPI", "GD32", "SC60228", "编码器"]
---

## 1. 背景

BLDC 需要用编码器测量转子当前位置，这里使用了赛卓的 12 位磁编码角度传感器 SC60228，采用的 SPI 通讯方式。

## 2. 引脚选择

查阅 [GD32F130 用户手册](Doc/GD32F1x0_User_Manual_Rev3.8_CN.pdf#page=221)，选择使用 SPI0 对应的四引脚。

![GD32F130 SPI 引脚](/images/SPI配置/GD32F130SPI引脚.PNG)

## 3. SPI 时钟

通过查询 [GD32F130 的时钟树](Doc/GD32F1x0_User_Manual_Rev3.8_CN.pdf#page=25)，可以看到 SPI0 挂载在 APB2 外设总线上，我这里 APB2 外设总线时钟设置的是最大频率 72MHz。

![GD32F130 时钟树 2](/images/SPI配置/GD32F130时钟树2.png)

## 4. SPI 参数

SPI 可以选择使用全双工、半双工、单工模式。

SPI 可以选择主机模式还是从机模式。

SPI 可以数据帧是 8 位还是 16 位。

SPI 可以选择片选引脚选通是通过硬件管理还是软件管理。

SPI 可以通过时钟极性和相位来控制数据传输的时机。

SPI 可以选择数据传输时 MSB 还是 LSB。

SPI 可以选择数据传输时是否使用硬件 CRC 校验功能。

SPI 可以通过预分频器设置 SPI 时钟频率。

## 0. 示例代码

```c
/*!
    \brief configure spi0 periph and its gpios
*/
void spi_config(void) {
    spi_parameter_struct spi_init_struct;

    /* enable GPIO clock and SPI0 clock*/
    rcu_periph_clock_enable(RCU_GPIOA);
    rcu_periph_clock_enable(RCU_SPI0);

    /* SPI0 GPIO config: SCK/PA5, MISO/PA6, MOSI/PA7 */
    gpio_af_set(GPIOA, GPIO_AF_0, GPIO_PIN_5 | GPIO_PIN_6 | GPIO_PIN_7);
    gpio_mode_set(GPIOA, GPIO_MODE_AF, GPIO_PUPD_NONE, GPIO_PIN_5 | GPIO_PIN_6 | GPIO_PIN_7);
    gpio_output_options_set(GPIOA, GPIO_OTYPE_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_5 | GPIO_PIN_6 | GPIO_PIN_7);

    /* SPI0 GPIO config: CS/PA4 */
    gpio_mode_set(GPIOA, GPIO_MODE_OUTPUT, GPIO_PUPD_NONE, GPIO_PIN_4);
    gpio_output_options_set(GPIOA, GPIO_OTYPE_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_4);

    /* pull up CS pin to end sending data */
    gpio_bit_set(GPIOA, GPIO_PIN_4);

    /* SPI0 parameter config */
    spi_init_struct.trans_mode = SPI_TRANSMODE_FULLDUPLEX;
    spi_init_struct.device_mode = SPI_MASTER;
    spi_init_struct.frame_size = SPI_FRAMESIZE_16BIT;
    spi_init_struct.clock_polarity_phase = SPI_CK_PL_LOW_PH_2EDGE;
    spi_init_struct.nss = SPI_NSS_SOFT;
    spi_init_struct.prescale = SPI_PRESCALE;
    spi_init_struct.endian = SPI_ENDIAN_MSB;
    spi_crc_off(SPI0);
    spi_init(SPI0, &spi_init_struct);

    /* SPI0 enable */
    spi_enable(SPI0);
}
```

```c
/*!
    \brief      spi0 transmit data for sc60228
    \param[in]  data: data to transmit
    \retval     data received from slave
*/
unsigned short spi_readwrite_halfword(unsigned short data) {
    unsigned short buffer;
    while (RESET == spi_i2s_flag_get(SPI0, SPI_FLAG_TBE));

    /* send this data through spi0 */
    spi_i2s_data_transmit(SPI0, data);
    while (RESET == spi_i2s_flag_get(SPI0, SPI_FLAG_RBNE));

    /* get data received through spi0 */
    buffer = spi_i2s_data_receive(SPI0);
    return buffer;
}
```
