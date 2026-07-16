---
title: "UART DMA 高效收发：环形缓冲区零拷贝实战，STM32 与 GD32 双芯片验证"
date: 2026-07-05
description: "基于 lwrb 环形缓冲区库，详解 UART DMA 循环接收 + 零拷贝发送的完整架构，以立创·天空星STM32F407VGT6 (HAL) 和立创·梁山派GD32F470ZGT6 (标准外设库) 双平台实现验证，三重中断触发、DMA 位置反推、链式 TX 发送逐一剖析"
image: UART DMA 零拷贝实战.png
categories:
  - "嵌入式"
  - "驱动开发"
tags:
  - "STM32"
  - "GD32"
  - "UART"
  - "DMA"
  - "环形缓冲区"
  - "lwrb"
  - "零拷贝"
math: true
---

## 前言

在嵌入式开发中，UART 串口通信是最基础也最常用的外设之一。然而，许多开发者仍然在使用中断逐字节收发或轮询方式，导致 CPU 占用高、数据丢失风险大。**DMA + 环形缓冲区**的组合，才是高效 UART 通信的正确打开方式。

本文基于 [lwrb](https://github.com/MaJerle/lwrb) 轻量级环形缓冲区库和 [stm32-usart-uart-dma-rx-tx](https://github.com/MaJerle/stm32-usart-uart-dma-rx-tx) 参考实现，分别用 **立创·天空星STM32F407VGT6 (HAL 库)** 和 **立创·梁山派GD32F470ZGT6 (标准外设库)** 完成双芯片验证，完整工程见 [STM32F407VGT6 UART DMA](https://github.com/Lingjia007/STM32F407VGT6_UART_DMA_Efficient_RingBuffer) 与 [GD32F470ZGT6 UART DMA](https://github.com/Lingjia007/GD32F470ZGT6_UART_DMA_Efficient_RingBuffer)，以有问有答的方式深入剖析每一个设计决策。

---

## 一、为什么不用中断逐字节收发？

### Q：中断收发有什么问题？

中断方式每收发一个字节就触发一次中断，在高波特率下问题严重：

| 波特率  | 每秒字节数 | 中断频率  | 9600bps 时 CPU 开销 |
| ------- | ---------- | --------- | ------------------- |
| 9600    | 960        | ~960 Hz   | 可忽略              |
| 115200  | 11520      | ~11.5 kHz | ~5-10%              |
| 921600  | 92160      | ~92 kHz   | ~30-50%             |
| 3000000 | 300000     | ~300 kHz  | 几乎无法工作        |

每个中断需要保存/恢复上下文（至少 12 个寄存器入栈），CPU 大量时间消耗在中断切换上。更危险的是：**中断处理稍慢就会溢出 ORE 标志，导致数据丢失**。

### Q：DMA 能解决什么？

DMA（Direct Memory Access）让硬件自动在 USART 数据寄存器和内存之间搬运数据，CPU 完全不参与逐字节的传输。对于 115200bps，DMA 方式 CPU 开销接近 0%。

---

## 二、整体架构：RX 循环 + TX 零拷贝

### Q：RX 和 TX 为什么用不同的 DMA 模式？

| 通路 | DMA 模式             | 原因                                                                          |
| ---- | -------------------- | ----------------------------------------------------------------------------- |
| RX   | **Circular（循环）** | 接收是被动行为，不知道何时有数据、数据多长。循环模式让 DMA 自动回绕，永不停止 |
| TX   | **Normal（单次）**   | 发送是主动行为，每次从环形缓冲区取一段线性数据发送。完成后链式启动下一段      |

### Q：整体数据流是怎样的？

```
RX 通路：
[外部数据] → USART_DR → DMA(Circular) → usart_rx_dma_buffer[64]
                                              ↓
                              IDLE / HT / TC 三重中断触发
                                              ↓
                                     usart_rx_check()
                                     (DMA 计数器反推写入位置)
                                              ↓
                                     usart_process_data()
                                              ↓
                                     lwrb_write() → TX 环形缓冲区

TX 通路：
应用层 lwrb_write() → TX 环形缓冲区
                              ↓
                   usart_start_tx_dma_transfer()
                   (零拷贝: DMA 直接从环形缓冲区线性块读取)
                              ↓
                   DMA(Normal) → USART_DR → [外部]
                              ↓
                   TX 完成中断 → lwrb_skip() + 链式启动下一段
```

---

## 三、RX 通路：DMA Circular + 三重中断

### Q：为什么 RX DMA 用 Circular 模式？

Normal 模式下，DMA 传输完成后自动停止，必须在中断中手动重启。重启期间的数据就会丢失。Circular 模式下，DMA 写到缓冲区末尾后自动回到头部继续写，**永不停止**，不存在重启窗口期。

### Q：三重中断分别是什么？为什么需要三个？

| 中断                       | 触发时机              | 作用                                                                  |
| -------------------------- | --------------------- | --------------------------------------------------------------------- |
| **HT (Half Transfer)**     | DMA 写到缓冲区一半时  | 及时处理前半段数据，防止被后半段覆盖                                  |
| **TC (Transfer Complete)** | DMA 写完整个缓冲区时  | 及时处理后半段数据，防止被新的前半段覆盖                              |
| **IDLE Line**              | UART 检测到总线空闲时 | 处理不定长数据的尾部——最后一个包可能不足半缓冲区，HT 和 TC 都不会触发 |

**关键**：如果只有 HT + TC，当数据量不足半缓冲区时，将永远不会触发中断，数据会"卡"在 DMA 缓冲区中。IDLE 中断解决了这个问题——总线空闲意味着一帧数据结束。

### Q：DMA 计数器怎么反推写入位置？

DMA 硬件维护一个递减计数器 `CNDTR`（GD32 中为 `CNT`），初始值为缓冲区大小，每次传输后减 1。当前写入位置：

$$\text{pos} = N - \text{CNDTR}$$

其中 $N$ 为缓冲区大小。例如 `N=64`，`CNDTR=52` 时，`pos=12`，表示 DMA 已写入前 12 个字节。

**STM32 HAL 实现：**

```c
size_t pos = ARRAY_LEN(usart_rx_dma_buffer) - __HAL_DMA_GET_COUNTER(&hdma_usart1_rx);
```

**GD32 标准库实现：**

```c
size_t pos = USART0_RX_DMA_LENGTH - dma_transfer_number_get(USART0_DMA, USART0_RX_DMA_CH);
```

### Q：回绕时怎么处理？

DMA Circular 写到缓冲区末尾会自动回到头部。`usart_rx_check()` 需要处理两种情况：

| 情况     | 条件            | 处理                                                  |
| -------- | --------------- | ----------------------------------------------------- |
| 线性写入 | `pos > old_pos` | 单次调用 `usart_process_data(old_pos, pos - old_pos)` |
| 回绕写入 | `pos < old_pos` | 分两段：尾部 `[old_pos, N)` + 头部 `[0, pos)`         |

```c
if (pos > old_pos) {
    usart_process_data(&usart_rx_dma_buffer[old_pos], pos - old_pos);
} else if (pos < old_pos) {
    usart_process_data(&usart_rx_dma_buffer[old_pos], N - old_pos);
    usart_process_data(&usart_rx_dma_buffer[0], pos);
}
old_pos = pos;
```

> [!IMPORTANT]
> `pos == old_pos` 表示没有新数据到来，无需处理。这个判断不能省略——DMA 计数器可能在两次检查之间没有变化。

### Q：STM32 HAL 的统一回调是怎么回事？

STM32 HAL 提供了 `HAL_UARTEx_ReceiveToIdle_DMA()`，它将 HT、TC、IDLE 三种中断统一到一个回调 `HAL_UARTEx_RxEventCallback()` 中：

```c
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size) {
    if (huart->Instance == USART1) {
        usart_rx_check();  // 三种中断最终都调用同一个函数
    }
}
```

GD32 方案则需要手动在三个中断处理函数中分别调用 `usart_rx_check()`。

---

## 四、TX 通路：零拷贝 DMA + 链式发送

### Q：什么是零拷贝？为什么 TX 可以零拷贝？

传统做法：先把环形缓冲区数据 `memcpy` 到一个线性缓冲区，再把线性缓冲区地址给 DMA。这有一次额外的内存拷贝。

**零拷贝**：DMA 直接从环形缓冲区的连续内存区域读取数据，无需中间缓冲区。这利用了 lwrb 的线性块接口：

```c
size_t len = lwrb_get_linear_block_read_length(&usart_tx_rb);    // 连续可读长度
void *addr = lwrb_get_linear_block_read_address(&usart_tx_rb);   // 连续可读起始地址
// DMA 直接从 addr 读取 len 字节
```

### Q：环形缓冲区的数据不是可能跨回绕边界吗？DMA 只能读连续内存怎么办？

确实，环形缓冲区的数据可能跨越缓冲区末尾回到头部，形成两段不连续的区域。lwrb 的 `get_linear_block_read_length()` 只返回**从读指针到写指针（或缓冲区末尾）的连续长度**，不会跨越回绕边界。

解决方案：**链式发送**。一次 DMA 只发送线性段，传输完成中断中再检查环形缓冲区是否还有数据，有则启动下一轮 DMA。如果环形缓冲区数据跨回绕边界，会自动拆分为两次 DMA 传输：

```
环形缓冲区: [....ABCDE.....FGH....]
                     ↑              ↑
                   读指针         写指针

DMA 第 1 次: 发送 "ABCDE" (线性段到缓冲区末尾)
DMA 第 2 次: 发送 "FGH"  (从缓冲区头部开始的新线性段)
```

### Q：TX DMA 的启动流程是什么？

```c
void usart_start_tx_dma_transfer(void) {
    __disable_irq();  // 进入临界区

    if (usart_tx_dma_current_len == 0) {  // DMA 空闲
        size_t len = lwrb_get_linear_block_read_length(&usart_tx_rb);
        if (len > 0) {
            void *addr = lwrb_get_linear_block_read_address(&usart_tx_rb);
            usart_tx_dma_current_len = len;

            // STM32 HAL:
            // HAL_DMA_Abort(&hdma_usart1_tx);
            // HAL_UART_Transmit_DMA(&huart1, addr, len);

            // GD32 标准库:
            dma_channel_disable(USART0_DMA, USART0_TX_DMA_CH);
            dma_flag_clear(USART0_DMA, USART0_TX_DMA_CH, ...);
            dma_transfer_number_config(USART0_DMA, USART0_TX_DMA_CH, len);
            dma_memory_address_config(USART0_DMA, USART0_TX_DMA_CH, (uint32_t)addr);
            dma_channel_enable(USART0_DMA, USART0_TX_DMA_CH);
        }
    }

    __enable_irq();  // 退出临界区
}
```

### Q：为什么需要 `usart_tx_dma_current_len` 这个全局变量？

它充当**互斥标志**：值为 0 表示 DMA 空闲，可以启动新传输；非 0 表示 DMA 正在传输，不能干扰。应用层写入环形缓冲区后调用 `usart_start_tx_dma_transfer()`，如果 DMA 正忙就返回——数据安全地留在环形缓冲区中，等当前传输完成的中断会自动启动下一轮。

### Q：TX 完成中断里做什么？

```c
// 1. 跳过已发送的数据（移动环形缓冲区读指针）
lwrb_skip(&usart_tx_rb, usart_tx_dma_current_len);

// 2. 标记 DMA 空闲
usart_tx_dma_current_len = 0;

// 3. 尝试链式启动下一轮
usart_start_tx_dma_transfer();
```

`lwrb_skip()` 只移动读指针，不做任何数据拷贝——这是零拷贝的另一半。

---

## 五、lwrb 环形缓冲区核心设计

### Q：lwrb 有什么特别之处？

| 特性       | 说明                                                            |
| ---------- | --------------------------------------------------------------- |
| 轻量       | 头文件 + 源文件各一个，无动态分配                               |
| 线程安全   | 读写指针用 `volatile` 修饰，单写单读场景无需锁                  |
| 零拷贝接口 | `get_linear_block_read_address/length` 直接暴露内部地址给 DMA   |
| 魔术字校验 | `magic1=0xDEADBEEF, magic2=~0xDEADBEEF`，检测缓冲区是否已初始化 |
| 实际容量   | `size - 1`，满条件 `w == r - 1`                                 |

### Q：为什么缓冲区满的条件是 `w == r - 1` 而不是 `w == r`？

`w == r` 表示"空"（读写指针重合）。如果满也用 `w == r`，就无法区分空和满。保留一个字节不写（`w` 比 `r` 少走一步），就能通过指针关系判断状态：

| 条件                  | 状态         |
| --------------------- | ------------ |
| `w == r`              | 空           |
| `(w + 1) % size == r` | 满           |
| 其他                  | 有数据但未满 |

### Q：lwrb 的线性块接口为什么对 DMA 至关重要？

DMA 只能搬运**连续**的内存区域。环形缓冲区的数据在回绕时分为两段不连续区域。线性块接口返回**从读指针到缓冲区末尾（或写指针，取较小者）**的连续区域，DMA 可以直接使用这段地址：

```c
// 返回从读指针开始的连续可读长度
size_t lwrb_get_linear_block_read_length(lwrb_t *lwrb) {
    size_t len, w;
    w = lwrb->w;  // 一次性读取写指针
    if (w >= lwrb->r) {
        len = w - lwrb->r;  // 线性区域
    } else {
        len = lwrb->size - lwrb->r;  // 读指针到缓冲区末尾
    }
    return len;
}
```

---

## 六、STM32 vs GD32：双芯片实现对比

### Q：两个平台的 DMA 架构有什么区别？

| 维度       | STM32F407                                   | GD32F470                                                               |
| ---------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| DMA 模型   | **Stream + Channel** (DMA2_Stream2_CH4)     | **Channel + Subperi** (DMA1_CH2_SUB4)                                  |
| RX DMA     | DMA2_Stream2                                | DMA1_Channel2                                                          |
| TX DMA     | DMA2_Stream7                                | DMA1_Channel7                                                          |
| DMA 计数器 | `__HAL_DMA_GET_COUNTER()` 宏                | `dma_transfer_number_get()` 函数                                       |
| 中断管理   | HAL 统一回调 (`HAL_UARTEx_RxEventCallback`) | 手动编写 IRQ Handler                                                   |
| 抽象层级   | 厚封装（HAL）                               | 薄封装（标准外设库）                                                   |
| GPIO 复用  | `HAL_GPIO_Init()` 一步完成                  | `gpio_af_set()` + `gpio_mode_set()` + `gpio_output_options_set()` 三步 |

### Q：中断处理方式有什么不同？

**STM32 HAL 方案**：

```c
// 中断自动分发到 HAL 回调
void USART1_IRQHandler(void) { HAL_UART_IRQHandler(&huart1); }
void DMA2_Stream2_IRQHandler(void) { HAL_DMA_IRQHandler(&hdma_usart1_rx); }
void DMA2_Stream7_IRQHandler(void) { HAL_DMA_IRQHandler(&hdma_usart1_tx); }

// 统一回调
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size) {
    usart_rx_check();  // HT/TC/IDLE 统一入口
}
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart) {
    lwrb_skip(&usart_tx_rb, usart_tx_dma_current_len);
    usart_tx_dma_current_len = 0;
    usart_start_tx_dma_transfer();
}
```

**GD32 标准库方案**：

```c
// 手动编写每个中断处理函数
void USART0_IRQHandler(void) {
    if (RESET != usart_interrupt_flag_get(USART0, USART_INT_FLAG_IDLE)) {
        usart_interrupt_flag_clear(USART0, USART_INT_FLAG_IDLE);
        usart_data_receive(USART0);  // 读 DR 清除 IDLE
        usart_rx_check();
    }
}

void DMA1_Channel2_IRQHandler(void) {  // RX DMA
    if (dma_interrupt_flag_get(DMA1, DMA_CH2, DMA_INT_FLAG_FTF)) {
        dma_interrupt_flag_clear(DMA1, DMA_CH2, DMA_INT_FLAG_FTF);
        usart_rx_check();
    }
    if (dma_interrupt_flag_get(DMA1, DMA_CH2, DMA_INT_FLAG_HTF)) {
        dma_interrupt_flag_clear(DMA1, DMA_CH2, DMA_INT_FLAG_HTF);
        usart_rx_check();
    }
}

void DMA1_Channel7_IRQHandler(void) {  // TX DMA
    if (dma_interrupt_flag_get(DMA1, DMA_CH7, DMA_INT_FLAG_FTF)) {
        dma_interrupt_flag_clear(DMA1, DMA_CH7, DMA_INT_FLAG_FTF);
        lwrb_skip(&usart_tx_rb, usart_tx_dma_current_len);
        usart_tx_dma_current_len = 0;
        usart_start_tx_dma_transfer();
    }
}
```

### Q：DMA 初始化的流程差异？

**STM32 HAL**（CubeMX 生成）：

```c
// RX DMA - 在 UART 初始化中自动关联
hdma_usart1_rx.Instance = DMA2_Stream2;
hdma_usart1_rx.Init.Channel = DMA_CHANNEL_4;
hdma_usart1_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
hdma_usart1_rx.Init.Mode = DMA_CIRCULAR;
HAL_DMA_Init(&hdma_usart1_rx);
__HAL_LINKDMA(&huart1, hdmarx, hdma_usart1_rx);  // 关联到 UART 句柄

// 启动接收
HAL_UARTEx_ReceiveToIdle_DMA(&huart1, usart_rx_dma_buffer, 64);
__HAL_DMA_DISABLE_IT(&hdma_usart1_rx, DMA_IT_HT);  // 可选：关闭 HT 中断
```

**GD32 标准库**（手动配置）：

```c
dma_single_data_parameter_struct dma_init;
dma_init.periph_addr = (uint32_t)&USART_DATA(USART0);
dma_init.periph_inc = DMA_PERIPH_INCREASE_DISABLE;
dma_init.memory0_addr = (uint32_t)usart_rx_dma_buffer;
dma_init.memory_inc = DMA_MEMORY_INCREASE_ENABLE;
dma_init.periph_memory_width = DMA_PERIPH_WIDTH_8BIT;
dma_init.circular_mode = DMA_CIRCULAR_MODE_ENABLE;  // 关键！
dma_init.direction = DMA_PERIPH_TO_MEMORY;
dma_init.number = USART0_RX_DMA_LENGTH;
dma_init.priority = DMA_PRIORITY_MEDIUM;
dma_single_data_mode_init(USART0_DMA, USART0_RX_DMA_CH, &dma_init);
dma_channel_subperipheral_select(USART0_DMA, USART0_RX_DMA_CH, USART0_DMA_SUBPERI);
dma_channel_enable(USART0_DMA, USART0_RX_DMA_CH);
```

### Q：NVIC 中断优先级有什么讲究？

所有相关中断（USART、RX DMA、TX DMA）必须设为**相同优先级**，确保它们不会互相抢占。这是因为 `usart_rx_check()` 不是可重入的——如果 RX DMA 的 TC 中断打断了 IDLE 中断中的 `usart_rx_check()`，会导致 `old_pos` 状态混乱。

| 平台  | 优先级设置                    |
| ----- | ----------------------------- |
| STM32 | 全部设为 `(0, 0)`             |
| GD32  | 全部设为 `(2, 2)` 或 `(2, 5)` |

> [!WARNING]
> 如果应用中还有其他更高优先级的中断（如定时器），UART 相关中断可以整体降低优先级，但三者之间必须相同。

---

## 七、常见问题与踩坑

### Q：为什么 DMA 初始化必须在 UART 初始化之前？

DMA 时钟必须先于 UART 时钟使能。STM32 中 `MX_DMA_Init()` 必须在 `MX_USART1_UART_Init()` 之前调用，否则 HAL 内部关联 DMA 句柄时会失败。CubeMX 生成的代码默认遵循此顺序，但手动调整时容易出错。

### Q：IDLE 中断标志怎么清除？

IDLE 标志的清除方式比较特殊——需要**先读状态寄存器，再读数据寄存器**：

| 平台      | 清除方式                                                |
| --------- | ------------------------------------------------------- |
| STM32 HAL | `__HAL_UART_CLEAR_IDLEFLAG()` (内部读 SR + DR)          |
| GD32      | `usart_interrupt_flag_clear()` + `usart_data_receive()` |

如果清除不正确，IDLE 中断会持续触发，进入死循环。

### Q：`HAL_UARTEx_ReceiveToIdle_DMA()` 为什么要每次回调后重新调用？

STM32 HAL 的设计：虽然 RX DMA 配置为 Circular 模式不会停止，但 `ReceiveToIdle_DMA` 内部还管理了 IDLE 中断的使能。回调中如果不重新调用，IDLE 中断可能不会再次触发。HT 和 TC 中断则由 DMA 硬件自动触发，不需要重新使能。

GD32 方案不存在这个问题——IDLE 中断在初始化时一次性使能，不需要每次重启。

### Q：RX 缓冲区大小怎么选？

| 缓冲区大小 | HT 触发间隔 (115200bps) | 适填场景             |
| ---------- | ----------------------- | -------------------- |
| 32 字节    | ~1.4 ms                 | 低速、短包           |
| 64 字节    | ~2.8 ms                 | 通用（推荐）         |
| 128 字节   | ~5.6 ms                 | 高速、长包           |
| 256 字节   | ~11.1 ms                | 超高速或主频低的 MCU |

原则：**缓冲区大小 ≥ 最大连续数据帧长度 × 2**。HT 和 TC 将缓冲区一分为二，各半必须能容纳一帧数据。

### Q：printf 重定向为什么用阻塞发送？

示例中的 `fputc()` 使用轮询方式逐字节发送，不用 DMA。原因是 printf 通常在调试场景使用，调用时需要确保数据完整输出。如果用 DMA 异步发送，printf 返回时数据可能还没发完。实际产品中建议用 `usart_send_string()` 代替 printf。

---

## 八、性能对比：DMA vs 中断 vs 轮询

### Q：三种方式的 CPU 开销对比如何？

以 115200bps、168MHz 主频为例，接收 1KB 数据：

| 方式 | CPU 时间        | 占比 | 数据丢失风险      |
| ---- | --------------- | ---- | ----------------- |
| 轮询 | ~5.6 ms (100%)  | 100% | 无（但 CPU 全占） |
| 中断 | ~0.5 ms (9%)    | ~9%  | 中断延迟时可能    |
| DMA  | ~0.02 ms (0.4%) | < 1% | 几乎为零          |

DMA 方式的 CPU 时间仅用于 `usart_rx_check()` 中的指针计算和数据搬运（写入 TX 环形缓冲区），实际 DMA 传输过程 CPU 不参与。

### Q：DMA 方式的延迟有多大？

| 事件                               | 最大延迟                              |
| ---------------------------------- | ------------------------------------- |
| 收到数据到 HT/TC 中断触发          | 半缓冲区传输时间 (64B@115200 ≈ 2.8ms) |
| 收到数据到 IDLE 中断触发           | 1 字符时间 + IDLE 检测时间 (≈ 0.1ms)  |
| TX 数据从写入环形缓冲区到 DMA 启动 | 当前 DMA 传输完成时间 + 临界区时间    |

IDLE 中断是延迟最低的触发方式，适合对实时性要求高的场景。

---

## 九、移植指南：三步移植到你的平台

### Q：如何将此方案移植到其他 MCU？

**第一步：配置 RX DMA**

1. DMA 配置为 Circular 模式，外设到内存
2. 外设地址 = USART 数据寄存器地址
3. 内存地址 = `usart_rx_dma_buffer`
4. 使能 HT + TC 中断
5. 使能 USART IDLE 中断

**第二步：配置 TX DMA**

1. DMA 配置为 Normal 模式，内存到外设
2. 不在初始化时启动（按需启动）
3. 使能 TC 中断

**第三步：实现三个核心函数**

1. `usart_rx_check()` — 读 DMA 计数器，计算位置，处理数据
2. `usart_start_tx_dma_transfer()` — 临界区保护，零拷贝启动
3. TX 完成中断处理 — `lwrb_skip()` + 链式启动

---

## 十、环形缓冲区 vs 乒乓缓存：应用范围考量

### Q：什么是乒乓缓存（Ping-Pong Buffer）？

乒乓缓存使用**两个等大的线性缓冲区**交替工作：DMA 写入 A 缓冲区时，CPU 处理 B 缓冲区；DMA 半传输完成时交换角色。本质上就是 DMA Circular + HT/TC 中断的"半缓冲区"机制。

```
缓冲区 A [0, N/2)   ← HT 中断时 CPU 处理 A，DMA 继续写 B
缓冲区 B [N/2, N)   ← TC 中断时 CPU 处理 B，DMA 继续写 A（回绕）
```

### Q：环形缓冲区和乒乓缓存的核心区别是什么？

| 维度       | 环形缓冲区 (lwrb)            | 乒乓缓存                           |
| ---------- | ---------------------------- | ---------------------------------- |
| 数据结构   | 一段连续内存 + 读写指针      | 两个等大线性缓冲区                 |
| 处理粒度   | 任意长度（按实际写入量处理） | 固定半缓冲区大小                   |
| 数据延迟   | IDLE 中断触发后立即处理      | 必须等 HT/TC 才能处理              |
| 不定长帧   | 天然支持（IDLE 可触发）      | 需要额外机制处理不足半缓冲区的尾部 |
| 内存利用   | 实际容量 `size - 1`          | 两个缓冲区各 `N/2`，必须等满才处理 |
| 实现复杂度 | 需要处理回绕、线性块         | 逻辑简单，HT/TC 直接切换           |

### Q：各适合什么场景？

**环形缓冲区更适合**：

- **不定长协议帧**：Modbus RTU、自定义命令协议，帧长度不固定，IDLE 中断可立即检测帧结束
- **低延迟交互**：人机交互、AT 指令响应，IDLE 中断保证帧尾不卡
- **TX 异步发送**：零拷贝 + 链式续传，应用层随时写入，DMA 自动排程

**乒乓缓存更适合**：

- **定长采样流**：ADC/I2S 音频采样、传感器高速等间隔数据流，每帧固定长度
- **批量处理管线**：FFT、数字滤波等需要整块数据才能运算的场景
- **双核共享**：一个核写 A，另一个核处理 B，天然无锁交替

### Q：UART 场景为什么首选环形缓冲区？

UART 通信的典型特征：

1. **数据帧长度不固定**：一条 AT 命令可能 10 字节，下一条可能 200 字节
2. **帧间隔不确定**：两次通信之间可能有 ms 级空闲（IDLE 可检测）
3. **需要低延迟响应**：收到命令后应尽快回复，不能等半缓冲区填满

乒乓缓存的 HT/TC 触发机制要求"半缓冲区填满才处理"，如果帧长不足半缓冲区，数据就会一直"卡"在里面。虽然可以加 IDLE 中断来补救，但本质上已经退化成环形缓冲区的处理逻辑了——不如一开始就用环形缓冲区。

> [!TIP]
> 如果你的 UART 场景是**定长高速数据流**（如连续传感器采样），乒乓缓存实现更简单；如果是**不定长命令交互**（绝大多数 UART 场景），环形缓冲区是更自然的选择。

---

## 十一、总结

### 核心设计要点

| 要点                                | 原因                                              |
| ----------------------------------- | ------------------------------------------------- |
| RX DMA Circular                     | 永不停止，无重启窗口期，不丢数据                  |
| TX DMA Normal                       | 每次发一段线性数据，链式续传                      |
| 三重中断 (HT+TC+IDLE)               | HT/TC 保证高速连续流不丢，IDLE 保证不定长包尾不卡 |
| DMA 计数器反推位置                  | 零额外硬件，纯软件计算当前写入位置                |
| TX 零拷贝                           | DMA 直接从环形缓冲区线性块读取，无中间缓冲区      |
| `usart_tx_dma_current_len` 互斥标志 | 避免应用层和中断层同时操作 DMA                    |
| 同优先级中断                        | 防止 `usart_rx_check()` 被抢占导致状态混乱        |

### 双芯片验证结论

STM32 HAL 和 GD32 标准外设库在**架构层面完全一致**——RX Circular + TX Normal + lwrb 零拷贝 + 三重中断。差异仅在 API 调用方式：

- **HAL 厚封装**：回调机制统一管理中断，开发快但调试时调用栈深
- **标准库薄封装**：手动管理中断标志，代码透明度高但细节多

选择哪种取决于项目需求——快速原型选 HAL，深度调试选标准库。

---

## 参考资料

- [lwrb — Lightweight ring buffer library](https://github.com/MaJerle/lwrb)
- [stm32-usart-uart-dma-rx-tx — UART DMA RX/TX example](https://github.com/MaJerle/stm32-usart-uart-dma-rx-tx)
- [STM32F407VGT6 UART DMA Efficient RingBuffer — 本文 STM32 工程](https://github.com/Lingjia007/STM32F407VGT6_UART_DMA_Efficient_RingBuffer)
- [GD32F470ZGT6 UART DMA Efficient RingBuffer — 本文 GD32 工程](https://github.com/Lingjia007/GD32F470ZGT6_UART_DMA_Efficient_RingBuffer)
- [STM32F407 Reference Manual (RM0090)](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [GD32F4xx User Manual](https://www.gigadevice.com.cn/)
- [C语言的精髓：结构体与指针](../oop-in-c-embedded/)
- [STM32F407 安全 Bootloader 设计](../stm32f407-secure-bootloader/)
