# DMA
## DMA Structure
![DMAStruceture](./Resources/DMA.png)
每个DMA外设有8个Stream用来控制DMA的读写，DMA的读写方向可以有：peripheral-to-memory, memory-to-peripheral and memory-to-memory

---

## DMA搬运流程
## CPU先配置DMA寄存器，包括：
```
使用哪个 stream
请求源是谁
传输方向
外设地址
内存地址
传输数量
数据宽度
地址是否自增
是否使用 FIFO
是否开启中断
```
## CPU配置DMAMUX
DMAMUX决定外设请求与DMA Stream之间的匹配，如果DMAMUX没有配置正确，即使DMA寄存器配置正确了，外设请求也进不到DMA。
## 外设发出DMA请求
```mermaid
graph LR
    外设 --> DMAMUX
    DMAMUX --> dma_strx
    dma_strx --> Stream
```
## Arbiter仲裁
如果多个Stream同时请求，仲裁器决定先服务谁，优先级由：软件优先级和硬件固定有限级共同决定。软件优先级更重要。
### 软件优先级
软件优先级是DMA配置里面主动设置的优先级，对应DMA_SxCR寄存器中的PL[1:0]，有四个优先级：
```text
DMA_PRIORITY_LOW
DMA_PRIORITY_MEDIUM
DMA_PRIORITY_HIGH
DMA_PRIORITY_VERY_HIGH
```
### 硬件固定优先级
当两个stream的软件优先级相同时，编号小的stream优先级高

## DMA通过AHB master port搬运数据
### 外设到内存
```text
 外设地址 -> DMA peripheral port -> FIFO -> DMA memory port -> SRAM Buffer
```
### 内存到外设
```text
 SRAM Buffer -> DMA memory port -> FIFO -> DMA peripheral -> 外设地址
```
### 内存到内存
```text
SRAM -> DMA memory/peripheral bus -> SRAM
``` 
## 更新地址和计数器
每传输一个数据项，DMA会根据配置自动更新：
```text
NDTR：剩余传输数量减少
M0AR：如果MINC=1，内存地址自增
PAR：如果PINC=1，外设地址自增
```
## 传输完成或出错后产生标志/中断
当NDTR等于0，或者发生错误时，DMA设置状态标志，常见标志有：
```text
TCIF：传输完成
HTIF：传输一半
TEIF：传输错误
FEIF：FIFO上溢、下溢或FIFO level错误
DMEIF：Direct mode错误
```
如果开启了NVIC中断，这些事件会送到NVIC

---
## 数据项
&emsp;&emsp;数据项是DMA传输时每次计数的一个传输单位，在寄存器中由DMA_SxNDTR表示，范围是0-65535，在DMA使能后，它会显示剩余要传输的数据项数量，并且每次DMA传输后递减。
&emsp;&emsp;数据项的大小由DMA配置的数据宽度决定，主要看：
```text
PSIZE：外设端数据宽度
MSIZE：内存端数据宽度
```
常见的数据宽度有：
```text
DMA_PDATAALIGN_BYTE/DMA_MDATAALIGN_BYTE 外设/内存1字节对齐
DMA_PDATAALIGN_HALFWORD/DMA_MDATAALIGN_HALFWORD 外设/内存2字节半字对齐
DMA_PDATAALIGN_WORD/DMA_MDATAALIGN_WORD 外设/内存4字节全字对齐
```
当 PSIZE 和 MSIZE 不一样时，FIFO 的核心作用就是做“数据宽度适配”，也就是手册里说的 packing / unpacking，打包 / 拆包。手册明确说明：
```text
只有使用内部 FIFO 时，源端和目标端的数据宽度才可以通过 PSIZE 和 MSIZE 分别配置为 8/16/32 bit；如果 PSIZE 和 MSIZE 不相等，DMA 会通过 FIFO 做 packing/unpacking。
```

---
## FIFO
![FIFO](./Resources/FIFO.png)
# STM32H743 DMA FIFO 重点笔记
FIFO 全称：
```text
First In, First Out
```
意思是：
```text
先进先出队列
```
在 DMA 中，FIFO 是每个 DMA stream 内部的临时缓存区。
可以理解为：
```text
源端数据
   ↓
DMA FIFO
   ↓
目标端数据
```
FIFO 不是用来长期存储数据的，而是 DMA 搬运过程中的临时中转站。

---

## DMA FIFO 的大小

STM32H743中每个DMA FIFO 大小为：
```text
4 words
```
其中：
```text
1 word = 32 bit = 4 bytes
```
所以 FIFO 总容量为：

```text
4 words = 16 bytes
```

可以理解为：

```text
FIFO = 4 个 32-bit word
```

每个 word 又可以拆成 4 个 byte lane：

```text
byte lane 0
byte lane 1
byte lane 2
byte lane 3
```

---

## FIFO 的水位

FIFO 有不同的填充程度：

```text
Empty
1/4 full
1/2 full
3/4 full
Full
```

因为 FIFO 总共是 4 words，所以：

```text
Empty    = 0 word
1/4 full = 1 word
1/2 full = 2 words
3/4 full = 3 words
Full     = 4 words
```

在 FIFO mode 下，可以配置 FIFO threshold：

```text
1/4 full
1/2 full
3/4 full
full
```

---

## FIFO mode 和 direct mode

DMA 有两种相关工作方式：

```text
FIFO mode
Direct mode
```
---

### FIFO mode

FIFO mode 下，数据会先进入 FIFO，再从 FIFO 输出到目标端。

路径：

```text
源端 → FIFO → 目标端
```

FIFO mode 支持：

```text
PSIZE != MSIZE
```

也就是源端和目标端数据宽度不同。

例如：

```text
byte      → word
byte      → half-word
half-word → word
word      → byte
```

---

### Direct mode

Direct mode 下，DMA 尽量直接搬运数据，不使用 FIFO threshold 机制。

特点：

```text
结构简单
延迟较低
适合 PSIZE = MSIZE 的普通 DMA 传输
```

但是要注意：

```text
Direct mode 不支持 packing / unpacking
```

因此：

```text
Direct mode 下不应配置 PSIZE != MSIZE
```

---

## PSIZE 和 MSIZE 是什么？

DMA 中有两个重要的数据宽度配置：

```text
PSIZE = Peripheral data size
MSIZE = Memory data size
```

常见取值：

```text
byte      = 8 bit  = 1 byte
half-word = 16 bit = 2 bytes
word      = 32 bit = 4 bytes
```

在 HAL 中对应：

```c
hdma.Init.PeriphDataAlignment
hdma.Init.MemDataAlignment
```

例如：

```c
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
hdma.Init.MemDataAlignment    = DMA_MDATAALIGN_BYTE;
```

---

##  FIFO 的核心作用

FIFO 的主要作用有：

```text
1. 临时缓存数据
2. 缓冲源端和目标端速度差
3. 在 PSIZE != MSIZE 时进行宽度适配
4. 支持 packing / unpacking
5. 提高总线访问效率
```

最重要的是：

```text
当 PSIZE 和 MSIZE 不一样时，FIFO 负责数据打包或拆包。
```

---

## Packing 是什么？

Packing 表示：

```text
把多个小宽度数据组合成一个大宽度数据
```

也就是：

```text
小 → 大
```

常见情况：

```text
byte      → half-word
byte      → word
half-word → word
```

---

### byte → word

例如：

```text
Source      = byte
Destination = word
```

过程：

```text
B0 + B1 + B2 + B3 → W0
B4 + B5 + B6 + B7 → W1
B8 + B9 + B10 + B11 → W2
B12 + B13 + B14 + B15 → W3
```

也就是说：

```text
4 个 byte 打包成 1 个 word
```

---

### byte → half-word

例如：

```text
Source      = byte
Destination = half-word
```

过程：

```text
B0 + B1 → H0
B2 + B3 → H1
B4 + B5 → H2
B6 + B7 → H3
```

也就是说：

```text
2 个 byte 打包成 1 个 half-word
```

---

### half-word → word

例如：

```text
Source      = half-word
Destination = word
```

过程：

```text
H0 + H1 → W0
H2 + H3 → W1
H4 + H5 → W2
H6 + H7 → W3
```

也就是说：

```text
2 个 half-word 打包成 1 个 word
```

---

## Unpacking 是什么？

Unpacking 表示：

```text
把一个大宽度数据拆成多个小宽度数据
```

也就是：

```text
大 → 小
```

常见情况：

```text
word      → half-word
word      → byte
half-word → byte
```

---

### half-word → byte

例如：

```text
Source      = half-word
Destination = byte
```

过程：

```text
H0 → B0 + B1
H1 → B2 + B3
H2 → B4 + B5
H3 → B6 + B7
```

也就是说：

```text
1 个 half-word 拆成 2 个 byte
```

---

### word → byte

例如：

```text
Source      = word
Destination = byte
```

过程：

```text
W0 → B0 + B1 + B2 + B3
```

也就是说：

```text
1 个 word 拆成 4 个 byte
```

---

## FIFO 和 NDTR 的关系

`DMA_SxNDTR` 表示要传输的数据项数量。

注意：

```text
NDTR 不是字节数，而是数据项数量。
```

当 `PSIZE` 和 `MSIZE` 相同时，比较好理解：

```text
PSIZE = MSIZE = byte      → 1 个数据项 = 1 byte
PSIZE = MSIZE = half-word → 1 个数据项 = 2 bytes
PSIZE = MSIZE = word      → 1 个数据项 = 4 bytes
```

当 `PSIZE != MSIZE` 时，要重点记住：

```text
NDTR 的数据项宽度按 PSIZE 理解
```

例如：

```text
PSIZE = byte
MSIZE = word
NDTR = 16
```

表示：

```text
总传输量 = 16 × 1 byte = 16 bytes
```

最终 FIFO 会把它打包成：

```text
4 个 word
```

再例如：

```text
PSIZE = half-word
MSIZE = word
NDTR = 8
```

表示：

```text
总传输量 = 8 × 2 bytes = 16 bytes
```

最终 FIFO 会把它打包成：

```text
4 个 word
```

---

## FIFO 在不同传输方向中的作用

### Peripheral-to-memory

例如：

```text
UART RX
SPI RX
ADC DMA
```

路径：

```text
外设 → FIFO → 内存
```

如果外设端是 byte，内存端是 word：

```text
byte → word
```

FIFO 会先收集多个 byte，再打包成 word 写入内存。

---

### Memory-to-peripheral

例如：

```text
UART TX
SPI TX
```

路径：

```text
内存 → FIFO → 外设
```

如果内存端是 word，外设端是 byte：

```text
word → byte
```

FIFO 会先从内存读取 word，再拆成多个 byte 送给外设。

---

### Memory-to-memory

例如：

```text
SRAM1 → AXI SRAM
```

路径：

```text
源内存 → FIFO → 目标内存
```

FIFO 可以提高连续搬运效率，也可以配合 burst transfer 使用。

---

## FIFO 相关错误

常见 FIFO 错误包括：

```text
FIFO underrun
FIFO overrun
FIFO threshold 和 burst size 不兼容
```

对应 DMA 标志：

```text
FEIFx = FIFO Error Interrupt Flag
```

如果出现 FIFO 错误，HAL 中通常会体现为：

```c
HAL_DMA_ERROR_FE
```
---

## FIFO总结

```text
FIFO 是 DMA stream 内部的 4-word 临时缓存。
```

```text
FIFO mode 支持 threshold：1/4、1/2、3/4、full。
```

```text
PSIZE 和 MSIZE 不同时，FIFO 负责 packing / unpacking。
```

```text
Packing：多个小宽度数据组合成一个大宽度数据。
```

```text
Unpacking：一个大宽度数据拆成多个小宽度数据。
```

```text
Direct mode 不支持 packing / unpacking。
```

```text
PSIZE != MSIZE 时，应使用 FIFO mode。
```

```text
NDTR 的数据项宽度在 PSIZE/MSIZE 不同时按 PSIZE 理解。
```

```text
FIFO 可以提高总线效率，也可以缓冲速度差，但容量只有 4 words，不能当大缓存使用。
```
---
# 突发传输 Burst Transfer
DMA获得一次总线访问的机会后，不是只搬运1个数据项Item，而是连续搬运多个数据项Items。
突发传输的大小由DMA_SxCR里面的MBUST和PBUST配置，burst size表示beats数量不是字节数
## beat
beat就是突发传输中的一次基本传输，一次beat所传输的数据大小由数据宽度决定
# DMA 配置结构体

## DMA 的核心结构体

HAL 中普通 DMA1 / DMA2 的配置主要围绕：

```c
DMA_HandleTypeDef hdma_xxx;
```

常见示例：

```c
DMA_HandleTypeDef hdma_spi2_rx;
DMA_HandleTypeDef hdma_spi2_tx;
DMA_HandleTypeDef hdma_adc1;
```

其中最重要的是：

```c
hdma_xxx.Instance
hdma_xxx.Init
```

可以理解为：

```text
Instance：选择使用哪个 DMA Stream
Init：配置这个 Stream 如何工作
```

---

## DMA_HandleTypeDef

```text
DMA_HandleTypeDef
├── Instance     ：选择 DMAx_Streamy
├── Init         ：DMA 初始化配置
├── Parent       ：绑定的外设句柄
├── State        ：HAL 内部状态
├── ErrorCode    ：HAL 错误码
└── Callback     ：完成、半完成、错误等回调函数
```

实际工程中，最常手动配置的是：

```text
Instance
Init.Request
Init.Direction
Init.PeriphInc
Init.MemInc
Init.PeriphDataAlignment
Init.MemDataAlignment
Init.Mode
Init.Priority
Init.FIFOMode
Init.FIFOThreshold
Init.MemBurst
Init.PeriphBurst
```

---

# Instance：选择 DMA Stream

## 作用

`Instance` 用来指定使用哪个 DMA 控制器和哪个 Stream。

示例：

```c
hdma_spi2_rx.Instance = DMA1_Stream0;
hdma_spi2_tx.Instance = DMA1_Stream1;
```

常见可选值：

```c
DMA1_Stream0
DMA1_Stream1
DMA1_Stream2
DMA1_Stream3
DMA1_Stream4
DMA1_Stream5
DMA1_Stream6
DMA1_Stream7

DMA2_Stream0
DMA2_Stream1
DMA2_Stream2
DMA2_Stream3
DMA2_Stream4
DMA2_Stream5
DMA2_Stream6
DMA2_Stream7
```

## 注意

```text
同一个 DMA Stream 同一时间只能服务一个 DMA 传输。
不同外设不能随意共用同一个 Stream。
```

---

# Request：选择 DMAMUX 请求源

## 作用

```c
hdma.Init.Request
```

用于告诉 DMAMUX：

```text
这个 DMA Stream 响应哪个外设的 DMA 请求。
```

例如：

```c
hdma_spi2_rx.Init.Request = DMA_REQUEST_SPI2_RX;
hdma_spi2_tx.Init.Request = DMA_REQUEST_SPI2_TX;
```

含义：

```text
SPI2_RX 请求 → 某个 DMA Stream
SPI2_TX 请求 → 某个 DMA Stream
```

## 常见值

```c
DMA_REQUEST_MEM2MEM

DMA_REQUEST_SPI1_RX
DMA_REQUEST_SPI1_TX
DMA_REQUEST_SPI2_RX
DMA_REQUEST_SPI2_TX

DMA_REQUEST_USART1_RX
DMA_REQUEST_USART1_TX
DMA_REQUEST_USART2_RX
DMA_REQUEST_USART2_TX

DMA_REQUEST_I2C1_RX
DMA_REQUEST_I2C1_TX

DMA_REQUEST_ADC1
DMA_REQUEST_ADC2

DMA_REQUEST_TIM1_CH1
DMA_REQUEST_TIM1_UP
DMA_REQUEST_TIM2_CH1
DMA_REQUEST_TIM2_UP
```

---

# Direction：传输方向

## 作用

```c
hdma.Init.Direction
```

用于设置 DMA 数据搬运方向。

## 常见值

```c
DMA_PERIPH_TO_MEMORY
DMA_MEMORY_TO_PERIPH
DMA_MEMORY_TO_MEMORY
```

| 值 | 含义 | 典型场景 |
|---|---|---|
| `DMA_PERIPH_TO_MEMORY` | 外设到内存 | UART RX、SPI RX、ADC |
| `DMA_MEMORY_TO_PERIPH` | 内存到外设 | UART TX、SPI TX、DAC |
| `DMA_MEMORY_TO_MEMORY` | 内存到内存 | RAM 拷贝 |

示例：

```c
hdma_uart_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
hdma_spi_tx.Init.Direction  = DMA_MEMORY_TO_PERIPH;
```

---

# PeriphInc：外设地址是否自增

## 作用

```c
hdma.Init.PeriphInc
```

用于设置外设端地址是否自增。

## 可选值

```c
DMA_PINC_ENABLE
DMA_PINC_DISABLE
```

| 值 | 含义 |
|---|---|
| `DMA_PINC_DISABLE` | 外设地址不变 |
| `DMA_PINC_ENABLE` | 外设地址自增 |

大多数外设 DMA 都使用：

```c
hdma.Init.PeriphInc = DMA_PINC_DISABLE;
```

因为外设数据寄存器地址通常固定，例如：

```text
SPIx->TXDR
SPIx->RXDR
USARTx->TDR
USARTx->RDR
ADCx->DR
```

---

# MemInc：内存地址是否自增

## 作用

```c
hdma.Init.MemInc
```

用于设置内存端地址是否自增。

## 可选值

```c
DMA_MINC_ENABLE
DMA_MINC_DISABLE
```

| 值 | 含义 |
|---|---|
| `DMA_MINC_ENABLE` | 内存地址自增 |
| `DMA_MINC_DISABLE` | 内存地址不变 |

大多数 buffer 传输使用：

```c
hdma.Init.MemInc = DMA_MINC_ENABLE;
```

例如 UART 接收：

```text
USART_RDR → rx_buf[0]
USART_RDR → rx_buf[1]
USART_RDR → rx_buf[2]
```

---

# PeriphDataAlignment：外设端数据宽度

## 作用

```c
hdma.Init.PeriphDataAlignment
```

对应参考手册中的：

```text
PSIZE
```

也就是外设端数据项宽度。

## 可选值

```c
DMA_PDATAALIGN_BYTE
DMA_PDATAALIGN_HALFWORD
DMA_PDATAALIGN_WORD
```

| 值 | 外设端宽度 |
|---|---|
| `DMA_PDATAALIGN_BYTE` | 8 bit，1 字节 |
| `DMA_PDATAALIGN_HALFWORD` | 16 bit，2 字节 |
| `DMA_PDATAALIGN_WORD` | 32 bit，4 字节 |

示例：

```c
// UART / SPI 8-bit
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;

// ADC 16-bit
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;

// 32-bit 外设数据
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
```

---

# MemDataAlignment：内存端数据宽度

## 作用

```c
hdma.Init.MemDataAlignment
```

对应参考手册中的：

```text
MSIZE
```

也就是内存端数据项宽度。

## 可选值

```c
DMA_MDATAALIGN_BYTE
DMA_MDATAALIGN_HALFWORD
DMA_MDATAALIGN_WORD
```

| 值 | 内存端宽度 |
|---|---|
| `DMA_MDATAALIGN_BYTE` | 8 bit，1 字节 |
| `DMA_MDATAALIGN_HALFWORD` | 16 bit，2 字节 |
| `DMA_MDATAALIGN_WORD` | 32 bit，4 字节 |

示例：

```c
// uint8_t buffer[]
hdma.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;

// uint16_t buffer[]
hdma.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;

// uint32_t buffer[]
hdma.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
```

---

# PSIZE 和 MSIZE 常见搭配

## UART / SPI 8-bit

```c
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
hdma.Init.MemDataAlignment    = DMA_MDATAALIGN_BYTE;
```

含义：

```text
外设端 1 字节
内存端 1 字节
```

---

## ADC 12-bit / 16-bit

```c
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;
hdma.Init.MemDataAlignment    = DMA_MDATAALIGN_HALFWORD;
```

含义：

```text
外设端 2 字节
内存端 2 字节
```

---

## 32-bit 数据

```c
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
hdma.Init.MemDataAlignment    = DMA_MDATAALIGN_WORD;
```

含义：

```text
外设端 4 字节
内存端 4 字节
```

---

## PSIZE 和 MSIZE 不同时

例如：

```c
hdma.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
hdma.Init.MemDataAlignment    = DMA_MDATAALIGN_WORD;
```

含义：

```text
外设端 1 字节
内存端 4 字节
```

此时：

```text
PSIZE != MSIZE
需要 FIFO mode 做 packing / unpacking
```

---

# Mode：DMA 工作模式

## 作用

```c
hdma.Init.Mode
```

用于设置 DMA 的工作模式。

## 常见值

```c
DMA_NORMAL
DMA_CIRCULAR
DMA_PFCTRL
DMA_DOUBLE_BUFFER_M0
DMA_DOUBLE_BUFFER_M1
```

| 值 | 含义 | 典型场景 |
|---|---|---|
| `DMA_NORMAL` | 普通模式，传完停止 | SPI 一次发送、UART 一次接收 |
| `DMA_CIRCULAR` | 循环模式，传完自动从头开始 | ADC 连续采样、UART 环形接收 |
| `DMA_PFCTRL` | 外设流控 | 由外设控制传输结束 |
| `DMA_DOUBLE_BUFFER_M0` | 双缓冲，先用 M0 | 音频、连续采集 |
| `DMA_DOUBLE_BUFFER_M1` | 双缓冲，先用 M1 | 音频、连续采集 |

最常用：

```c
hdma.Init.Mode = DMA_NORMAL;
```

连续采样常用：

```c
hdma.Init.Mode = DMA_CIRCULAR;
```

---

# Priority：DMA 软件优先级

## 作用

```c
hdma.Init.Priority
```

用于设置 DMA Stream 的软件优先级。

当同一个 DMA 控制器中多个 Stream 同时请求传输时，仲裁器会优先服务优先级高的 Stream。

## 可选值

```c
DMA_PRIORITY_LOW
DMA_PRIORITY_MEDIUM
DMA_PRIORITY_HIGH
DMA_PRIORITY_VERY_HIGH
```

| 值 | 优先级 |
|---|---|
| `DMA_PRIORITY_LOW` | 低 |
| `DMA_PRIORITY_MEDIUM` | 中 |
| `DMA_PRIORITY_HIGH` | 高 |
| `DMA_PRIORITY_VERY_HIGH` | 很高 |

示例：

```c
hdma_adc.Init.Priority = DMA_PRIORITY_HIGH;
hdma_uart_rx.Init.Priority = DMA_PRIORITY_MEDIUM;
```

注意：

```text
DMA Priority 不是 NVIC 中断优先级。
DMA Priority 决定 DMA Stream 使用总线资源的优先顺序。
NVIC Priority 决定 CPU 响应中断的优先顺序。
```

---

# FIFOMode：FIFO 模式

## 作用

```c
hdma.Init.FIFOMode
```

用于选择 DMA 是否启用 FIFO mode。

## 可选值

```c
DMA_FIFOMODE_DISABLE
DMA_FIFOMODE_ENABLE
```

| 值 | 含义 |
|---|---|
| `DMA_FIFOMODE_DISABLE` | Direct mode |
| `DMA_FIFOMODE_ENABLE` | FIFO mode |

普通外设 DMA 常用：

```c
hdma.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
```

当 `PSIZE != MSIZE` 时，应该使用：

```c
hdma.Init.FIFOMode = DMA_FIFOMODE_ENABLE;
```

---

# FIFOThreshold：FIFO 阈值

## 作用

```c
hdma.Init.FIFOThreshold
```

用于设置 FIFO 水位阈值。

只有在启用 FIFO mode 时才有意义：

```c
hdma.Init.FIFOMode = DMA_FIFOMODE_ENABLE;
```

## 可选值

```c
DMA_FIFO_THRESHOLD_1QUARTERFULL
DMA_FIFO_THRESHOLD_HALFFULL
DMA_FIFO_THRESHOLD_3QUARTERSFULL
DMA_FIFO_THRESHOLD_FULL
```

| 值 | 含义 |
|---|---|
| `DMA_FIFO_THRESHOLD_1QUARTERFULL` | 1/4 满 |
| `DMA_FIFO_THRESHOLD_HALFFULL` | 1/2 满 |
| `DMA_FIFO_THRESHOLD_3QUARTERSFULL` | 3/4 满 |
| `DMA_FIFO_THRESHOLD_FULL` | 全满 |

常用：

```c
hdma.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_FULL;
```

---

# MemBurst：内存端突发传输

## 作用

```c
hdma.Init.MemBurst
```

设置内存端 burst 传输方式。

## 可选值

```c
DMA_MBURST_SINGLE
DMA_MBURST_INC4
DMA_MBURST_INC8
DMA_MBURST_INC16
```

| 值 | 含义 |
|---|---|
| `DMA_MBURST_SINGLE` | 单次传输 |
| `DMA_MBURST_INC4` | 4-beat burst |
| `DMA_MBURST_INC8` | 8-beat burst |
| `DMA_MBURST_INC16` | 16-beat burst |

普通外设 DMA 初学建议：

```c
hdma.Init.MemBurst = DMA_MBURST_SINGLE;
```

---

# PeriphBurst：外设端突发传输

## 作用

```c
hdma.Init.PeriphBurst
```

设置外设端 burst 传输方式。

## 可选值

```c
DMA_PBURST_SINGLE
DMA_PBURST_INC4
DMA_PBURST_INC8
DMA_PBURST_INC16
```

| 值 | 含义 |
|---|---|
| `DMA_PBURST_SINGLE` | 单次传输 |
| `DMA_PBURST_INC4` | 4-beat burst |
| `DMA_PBURST_INC8` | 8-beat burst |
| `DMA_PBURST_INC16` | 16-beat burst |

普通外设 DMA 初学建议：

```c
hdma.Init.PeriphBurst = DMA_PBURST_SINGLE;
```

如果外设地址不自增，通常不要随便开启 peripheral burst。

---

# 字段和值总表

| 字段 | 作用 | 常用值 |
|---|---|---|
| `Instance` | 选择 DMA Stream | `DMA1_Stream0` 等 |
| `Request` | 选择 DMAMUX 请求源 | `DMA_REQUEST_SPI2_RX` 等 |
| `Direction` | 传输方向 | `DMA_PERIPH_TO_MEMORY` / `DMA_MEMORY_TO_PERIPH` / `DMA_MEMORY_TO_MEMORY` |
| `PeriphInc` | 外设地址自增 | 多数外设用 `DMA_PINC_DISABLE` |
| `MemInc` | 内存地址自增 | buffer 一般用 `DMA_MINC_ENABLE` |
| `PeriphDataAlignment` | 外设端数据宽度 | `BYTE` / `HALFWORD` / `WORD` |
| `MemDataAlignment` | 内存端数据宽度 | `BYTE` / `HALFWORD` / `WORD` |
| `Mode` | 工作模式 | `DMA_NORMAL` / `DMA_CIRCULAR` |
| `Priority` | DMA 软件优先级 | `LOW` / `MEDIUM` / `HIGH` / `VERY_HIGH` |
| `FIFOMode` | FIFO 或 Direct mode | 常用 `DMA_FIFOMODE_DISABLE` |
| `FIFOThreshold` | FIFO 水位 | 开 FIFO 时使用 |
| `MemBurst` | 内存端 burst | 常用 `DMA_MBURST_SINGLE` |
| `PeriphBurst` | 外设端 burst | 常用 `DMA_PBURST_SINGLE` |

---
