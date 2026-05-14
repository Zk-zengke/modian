# 05 软硬 I2C 读写与 OLED 示例

本文用 STM32F10x 标准库 SPL 风格，对比软 I2C 和硬件 I2C 的读写函数，并给一个控制 SSD1306 OLED 的例子。

示例约定：

```text
芯片: STM32F103
库: STM32 Standard Peripheral Library
OLED: SSD1306 I2C OLED
OLED 7 位地址: 0x3C
OLED 写地址: 0x78
OLED 读地址: 0x79
```

本文代码统一使用 **8 位 I2C 地址**：

```c
#define OLED_ADDR_WRITE  0x78
#define OLED_ADDR_READ   0x79
```

也就是：

```text
写地址 = 7 位地址 << 1 | 0
读地址 = 7 位地址 << 1 | 1
```

如果你的工程习惯传 7 位地址，就需要在发送地址前自己左移一位。

## 1. 软 I2C 和硬 I2C 的核心区别

硬件 I2C：

```text
使用 STM32 内部 I2C 外设
通过 SR1/SR2/DR 等寄存器工作
有 I2C_EVENT_xxx 事件
速度稳定，CPU 占用低
```

软 I2C：

```text
使用普通 GPIO 模拟 I2C 时序
没有 I2C 状态寄存器
没有 I2C_EVENT_xxx 事件
靠拉高/拉低 SCL、SDA 和读取 SDA 完成通信
引脚灵活，调试直观
```

简单对应关系：

| 功能 | 硬件 I2C | 软 I2C |
|---|---|---|
| START | `I2C_GenerateSTART()` | `SoftI2C_Start()` |
| STOP | `I2C_GenerateSTOP()` | `SoftI2C_Stop()` |
| 发地址 | `I2C_Send7bitAddress()` | `SoftI2C_SendByte(addr)` |
| 发数据 | `I2C_SendData()` | `SoftI2C_SendByte(data)` |
| 收数据 | `I2C_ReceiveData()` | `SoftI2C_ReadByte()` |
| 等待完成 | `I2C_WaitEvent()` | 延时 + `SoftI2C_WaitAck()` |
| 判断 ACK | 硬件事件/状态位 | 释放 SDA 后读取 SDA |

## 2. 软 I2C 底层代码

这里假设软 I2C 使用：

```text
PB10 -> SCL
PB11 -> SDA
```

### 2.1 GPIO 初始化

```c
#include "stm32f10x.h"

#define SOFT_I2C_PORT GPIOB
#define SOFT_I2C_SCL  GPIO_Pin_10
#define SOFT_I2C_SDA  GPIO_Pin_11

#define SCL_HIGH() GPIO_SetBits(SOFT_I2C_PORT, SOFT_I2C_SCL)
#define SCL_LOW()  GPIO_ResetBits(SOFT_I2C_PORT, SOFT_I2C_SCL)

#define SDA_HIGH() GPIO_SetBits(SOFT_I2C_PORT, SOFT_I2C_SDA)
#define SDA_LOW()  GPIO_ResetBits(SOFT_I2C_PORT, SOFT_I2C_SDA)

#define SDA_READ() GPIO_ReadInputDataBit(SOFT_I2C_PORT, SOFT_I2C_SDA)

void SoftI2C_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);

    GPIO_InitStructure.GPIO_Pin = SOFT_I2C_SCL | SOFT_I2C_SDA;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_OD;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(SOFT_I2C_PORT, &GPIO_InitStructure);

    SCL_HIGH();
    SDA_HIGH();
}
```

注意：

```c
GPIO_Mode_Out_OD
```

表示开漏输出。软 I2C 推荐使用开漏输出，并且 SCL/SDA 要有上拉电阻。

在开漏模式下：

```text
SDA_LOW()  -> 主动拉低 SDA
SDA_HIGH() -> 释放 SDA，让上拉电阻拉高
```

### 2.2 延时函数

```c
static void SoftI2C_Delay(void)
{
    volatile uint16_t i;

    for (i = 0; i < 30; i++)
    {
    }
}
```

如果通信不稳定，可以先把 `30` 调大，例如 `80` 或 `100`。

### 2.3 START 和 STOP

```c
static void SoftI2C_Start(void)
{
    SDA_HIGH();
    SCL_HIGH();
    SoftI2C_Delay();

    SDA_LOW();
    SoftI2C_Delay();

    SCL_LOW();
    SoftI2C_Delay();
}

static void SoftI2C_Stop(void)
{
    SCL_LOW();
    SDA_LOW();
    SoftI2C_Delay();

    SCL_HIGH();
    SoftI2C_Delay();

    SDA_HIGH();
    SoftI2C_Delay();
}
```

START：

```text
SCL 高电平时，SDA 从高变低
```

STOP：

```text
SCL 高电平时，SDA 从低变高
```

这两个是 I2C 的特殊信号。普通数据位才要求 SCL 高电平时 SDA 保持稳定。

### 2.4 发送 1 个字节

```c
static void SoftI2C_SendByte(uint8_t byte)
{
    uint8_t i;

    for (i = 0; i < 8; i++)
    {
        SCL_LOW();

        if (byte & 0x80)
        {
            SDA_HIGH();
        }
        else
        {
            SDA_LOW();
        }

        SoftI2C_Delay();

        SCL_HIGH();
        SoftI2C_Delay();

        byte <<= 1;
    }

    SCL_LOW();
}
```

I2C 发送数据是高位先发，也就是先发 bit7。

### 2.5 等待 ACK

```c
static uint8_t SoftI2C_WaitAck(void)
{
    uint8_t ack;

    SDA_HIGH();
    SoftI2C_Delay();

    SCL_HIGH();
    SoftI2C_Delay();

    if (SDA_READ() == Bit_RESET)
    {
        ack = 0;
    }
    else
    {
        ack = 1;
    }

    SCL_LOW();
    SoftI2C_Delay();

    return ack;
}
```

返回值约定：

```text
0: 收到 ACK
1: 没收到 ACK
```

主机发完 8 位后必须释放 SDA：

```c
SDA_HIGH();
```

然后在第 9 个时钟读取 SDA。

如果从机拉低 SDA，就是 ACK。

### 2.6 主机发送 ACK / NACK

```c
static void SoftI2C_SendAck(void)
{
    SCL_LOW();
    SDA_LOW();
    SoftI2C_Delay();

    SCL_HIGH();
    SoftI2C_Delay();

    SCL_LOW();
    SDA_HIGH();
    SoftI2C_Delay();
}

static void SoftI2C_SendNack(void)
{
    SCL_LOW();
    SDA_HIGH();
    SoftI2C_Delay();

    SCL_HIGH();
    SoftI2C_Delay();

    SCL_LOW();
    SoftI2C_Delay();
}
```

读多个字节时：

```text
前面的字节读完后，主机发 ACK
最后一个字节读完后，主机发 NACK
```

### 2.7 读取 1 个字节

```c
static uint8_t SoftI2C_ReadByte(void)
{
    uint8_t i;
    uint8_t byte = 0;

    SDA_HIGH();

    for (i = 0; i < 8; i++)
    {
        byte <<= 1;

        SCL_LOW();
        SoftI2C_Delay();

        SCL_HIGH();
        SoftI2C_Delay();

        if (SDA_READ() == Bit_SET)
        {
            byte |= 0x01;
        }
    }

    SCL_LOW();

    return byte;
}
```

读数据时，主机释放 SDA，让从机控制 SDA。

## 3. 软 I2C 读写函数

### 3.1 软 I2C 写多个字节

```c
uint8_t SoftI2C_WriteBytes(uint8_t dev_addr_write, uint8_t reg_addr, uint8_t *buf, uint8_t len)
{
    uint8_t i;

    if (len == 0)
    {
        return 1;
    }

    SoftI2C_Start();

    SoftI2C_SendByte(dev_addr_write);
    if (SoftI2C_WaitAck())
    {
        SoftI2C_Stop();
        return 2;
    }

    SoftI2C_SendByte(reg_addr);
    if (SoftI2C_WaitAck())
    {
        SoftI2C_Stop();
        return 3;
    }

    for (i = 0; i < len; i++)
    {
        SoftI2C_SendByte(buf[i]);
        if (SoftI2C_WaitAck())
        {
            SoftI2C_Stop();
            return 4;
        }
    }

    SoftI2C_Stop();
    return 0;
}
```

时序：

```text
START
设备地址 + W
ACK
寄存器地址
ACK
data0
ACK
data1
ACK
...
STOP
```

### 3.2 软 I2C 读多个字节

```c
uint8_t SoftI2C_ReadBytes(uint8_t dev_addr_write, uint8_t dev_addr_read, uint8_t reg_addr, uint8_t *buf, uint8_t len)
{
    uint8_t i;

    if (len == 0)
    {
        return 1;
    }

    SoftI2C_Start();

    SoftI2C_SendByte(dev_addr_write);
    if (SoftI2C_WaitAck())
    {
        SoftI2C_Stop();
        return 2;
    }

    SoftI2C_SendByte(reg_addr);
    if (SoftI2C_WaitAck())
    {
        SoftI2C_Stop();
        return 3;
    }

    SoftI2C_Start();

    SoftI2C_SendByte(dev_addr_read);
    if (SoftI2C_WaitAck())
    {
        SoftI2C_Stop();
        return 4;
    }

    for (i = 0; i < len; i++)
    {
        buf[i] = SoftI2C_ReadByte();

        if (i == len - 1)
        {
            SoftI2C_SendNack();
        }
        else
        {
            SoftI2C_SendAck();
        }
    }

    SoftI2C_Stop();
    return 0;
}
```

时序：

```text
START
设备地址 + W
ACK
寄存器地址
ACK
RESTART
设备地址 + R
ACK
data0
ACK
data1
ACK
...
last_data
NACK
STOP
```

## 4. 硬件 I2C 初始化

这里使用硬件 I2C1：

```text
PB6 -> I2C1_SCL
PB7 -> I2C1_SDA
```

```c
void HardI2C1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    I2C_InitTypeDef I2C_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_OD;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOB, &GPIO_InitStructure);

    I2C_DeInit(I2C1);

    I2C_InitStructure.I2C_Mode = I2C_Mode_I2C;
    I2C_InitStructure.I2C_DutyCycle = I2C_DutyCycle_2;
    I2C_InitStructure.I2C_OwnAddress1 = 0x00;
    I2C_InitStructure.I2C_Ack = I2C_Ack_Enable;
    I2C_InitStructure.I2C_AcknowledgedAddress = I2C_AcknowledgedAddress_7bit;
    I2C_InitStructure.I2C_ClockSpeed = 100000;
    I2C_Init(I2C1, &I2C_InitStructure);

    I2C_Cmd(I2C1, ENABLE);
}
```

硬件 I2C 的 SCL/SDA 要配置成：

```c
GPIO_Mode_AF_OD
```

也就是复用开漏输出。

## 5. 硬件 I2C 等待函数

```c
#define I2C_TIMEOUT  10000

static uint8_t HardI2C_WaitBusy(void)
{
    uint32_t timeout = I2C_TIMEOUT;

    while (I2C_GetFlagStatus(I2C1, I2C_FLAG_BUSY) == SET)
    {
        if (timeout-- == 0)
        {
            return 1;
        }
    }

    return 0;
}

static uint8_t HardI2C_WaitEvent(uint32_t event)
{
    uint32_t timeout = I2C_TIMEOUT;

    while (I2C_CheckEvent(I2C1, event) != SUCCESS)
    {
        if (timeout-- == 0)
        {
            return 1;
        }
    }

    return 0;
}

static uint8_t HardI2C_WaitFlag(FlagStatus status, uint32_t flag)
{
    uint32_t timeout = I2C_TIMEOUT;

    while (I2C_GetFlagStatus(I2C1, flag) != status)
    {
        if (timeout-- == 0)
        {
            return 1;
        }
    }

    return 0;
}
```

这里硬件 I2C 等的是外设状态：

```text
BUSY
START 已发送
地址已发送并应答
字节已发送
字节已接收
```

软 I2C 没有这些状态位，所以软 I2C 只能自己读 SDA。

## 6. 硬件 I2C 写多个字节

```c
uint8_t HardI2C_WriteBytes(uint8_t dev_addr_write, uint8_t reg_addr, uint8_t *buf, uint8_t len)
{
    uint8_t i;

    if (len == 0)
    {
        return 1;
    }

    if (HardI2C_WaitBusy())
    {
        return 2;
    }

    I2C_GenerateSTART(I2C1, ENABLE);
    if (HardI2C_WaitEvent(I2C_EVENT_MASTER_MODE_SELECT))
    {
        I2C_GenerateSTOP(I2C1, ENABLE);
        return 3;
    }

    I2C_Send7bitAddress(I2C1, dev_addr_write, I2C_Direction_Transmitter);
    if (HardI2C_WaitEvent(I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED))
    {
        I2C_GenerateSTOP(I2C1, ENABLE);
        return 4;
    }

    I2C_SendData(I2C1, reg_addr);
    if (HardI2C_WaitEvent(I2C_EVENT_MASTER_BYTE_TRANSMITTED))
    {
        I2C_GenerateSTOP(I2C1, ENABLE);
        return 5;
    }

    for (i = 0; i < len; i++)
    {
        I2C_SendData(I2C1, buf[i]);
        if (HardI2C_WaitEvent(I2C_EVENT_MASTER_BYTE_TRANSMITTED))
        {
            I2C_GenerateSTOP(I2C1, ENABLE);
            return 6;
        }
    }

    I2C_GenerateSTOP(I2C1, ENABLE);
    return 0;
}
```

这个函数和软 I2C 写函数的流程本质一样：

```text
START -> 地址写 -> 寄存器地址 -> 数据 -> STOP
```

区别是硬件 I2C 每一步要等 `I2C_EVENT_xxx`。

## 7. 硬件 I2C 读多个字节

STM32F1 的硬件 I2C 读多个字节时 ACK 控制比较麻烦。下面给一个适合学习理解的版本，支持 `len >= 1`。

```c
uint8_t HardI2C_ReadBytes(uint8_t dev_addr_write, uint8_t dev_addr_read, uint8_t reg_addr, uint8_t *buf, uint8_t len)
{
    uint8_t i;

    if (len == 0)
    {
        return 1;
    }

    if (HardI2C_WaitBusy())
    {
        return 2;
    }

    I2C_AcknowledgeConfig(I2C1, ENABLE);

    I2C_GenerateSTART(I2C1, ENABLE);
    if (HardI2C_WaitEvent(I2C_EVENT_MASTER_MODE_SELECT))
    {
        I2C_GenerateSTOP(I2C1, ENABLE);
        return 3;
    }

    I2C_Send7bitAddress(I2C1, dev_addr_write, I2C_Direction_Transmitter);
    if (HardI2C_WaitEvent(I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED))
    {
        I2C_GenerateSTOP(I2C1, ENABLE);
        return 4;
    }

    I2C_SendData(I2C1, reg_addr);
    if (HardI2C_WaitEvent(I2C_EVENT_MASTER_BYTE_TRANSMITTED))
    {
        I2C_GenerateSTOP(I2C1, ENABLE);
        return 5;
    }

    I2C_GenerateSTART(I2C1, ENABLE);
    if (HardI2C_WaitEvent(I2C_EVENT_MASTER_MODE_SELECT))
    {
        I2C_GenerateSTOP(I2C1, ENABLE);
        return 6;
    }

    I2C_Send7bitAddress(I2C1, dev_addr_read, I2C_Direction_Receiver);

    if (len == 1)
    {
        if (HardI2C_WaitEvent(I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED))
        {
            I2C_GenerateSTOP(I2C1, ENABLE);
            return 7;
        }

        I2C_AcknowledgeConfig(I2C1, DISABLE);
        I2C_GenerateSTOP(I2C1, ENABLE);

        if (HardI2C_WaitEvent(I2C_EVENT_MASTER_BYTE_RECEIVED))
        {
            return 8;
        }

        buf[0] = I2C_ReceiveData(I2C1);
    }
    else
    {
        if (HardI2C_WaitEvent(I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED))
        {
            I2C_GenerateSTOP(I2C1, ENABLE);
            return 9;
        }

        for (i = 0; i < len; i++)
        {
            if (i == len - 1)
            {
                I2C_AcknowledgeConfig(I2C1, DISABLE);
                I2C_GenerateSTOP(I2C1, ENABLE);
            }

            if (HardI2C_WaitEvent(I2C_EVENT_MASTER_BYTE_RECEIVED))
            {
                I2C_GenerateSTOP(I2C1, ENABLE);
                return 10;
            }

            buf[i] = I2C_ReceiveData(I2C1);
        }
    }

    I2C_AcknowledgeConfig(I2C1, ENABLE);
    return 0;
}
```

读函数的本质流程：

```text
START
设备地址 + W
寄存器地址
RESTART
设备地址 + R
读数据
STOP
```

读最后一个字节前要关闭 ACK，这样主机读完最后一个字节后给从机 NACK，表示不继续读了。

## 8. OLED 控制原理

SSD1306 OLED 的 I2C 地址通常是：

```text
7 位地址: 0x3C
写地址: 0x78
```

OLED I2C 写入时，需要先发送一个控制字节：

```text
0x00: 后面是命令
0x40: 后面是显示数据
```

所以发送命令：

```text
START
0x78
ACK
0x00
ACK
cmd
ACK
STOP
```

发送数据：

```text
START
0x78
ACK
0x40
ACK
data
ACK
STOP
```

这里可以把 `0x00` 或 `0x40` 当成前面读写函数里的 `reg_addr`。

所以 OLED 写命令可以直接调用：

```c
SoftI2C_WriteBytes(OLED_ADDR_WRITE, 0x00, &cmd, 1);
HardI2C_WriteBytes(OLED_ADDR_WRITE, 0x00, &cmd, 1);
```

OLED 写数据可以调用：

```c
SoftI2C_WriteBytes(OLED_ADDR_WRITE, 0x40, &data, 1);
HardI2C_WriteBytes(OLED_ADDR_WRITE, 0x40, &data, 1);
```

## 9. OLED 示例代码

下面代码通过一个宏选择使用软 I2C 或硬件 I2C。

```c
#define OLED_ADDR_WRITE  0x78
#define OLED_ADDR_READ   0x79

#define OLED_USE_SOFT_I2C  1
```

### 9.1 OLED 底层写函数

```c
static void OLED_WriteCommand(uint8_t cmd)
{
#if OLED_USE_SOFT_I2C
    SoftI2C_WriteBytes(OLED_ADDR_WRITE, 0x00, &cmd, 1);
#else
    HardI2C_WriteBytes(OLED_ADDR_WRITE, 0x00, &cmd, 1);
#endif
}

static void OLED_WriteData(uint8_t data)
{
#if OLED_USE_SOFT_I2C
    SoftI2C_WriteBytes(OLED_ADDR_WRITE, 0x40, &data, 1);
#else
    HardI2C_WriteBytes(OLED_ADDR_WRITE, 0x40, &data, 1);
#endif
}

static void OLED_WriteDataBuf(uint8_t *buf, uint16_t len)
{
    uint16_t i;

    for (i = 0; i < len; i++)
    {
        OLED_WriteData(buf[i]);
    }
}
```

这里为了简单，每次写一个显示数据字节。

如果想提高刷新速度，可以一次发送多个数据字节：

```text
START -> 0x78 -> 0x40 -> data0 -> data1 -> data2 -> ... -> STOP
```

### 9.2 OLED 设置位置

SSD1306 常见分辨率是 `128x64`。

它通常分成 8 页：

```text
page 0: 第 0 到 7 行
page 1: 第 8 到 15 行
...
page 7: 第 56 到 63 行
```

设置显示位置：

```c
static void OLED_SetPos(uint8_t x, uint8_t page)
{
    OLED_WriteCommand(0xB0 + page);
    OLED_WriteCommand(0x00 + (x & 0x0F));
    OLED_WriteCommand(0x10 + ((x >> 4) & 0x0F));
}
```

### 9.3 OLED 清屏

```c
void OLED_Clear(void)
{
    uint8_t page;
    uint8_t x;

    for (page = 0; page < 8; page++)
    {
        OLED_SetPos(0, page);

        for (x = 0; x < 128; x++)
        {
            OLED_WriteData(0x00);
        }
    }
}
```

### 9.4 OLED 全屏点亮

```c
void OLED_Fill(void)
{
    uint8_t page;
    uint8_t x;

    for (page = 0; page < 8; page++)
    {
        OLED_SetPos(0, page);

        for (x = 0; x < 128; x++)
        {
            OLED_WriteData(0xFF);
        }
    }
}
```

### 9.5 OLED 初始化

```c
void OLED_Init(void)
{
#if OLED_USE_SOFT_I2C
    SoftI2C_Init();
#else
    HardI2C1_Init();
#endif

    OLED_WriteCommand(0xAE); // display off
    OLED_WriteCommand(0x20); // memory addressing mode
    OLED_WriteCommand(0x10); // page addressing mode
    OLED_WriteCommand(0xB0); // page start address
    OLED_WriteCommand(0xC8); // COM scan direction
    OLED_WriteCommand(0x00); // low column address
    OLED_WriteCommand(0x10); // high column address
    OLED_WriteCommand(0x40); // start line address
    OLED_WriteCommand(0x81); // contrast
    OLED_WriteCommand(0x7F);
    OLED_WriteCommand(0xA1); // segment remap
    OLED_WriteCommand(0xA6); // normal display
    OLED_WriteCommand(0xA8); // multiplex ratio
    OLED_WriteCommand(0x3F);
    OLED_WriteCommand(0xA4); // display follows RAM
    OLED_WriteCommand(0xD3); // display offset
    OLED_WriteCommand(0x00);
    OLED_WriteCommand(0xD5); // display clock divide
    OLED_WriteCommand(0x80);
    OLED_WriteCommand(0xD9); // pre-charge period
    OLED_WriteCommand(0xF1);
    OLED_WriteCommand(0xDA); // COM pins config
    OLED_WriteCommand(0x12);
    OLED_WriteCommand(0xDB); // VCOMH deselect level
    OLED_WriteCommand(0x40);
    OLED_WriteCommand(0x8D); // charge pump
    OLED_WriteCommand(0x14);
    OLED_WriteCommand(0xAF); // display on

    OLED_Clear();
}
```

不同 OLED 模块可能初始化命令略有差异。如果屏幕显示方向反了，通常改这两条：

```c
OLED_WriteCommand(0xA1);
OLED_WriteCommand(0xC8);
```

可以尝试换成：

```c
OLED_WriteCommand(0xA0);
OLED_WriteCommand(0xC0);
```

### 9.6 在 OLED 上显示一个简单图案

这个例子不依赖字库，直接在屏幕左上角画几条竖线。

```c
void OLED_TestPattern(void)
{
    uint8_t i;

    OLED_Clear();

    OLED_SetPos(0, 0);

    for (i = 0; i < 16; i++)
    {
        OLED_WriteData(0xFF);
        OLED_WriteData(0x00);
    }

    OLED_SetPos(0, 2);

    for (i = 0; i < 32; i++)
    {
        OLED_WriteData(0x18);
    }
}
```

效果：

```text
page 0 左侧出现竖条纹
page 2 左侧出现一条横线
```

### 9.7 主函数示例

```c
int main(void)
{
    OLED_Init();

    OLED_TestPattern();

    while (1)
    {
    }
}
```

如果要切换软 I2C / 硬件 I2C，只改：

```c
#define OLED_USE_SOFT_I2C  1
```

软 I2C：

```c
#define OLED_USE_SOFT_I2C  1
```

硬件 I2C：

```c
#define OLED_USE_SOFT_I2C  0
```

## 10. 为什么 OLED 示例没有用读函数

很多 SSD1306 OLED 模块只接了写方向，实际项目里通常只向 OLED 写命令和显示数据。

所以控制 OLED 主要用：

```c
WriteCommand
WriteData
WriteDataBuf
```

读函数更常用于：

```text
MPU6050
温湿度传感器
EEPROM
RTC
电源管理芯片
```

比如读 MPU6050 的 `WHO_AM_I`：

```c
uint8_t id;

SoftI2C_ReadBytes(0xD0, 0xD1, 0x75, &id, 1);
```

或者硬件 I2C：

```c
uint8_t id;

HardI2C_ReadBytes(0xD0, 0xD1, 0x75, &id, 1);
```

如果 MPU6050 地址是 `0x68`，那么：

```text
写地址: 0xD0
读地址: 0xD1
WHO_AM_I 寄存器: 0x75
正常读到: 0x68
```

## 11. 最后总结

软 I2C 写：

```text
GPIO 手动产生 START
SendByte 地址
WaitAck
SendByte 寄存器
WaitAck
SendByte 数据
WaitAck
STOP
```

硬件 I2C 写：

```text
GenerateSTART
WaitEvent START
Send7bitAddress
WaitEvent 地址应答
SendData
WaitEvent 字节发送完成
STOP
```

软 I2C 读：

```text
先写寄存器地址
重复 START
发送读地址
ReadByte
最后一个字节 NACK
STOP
```

硬件 I2C 读：

```text
流程和软 I2C 一样
但每一步由 I2C 外设完成
代码需要等待事件并控制 ACK
```

OLED 控制：

```text
0x00 后面跟命令
0x40 后面跟显示数据
```

所以 OLED 写命令：

```c
WriteBytes(0x78, 0x00, &cmd, 1);
```

OLED 写数据：

```c
WriteBytes(0x78, 0x40, &data, 1);
```
