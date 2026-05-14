# STM32 串口 USART 小白教程

## 学习目标

学完这份笔记，你应该能理解：

- 串口通信是干什么的。
- `TX`、`RX`、`GND` 怎么接。
- TTL 串口、USB TO TTL 模块分别是什么。
- STM32 里的 USART 是什么外设。
- 串口初始化代码每一项在配置什么。
- 发送、接收、中断、标志位之间是什么关系。
- 51 单片机串口和 STM32 串口的相同点与区别。

## 教学清单

1. 串口到底是干什么的
2. TX、RX、GND 怎么接
3. TTL 电平和 USB TO TTL 是什么
4. UART 和 USART 是什么关系
5. 串口一帧数据长什么样
6. STM32 里的 USART 是什么
7. USART 初始化流程
8. STM32 串口发送一个字节的流程
9. STM32 串口发送字符串的流程
10. STM32 串口接收数据的流程
11. 轮询、中断、DMA 接收的区别
12. 常见标志位：`TXE`、`TC`、`RXNE`、`ORE`、`IDLE`
13. 51 单片机串口和 STM32 串口的关系
14. 串口不通时怎么排查

## 1. 串口到底是干什么的

串口就是让两个设备之间传数据。

例如：

- STM32 给电脑发送调试信息。
- 电脑给 STM32 发送控制命令。
- STM32 和蓝牙模块通信。
- STM32 和 WiFi、GPS、4G 模块通信。
- 两个单片机之间互相通信。

常见连接关系：

```text
STM32 USART <-> USB TO TTL <-> 电脑串口助手
```

电脑本身不能直接识别 STM32 的 TTL 串口信号，所以通常需要一个 USB TO TTL 模块作为转换器。

## 2. TX、RX、GND 怎么接

串口最少需要三根线：

| 引脚 | 作用 |
|---|---|
| `TX` | 发送数据 |
| `RX` | 接收数据 |
| `GND` | 共地 |

连接规则是交叉连接：

```text
STM32 TX  -> USB TO TTL RX
STM32 RX  -> USB TO TTL TX
STM32 GND -> USB TO TTL GND
```

注意：

- `TX` 要接对方的 `RX`。
- `RX` 要接对方的 `TX`。
- 两边必须共地。
- 如果 TX 接 TX、RX 接 RX，通常收不到数据。

## 3. TTL 电平和 USB TO TTL

STM32 串口输出的是 TTL 电平。

常见 TTL 电平：

```text
高电平：3.3V 或 5V
低电平：0V
```

STM32 大多数 IO 是 3.3V 电平，连接外部模块时要注意电平是否匹配。

USB TO TTL 模块的作用：

```text
电脑 USB 信号 <-> TTL 串口信号
```

它让电脑可以通过串口助手和 STM32 通信。

不要把 TTL 串口和传统 RS232 串口直接混接，因为它们的电平标准不同。

## 4. UART 和 USART 是什么关系

UART 是异步串口。

USART 是通用同步/异步收发器，比 UART 多了同步通信能力。

但是学习 STM32 时，大多数情况下：

```text
USART 当 UART 用
```

也就是只使用：

```text
TX
RX
GND
```

不使用额外的同步时钟线。

## 5. 串口一帧数据长什么样

串口不是一次性把一个字节扔过去，而是按位发送。

最常见的数据帧格式是：

```text
115200, 8N1
```

含义：

| 参数 | 含义 |
|---|---|
| `115200` | 波特率，每秒传输 115200 bit |
| `8` | 8 个数据位 |
| `N` | No parity，无校验 |
| `1` | 1 个停止位 |

一帧数据通常由这些部分组成：

```text
起始位 + 数据位 + 校验位 + 停止位
```

常见 `8N1` 实际是：

```text
1 位起始位 + 8 位数据位 + 1 位停止位
```

串口空闲时一般是高电平。开始发送时，先拉低产生起始位，然后发送数据位，最后输出停止位回到高电平。

## 6. STM32 里的 USART 是什么

USART 是 STM32 芯片内部集成的片上外设。

可以这样理解：

```text
CPU：执行你的 C 代码
USART：负责串口收发
GPIO：把 USART 信号连接到芯片引脚
```

发送时：

```text
CPU 把数据写入 USART 数据寄存器
USART 硬件自动按波特率从 TX 引脚发出去
```

接收时：

```text
外部数据进入 RX 引脚
USART 硬件自动解析数据
收到后置位 RXNE 标志
CPU 再读取数据
```

所以使用硬件 USART 时，不是 CPU 自己一位一位模拟电平，而是 USART 外设自动完成串口时序。

## 7. USART 初始化流程

STM32 使用 USART，一般按下面步骤：

1. 打开 GPIO 时钟。
2. 打开 USART 时钟。
3. 配置 `TX` 引脚为复用推挽输出。
4. 配置 `RX` 引脚为输入模式，或复用输入模式。
5. 配置 USART 参数：波特率、数据位、停止位、校验位。
6. 如果使用中断，配置 NVIC。
7. 使能 USART。

标准外设库常见初始化代码：

```c
USART_InitStructure.USART_BaudRate = 115200;
USART_InitStructure.USART_WordLength = USART_WordLength_8b;
USART_InitStructure.USART_StopBits = USART_StopBits_1;
USART_InitStructure.USART_Parity = USART_Parity_No;
USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;
USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
USART_Init(USART1, &USART_InitStructure);
USART_Cmd(USART1, ENABLE);
```

对应含义：

| 代码 | 作用 |
|---|---|
| `USART_BaudRate = 115200` | 设置波特率 |
| `USART_WordLength_8b` | 设置 8 位数据位 |
| `USART_StopBits_1` | 设置 1 位停止位 |
| `USART_Parity_No` | 不使用校验 |
| `USART_Mode_Rx \| USART_Mode_Tx` | 同时开启接收和发送 |
| `USART_HardwareFlowControl_None` | 不使用硬件流控 |
| `USART_Init(...)` | 把配置写入 USART 外设 |
| `USART_Cmd(..., ENABLE)` | 使能 USART，让它真正工作 |

## 8. 发送一个字节的流程

例如：

```c
USART_SendData(USART1, 'A');
```

本质上是：

```text
把字符 'A' 写入 USART1 的数据寄存器
```

然后 USART 硬件会自动从 `TX` 引脚发出去。

发送时常用 `TXE` 标志：

```text
TXE = 1：发送数据寄存器空，可以写入新数据
```

常见发送流程：

```text
等待 TXE = 1
写入一个字节
USART 硬件发送
继续等待 TXE
发送下一个字节
```

## 9. 发送字符串的流程

字符串发送就是循环发送多个字节。

例如：

```c
My_USART_SendString(USART1, "Hello\r\n");
```

实际发送顺序：

```text
H
e
l
l
o
\r
\n
```

常见函数逻辑：

```c
void My_USART_SendString(USART_TypeDef *USARTx, char *String)
{
    while (*String != '\0')
    {
        USART_SendData(USARTx, *String);
        while (USART_GetFlagStatus(USARTx, USART_FLAG_TXE) == RESET);
        String++;
    }
}
```

重点：

- `'\0'` 是 C 语言字符串结束标志。
- 函数每次发送一个字符。
- 发送完成一个字符后，指针移动到下一个字符。
- 直到遇到 `'\0'` 停止。

## 10. 接收数据的流程

当电脑发送一个字符给 STM32：

```text
电脑串口助手发送 'A'
USB TO TTL 转换成 TTL 串口信号
信号进入 STM32 RX 引脚
USART 硬件接收完成
RXNE = 1
CPU 读取 USART 数据寄存器
```

`RXNE` 的意思是：

```text
Receive Data Register Not Empty
接收数据寄存器非空
```

也就是：

```text
收到数据了，可以读取
```

轮询接收示例：

```c
if (USART_GetFlagStatus(USART1, USART_FLAG_RXNE) == SET)
{
    uint8_t data = USART_ReceiveData(USART1);
}
```

## 11. 轮询、中断、DMA 接收的区别

| 方式 | 特点 | 适合场景 |
|---|---|---|
| 轮询 | CPU 一直检查标志位，简单但浪费 CPU | 初学、简单测试 |
| 中断 | 收到数据后自动进入中断函数 | 命令接收、普通项目 |
| DMA | 数据自动搬到内存，CPU 负担小 | 大量数据、高速数据 |

建议学习顺序：

1. 先学轮询发送。
2. 再学轮询接收。
3. 再学接收中断。
4. 最后学 DMA + 空闲中断。

## 12. 常见 USART 标志位

| 标志位 | 含义 |
|---|---|
| `TXE` | 发送数据寄存器空，可以写入新数据 |
| `TC` | 发送完成，最后一位也发完了 |
| `RXNE` | 接收数据寄存器非空，收到数据了 |
| `ORE` | 接收溢出，数据没及时读导致丢失 |
| `IDLE` | 总线空闲，常用于判断一包不定长数据结束 |

初学时先记：

```text
发送看 TXE
接收看 RXNE
发送彻底完成看 TC
不定长接收常用 IDLE
```

## 13. 51 单片机串口和 STM32 串口的关系

51 单片机也有硬件串口，不一定都是软件模拟电平。

51 常见串口相关内容：

| 51 单片机 | 作用 |
|---|---|
| `SBUF` | 串口数据缓冲寄存器 |
| `SCON` | 串口控制寄存器 |
| `PCON` | 电源控制寄存器，里面有波特率倍速位 `SMOD` |
| `TI` | 发送完成标志 |
| `RI` | 接收完成标志 |

STM32 中类似关系：

| 51 | STM32 | 作用 |
|---|---|---|
| `SBUF` | `DR` / `TDR` / `RDR` | 数据寄存器 |
| `TI` | `TXE` / `TC` | 发送相关标志 |
| `RI` | `RXNE` | 接收完成标志 |
| `SCON` | `CR1` / `CR2` / `CR3` | 控制寄存器 |
| 定时器产生波特率 | `BRR` | 波特率配置 |

本质流程相同：

```text
CPU 配置串口外设
CPU 写数据寄存器
硬件自动产生 TX 电平
硬件收到 RX 数据
CPU 读取数据寄存器
```

只有软件串口才是用普通 GPIO 加延时去模拟高低电平。

## 14. 串口不通时怎么排查

优先检查：

1. TX/RX 是否交叉连接。
2. GND 是否共地。
3. 波特率是否一致。
4. 数据位、停止位、校验位是否一致。
5. STM32 是否打开 GPIO 时钟。
6. STM32 是否打开 USART 时钟。
7. TX 引脚是否配置成复用推挽输出。
8. RX 引脚是否配置正确。
9. 是否调用了 `USART_Cmd(USARTx, ENABLE)`。
10. 串口助手选择的 COM 口是否正确。
11. USB TO TTL 模块电平是否匹配。
12. 如果用中断，NVIC 和 USART 中断是否都开启。

## 核心总结

STM32 串口通信可以这样记：

```text
USART 是 STM32 内部的串口硬件。
GPIO 负责把 USART 连接到 TX/RX 引脚。
USB TO TTL 负责让 STM32 和电脑通信。
CPU 只负责配置 USART、写数据、读数据。
真正的串口电平时序由 USART 硬件自动完成。
```

最重要的主线：

```text
初始化 USART
配置 TX/RX 引脚
设置波特率和数据格式
使能 USART
发送时写数据寄存器
接收时看 RXNE 并读数据寄存器
```

