---
title: "Efficient UART DMA with Ring Buffer Zero-Copy: Verified on STM32 and GD32 Dual Platforms"
date: 2026-07-05
description: "Based on the lwrb lightweight ring buffer library and stm32-usart-uart-dma-rx-tx reference implementation, this post details the complete UART DMA circular receive + zero-copy transmit architecture, verified on both LCEDA SkyStar STM32F407VGT6 (HAL) and LCEDA LiangShan GD32F470ZGT6 (Standard Peripheral Library), with in-depth analysis of triple interrupt triggers, DMA position inference, and chained TX transmission"
image: UART DMA 零拷贝实战.png
categories:
  - "Embedded"
  - "Driver"
tags:
  - "STM32"
  - "GD32"
  - "UART"
  - "DMA"
  - "Ring-Buffer"
  - "lwrb"
  - "Zero-Copy"
math: true
---

## Foreword

In embedded development, UART serial communication is one of the most fundamental and commonly used peripherals. However, many developers still use byte-by-byte interrupt-driven or polling approaches, resulting in high CPU overhead and significant risk of data loss. The **DMA + ring buffer** combination is the proper way to achieve efficient UART communication.

This post is based on the [lwrb](https://github.com/MaJerle/lwrb) lightweight ring buffer library and the [stm32-usart-uart-dma-rx-tx](https://github.com/MaJerle/stm32-usart-uart-dma-rx-tx) reference implementation, verified on both **LCEDA SkyStar STM32F407VGT6 (HAL library)** and **LCEDA LiangShan GD32F470ZGT6 (Standard Peripheral Library)** dual platforms. Complete project sources: [STM32F407VGT6 UART DMA](https://github.com/Lingjia007/STM32F407VGT6_UART_DMA_Efficient_RingBuffer) and [GD32F470ZGT6 UART DMA](https://github.com/Lingjia007/GD32F470ZGT6_UART_DMA_Efficient_RingBuffer). Every design decision is deeply analyzed in a Q&A format.

---

## 1. Why Not Use Byte-by-Byte Interrupt Transmission?

### Q: What's wrong with interrupt-driven transmission?

The interrupt approach triggers an interrupt for every byte sent or received, which becomes a severe problem at high baud rates:

| Baud Rate | Bytes/sec | Interrupt Freq | CPU Overhead at 9600bps |
| --------- | --------- | -------------- | ----------------------- |
| 9600      | 960       | ~960 Hz        | Negligible              |
| 115200    | 11520     | ~11.5 kHz      | ~5-10%                  |
| 921600    | 92160     | ~92 kHz        | ~30-50%                 |
| 3000000   | 300000    | ~300 kHz       | Nearly unusable         |

Each interrupt requires saving/restoring context (at least 12 registers pushed onto the stack), consuming significant CPU time in interrupt switching. Even more dangerously: **if interrupt processing is slightly slow, the ORE (Overrun Error) flag will be set, causing data loss**.

### Q: What can DMA solve?

DMA (Direct Memory Access) allows hardware to automatically transfer data between the USART data register and memory, with zero CPU involvement in byte-by-byte transfers. At 115200bps, the DMA approach has nearly 0% CPU overhead.

---

## 2. Overall Architecture: RX Circular + TX Zero-Copy

### Q: Why do RX and TX use different DMA modes?

| Path | DMA Mode     | Reason                                                                                                                                                    |
| ---- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RX   | **Circular** | Reception is passive — you don't know when data will arrive or how long it will be. Circular mode lets DMA automatically wrap around, never stopping      |
| TX   | **Normal**   | Transmission is active — each time a linear segment is fetched from the ring buffer and sent. After completion, the next segment is launched via chaining |

### Q: What does the overall data flow look like?

```
RX Path:
[External Data] -> USART_DR -> DMA(Circular) -> usart_rx_dma_buffer[64]
                                              |
                              IDLE / HT / TC triple interrupt triggers
                                              |
                                     usart_rx_check()
                                     (DMA counter infers write position)
                                              |
                                     usart_process_data()
                                              |
                                     lwrb_write() -> TX ring buffer

TX Path:
Application lwrb_write() -> TX ring buffer
                              |
                   usart_start_tx_dma_transfer()
                   (Zero-copy: DMA reads directly from ring buffer linear block)
                              |
                   DMA(Normal) -> USART_DR -> [External]
                              |
                   TX complete interrupt -> lwrb_skip() + chained launch of next segment
```

---

## 3. RX Path: DMA Circular + Triple Interrupt

### Q: Why does RX DMA use Circular mode?

In Normal mode, DMA automatically stops after a transfer completes, and must be manually restarted in an interrupt. Any data arriving during the restart window will be lost. In Circular mode, after DMA writes to the end of the buffer, it automatically wraps back to the beginning and continues writing, **never stopping** — there is no restart window.

### Q: What are the triple interrupts? Why are three needed?

| Interrupt                  | Trigger Timing                  | Purpose                                                                                                                          |
| -------------------------- | ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **HT (Half Transfer)**     | When DMA writes to half buffer  | Process the first half of data promptly, preventing it from being overwritten by the second half                                 |
| **TC (Transfer Complete)** | When DMA writes the full buffer | Process the second half of data promptly, preventing it from being overwritten by the new first half                             |
| **IDLE Line**              | When UART detects bus idle      | Handle the tail of variable-length data — the last packet may be smaller than half the buffer, so neither HT nor TC will trigger |

**Key point**: With only HT + TC, when the data volume is less than half the buffer size, no interrupt will ever trigger, and data will be "stuck" in the DMA buffer. The IDLE interrupt solves this — bus idle means a frame of data has ended.

### Q: How does the DMA counter infer the write position?

DMA hardware maintains a decrementing counter `CNDTR` (`CNT` in GD32), initialized to the buffer size, decrementing by 1 after each transfer. The current write position:

$$\text{pos} = N - \text{CNDTR}$$

where $N$ is the buffer size. For example, with `N=64` and `CNDTR=52`, `pos=12`, meaning DMA has written the first 12 bytes.

**STM32 HAL implementation:**

```c
size_t pos = ARRAY_LEN(usart_rx_dma_buffer) - __HAL_DMA_GET_COUNTER(&hdma_usart1_rx);
```

**GD32 Standard Peripheral Library implementation:**

```c
size_t pos = USART0_RX_DMA_LENGTH - dma_transfer_number_get(USART0_DMA, USART0_RX_DMA_CH);
```

### Q: How is wrap-around handled?

DMA Circular mode automatically wraps back to the beginning after writing to the end of the buffer. `usart_rx_check()` needs to handle two cases:

| Case         | Condition       | Processing                                                  |
| ------------ | --------------- | ----------------------------------------------------------- |
| Linear write | `pos > old_pos` | Single call to `usart_process_data(old_pos, pos - old_pos)` |
| Wrap-around  | `pos < old_pos` | Two segments: tail `[old_pos, N)` + head `[0, pos)`         |

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
> `pos == old_pos` means no new data has arrived and no processing is needed. This check must not be omitted — the DMA counter may not have changed between two checks.

### Q: What is the unified callback in STM32 HAL?

STM32 HAL provides `HAL_UARTEx_ReceiveToIdle_DMA()`, which unifies HT, TC, and IDLE interrupts into a single callback `HAL_UARTEx_RxEventCallback()`:

```c
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size) {
    if (huart->Instance == USART1) {
        usart_rx_check();  // All three interrupts ultimately call the same function
    }
}
```

The GD32 approach requires manually calling `usart_rx_check()` in each of the three interrupt handlers separately.

---

## 4. TX Path: Zero-Copy DMA + Chained Transmission

### Q: What is zero-copy? Why can TX be zero-copy?

Traditional approach: first `memcpy` data from the ring buffer to a linear buffer, then give the linear buffer address to DMA. This involves one extra memory copy.

**Zero-copy**: DMA reads directly from the ring buffer's contiguous memory region, with no intermediate buffer. This leverages lwrb's linear block interface:

```c
size_t len = lwrb_get_linear_block_read_length(&usart_tx_rb);    // Contiguous readable length
void *addr = lwrb_get_linear_block_read_address(&usart_tx_rb);   // Contiguous readable start address
// DMA reads len bytes directly from addr
```

### Q: The ring buffer data might span the wrap-around boundary, right? DMA can only read contiguous memory — how is this handled?

Indeed, ring buffer data may cross the buffer end and wrap back to the beginning, forming two non-contiguous regions. lwrb's `get_linear_block_read_length()` only returns the **contiguous length from the read pointer to the write pointer (or buffer end)**, never crossing the wrap-around boundary.

The solution: **chained transmission**. Each DMA transfer sends only one linear segment. In the transfer-complete interrupt, check if the ring buffer still has data; if so, launch the next DMA round. If the ring buffer data crosses a wrap-around boundary, it is automatically split into two DMA transfers:

```
Ring Buffer: [....ABCDE.....FGH....]
                     ^              ^
                   read ptr      write ptr

DMA 1st transfer: Send "ABCDE" (linear segment to buffer end)
DMA 2nd transfer: Send "FGH"  (new linear segment from buffer beginning)
```

### Q: What is the TX DMA startup flow?

```c
void usart_start_tx_dma_transfer(void) {
    __disable_irq();  // Enter critical section

    if (usart_tx_dma_current_len == 0) {  // DMA idle
        size_t len = lwrb_get_linear_block_read_length(&usart_tx_rb);
        if (len > 0) {
            void *addr = lwrb_get_linear_block_read_address(&usart_tx_rb);
            usart_tx_dma_current_len = len;

            // STM32 HAL:
            // HAL_DMA_Abort(&hdma_usart1_tx);
            // HAL_UART_Transmit_DMA(&huart1, addr, len);

            // GD32 Standard Peripheral Library:
            dma_channel_disable(USART0_DMA, USART0_TX_DMA_CH);
            dma_flag_clear(USART0_DMA, USART0_TX_DMA_CH, ...);
            dma_transfer_number_config(USART0_DMA, USART0_TX_DMA_CH, len);
            dma_memory_address_config(USART0_DMA, USART0_TX_DMA_CH, (uint32_t)addr);
            dma_channel_enable(USART0_DMA, USART0_TX_DMA_CH);
        }
    }

    __enable_irq();  // Exit critical section
}
```

### Q: Why is the `usart_tx_dma_current_len` global variable needed?

It serves as a **mutex flag**: a value of 0 means DMA is idle and a new transfer can be started; non-zero means DMA is currently transferring and must not be disturbed. After writing to the ring buffer, the application calls `usart_start_tx_dma_transfer()`, which returns immediately if DMA is busy — the data safely remains in the ring buffer, and the current transfer-complete interrupt will automatically launch the next round.

### Q: What happens in the TX complete interrupt?

```c
// 1. Skip the sent data (advance the ring buffer read pointer)
lwrb_skip(&usart_tx_rb, usart_tx_dma_current_len);

// 2. Mark DMA as idle
usart_tx_dma_current_len = 0;

// 3. Attempt chained launch of next round
usart_start_tx_dma_transfer();
```

`lwrb_skip()` only moves the read pointer with zero data copying — this is the other half of zero-copy.

---

## 5. lwrb Ring Buffer Core Design

### Q: What makes lwrb special?

| Feature          | Description                                                                                  |
| ---------------- | -------------------------------------------------------------------------------------------- |
| Lightweight      | One header + one source file, no dynamic allocation                                          |
| Thread-safe      | Read/write pointers use `volatile`, no locks needed in single-writer single-reader scenarios |
| Zero-copy API    | `get_linear_block_read_address/length` exposes internal addresses directly to DMA            |
| Magic word check | `magic1=0xDEADBEEF, magic2=~0xDEADBEEF`, detects whether buffer is initialized               |
| Actual capacity  | `size - 1`, full condition is `w == r - 1`                                                   |

### Q: Why is the buffer full condition `w == r - 1` instead of `w == r`?

`w == r` means "empty" (read and write pointers coincide). If full also used `w == r`, there would be no way to distinguish empty from full. By reserving one byte (keeping `w` one step behind `r`), the state can be determined from the pointer relationship:

| Condition             | State                 |
| --------------------- | --------------------- |
| `w == r`              | Empty                 |
| `(w + 1) % size == r` | Full                  |
| Otherwise             | Has data but not full |

### Q: Why is lwrb's linear block interface critical for DMA?

DMA can only transfer **contiguous** memory regions. Ring buffer data, when wrapped, is split into two non-contiguous regions. The linear block interface returns the **contiguous region from the read pointer to the buffer end (or the write pointer, whichever is smaller)**, which DMA can use directly:

```c
// Returns contiguous readable length from the read pointer
size_t lwrb_get_linear_block_read_length(lwrb_t *lwrb) {
    size_t len, w;
    w = lwrb->w;  // Read write pointer once
    if (w >= lwrb->r) {
        len = w - lwrb->r;  // Linear region
    } else {
        len = lwrb->size - lwrb->r;  // Read pointer to buffer end
    }
    return len;
}
```

---

## 6. STM32 vs GD32: Dual-Platform Implementation Comparison

### Q: What are the differences in DMA architecture between the two platforms?

| Dimension               | STM32F407                                           | GD32F470                                                                      |
| ----------------------- | --------------------------------------------------- | ----------------------------------------------------------------------------- |
| DMA Model               | **Stream + Channel** (DMA2_Stream2_CH4)             | **Channel + Subperi** (DMA1_CH2_SUB4)                                         |
| RX DMA                  | DMA2_Stream2                                        | DMA1_Channel2                                                                 |
| TX DMA                  | DMA2_Stream7                                        | DMA1_Channel7                                                                 |
| DMA Counter             | `__HAL_DMA_GET_COUNTER()` macro                     | `dma_transfer_number_get()` function                                          |
| Interrupt Mgmt          | HAL unified callback (`HAL_UARTEx_RxEventCallback`) | Manual IRQ Handler writing                                                    |
| Abstraction Level       | Thick wrapper (HAL)                                 | Thin wrapper (Standard Peripheral Library)                                    |
| GPIO Alternate Function | `HAL_GPIO_Init()` done in one step                  | `gpio_af_set()` + `gpio_mode_set()` + `gpio_output_options_set()` three steps |

### Q: How do the interrupt handling approaches differ?

**STM32 HAL approach**:

```c
// Interrupts automatically dispatched to HAL callbacks
void USART1_IRQHandler(void) { HAL_UART_IRQHandler(&huart1); }
void DMA2_Stream2_IRQHandler(void) { HAL_DMA_IRQHandler(&hdma_usart1_rx); }
void DMA2_Stream7_IRQHandler(void) { HAL_DMA_IRQHandler(&hdma_usart1_tx); }

// Unified callback
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size) {
    usart_rx_check();  // HT/TC/IDLE unified entry
}
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart) {
    lwrb_skip(&usart_tx_rb, usart_tx_dma_current_len);
    usart_tx_dma_current_len = 0;
    usart_start_tx_dma_transfer();
}
```

**GD32 Standard Peripheral Library approach**:

```c
// Manually write each interrupt handler
void USART0_IRQHandler(void) {
    if (RESET != usart_interrupt_flag_get(USART0, USART_INT_FLAG_IDLE)) {
        usart_interrupt_flag_clear(USART0, USART_INT_FLAG_IDLE);
        usart_data_receive(USART0);  // Read DR to clear IDLE
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

### Q: What are the differences in DMA initialization flow?

**STM32 HAL** (CubeMX generated):

```c
// RX DMA - automatically linked during UART initialization
hdma_usart1_rx.Instance = DMA2_Stream2;
hdma_usart1_rx.Init.Channel = DMA_CHANNEL_4;
hdma_usart1_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
hdma_usart1_rx.Init.Mode = DMA_CIRCULAR;
HAL_DMA_Init(&hdma_usart1_rx);
__HAL_LINKDMA(&huart1, hdmarx, hdma_usart1_rx);  // Link to UART handle

// Start reception
HAL_UARTEx_ReceiveToIdle_DMA(&huart1, usart_rx_dma_buffer, 64);
__HAL_DMA_DISABLE_IT(&hdma_usart1_rx, DMA_IT_HT);  // Optional: disable HT interrupt
```

**GD32 Standard Peripheral Library** (manual configuration):

```c
dma_single_data_parameter_struct dma_init;
dma_init.periph_addr = (uint32_t)&USART_DATA(USART0);
dma_init.periph_inc = DMA_PERIPH_INCREASE_DISABLE;
dma_init.memory0_addr = (uint32_t)usart_rx_dma_buffer;
dma_init.memory_inc = DMA_MEMORY_INCREASE_ENABLE;
dma_init.periph_memory_width = DMA_PERIPH_WIDTH_8BIT;
dma_init.circular_mode = DMA_CIRCULAR_MODE_ENABLE;  // Key!
dma_init.direction = DMA_PERIPH_TO_MEMORY;
dma_init.number = USART0_RX_DMA_LENGTH;
dma_init.priority = DMA_PRIORITY_MEDIUM;
dma_single_data_mode_init(USART0_DMA, USART0_RX_DMA_CH, &dma_init);
dma_channel_subperipheral_select(USART0_DMA, USART0_RX_DMA_CH, USART0_DMA_SUBPERI);
dma_channel_enable(USART0_DMA, USART0_RX_DMA_CH);
```

### Q: What are the considerations for NVIC interrupt priority?

All related interrupts (USART, RX DMA, TX DMA) must be set to the **same priority** to ensure they do not preempt each other. This is because `usart_rx_check()` is not reentrant — if the RX DMA TC interrupt preempts `usart_rx_check()` during an IDLE interrupt, it will cause `old_pos` state corruption.

| Platform | Priority Setting                |
| -------- | ------------------------------- |
| STM32    | All set to `(0, 0)`             |
| GD32     | All set to `(2, 2)` or `(2, 5)` |

> [!WARNING]
> If the application has other higher-priority interrupts (e.g., timers), the UART-related interrupts can be lowered in priority as a group, but the three must remain at the same priority relative to each other.

---

## 7. Common Pitfalls and FAQs

### Q: Why must DMA initialization come before UART initialization?

The DMA clock must be enabled before the UART clock. On STM32, `MX_DMA_Init()` must be called before `MX_USART1_UART_Init()`, otherwise the HAL will fail when internally linking the DMA handle. CubeMX-generated code follows this order by default, but it is easy to get wrong when manually adjusting the code.

### Q: How is the IDLE interrupt flag cleared?

The IDLE flag clearing method is somewhat special — it requires **reading the status register first, then reading the data register**:

| Platform  | Clearing Method                                          |
| --------- | -------------------------------------------------------- |
| STM32 HAL | `__HAL_UART_CLEAR_IDLEFLAG()` (internally reads SR + DR) |
| GD32      | `usart_interrupt_flag_clear()` + `usart_data_receive()`  |

If not cleared correctly, the IDLE interrupt will trigger continuously, entering an infinite loop.

### Q: Why does `HAL_UARTEx_ReceiveToIdle_DMA()` need to be called again after each callback?

This is by STM32 HAL design: although RX DMA configured in Circular mode does not stop, `ReceiveToIdle_DMA` internally also manages IDLE interrupt enabling. If not called again in the callback, the IDLE interrupt may not trigger again. HT and TC interrupts are automatically triggered by DMA hardware and do not need re-enabling.

The GD32 approach does not have this issue — the IDLE interrupt is enabled once during initialization and does not need to be restarted each time.

### Q: How should the RX buffer size be chosen?

| Buffer Size | HT Trigger Interval (115200bps) | Suitable Scenario                     |
| ----------- | ------------------------------- | ------------------------------------- |
| 32 bytes    | ~1.4 ms                         | Low speed, short packets              |
| 64 bytes    | ~2.8 ms                         | General use (recommended)             |
| 128 bytes   | ~5.6 ms                         | High speed, long packets              |
| 256 bytes   | ~11.1 ms                        | Ultra-high speed or low-frequency MCU |

Rule: **Buffer size >= maximum continuous data frame length x 2**. HT and TC split the buffer in half, and each half must be able to hold one frame of data.

### Q: Why does printf redirection use blocking transmission?

The `fputc()` in the example uses polling mode for byte-by-byte transmission instead of DMA. The reason is that printf is typically used in debugging scenarios where you need to ensure data is fully output before returning. With DMA asynchronous transmission, the data may not have been fully sent when printf returns. In production, it is recommended to use `usart_send_string()` instead of printf.

---

## 8. Performance Comparison: DMA vs Interrupt vs Polling

### Q: How do the CPU overheads of the three approaches compare?

Using 115200bps, 168MHz main frequency, receiving 1KB of data:

| Approach  | CPU Time        | Proportion | Data Loss Risk                   |
| --------- | --------------- | ---------- | -------------------------------- |
| Polling   | ~5.6 ms (100%)  | 100%       | None (but CPU fully occupied)    |
| Interrupt | ~0.5 ms (9%)    | ~9%        | Possible under interrupt latency |
| DMA       | ~0.02 ms (0.4%) | < 1%       | Nearly zero                      |

The CPU time for the DMA approach is only used for pointer calculation and data movement in `usart_rx_check()` (writing to the TX ring buffer); the actual DMA transfer process involves no CPU participation.

### Q: What is the latency of the DMA approach?

| Event                                       | Maximum Latency                                              |
| ------------------------------------------- | ------------------------------------------------------------ |
| Data received to HT/TC interrupt trigger    | Half-buffer transfer time (64B@115200 ~= 2.8ms)              |
| Data received to IDLE interrupt trigger     | 1 character time + IDLE detection time (~= 0.1ms)            |
| TX data from ring buffer write to DMA start | Current DMA transfer completion time + critical section time |

The IDLE interrupt is the lowest-latency trigger method, suitable for scenarios with strict real-time requirements.

---

## 9. Porting Guide: Three Steps to Port to Your Platform

### Q: How to port this solution to other MCUs?

**Step 1: Configure RX DMA**

1. Configure DMA as Circular mode, peripheral-to-memory
2. Peripheral address = USART data register address
3. Memory address = `usart_rx_dma_buffer`
4. Enable HT + TC interrupts
5. Enable USART IDLE interrupt

**Step 2: Configure TX DMA**

1. Configure DMA as Normal mode, memory-to-peripheral
2. Do not start at initialization (start on demand)
3. Enable TC interrupt

**Step 3: Implement three core functions**

1. `usart_rx_check()` — read DMA counter, calculate position, process data
2. `usart_start_tx_dma_transfer()` — critical section protection, zero-copy startup
3. TX complete interrupt handler — `lwrb_skip()` + chained launch

---

## 10. Ring Buffer vs Ping-Pong Buffer: Application Scope

### Q: What is a Ping-Pong Buffer?

A ping-pong buffer uses **two equal-sized linear buffers** that alternate: while DMA writes to buffer A, the CPU processes buffer B; when DMA reaches the half-transfer point, the roles swap. This is essentially the "half-buffer" mechanism of DMA Circular mode with HT/TC interrupts.

```
Buffer A [0, N/2)    ← HT interrupt: CPU processes A, DMA continues writing B
Buffer B [N/2, N)    ← TC interrupt: CPU processes B, DMA wraps to write A
```

### Q: What are the core differences between ring buffer and ping-pong buffer?

| Dimension                 | Ring Buffer (lwrb)                                | Ping-Pong Buffer                                        |
| ------------------------- | ------------------------------------------------- | ------------------------------------------------------- |
| Data structure            | One contiguous memory block + read/write pointers | Two equal-sized linear buffers                          |
| Processing granularity    | Any length (process actual written amount)        | Fixed half-buffer size                                  |
| Data latency              | Process immediately after IDLE interrupt          | Must wait for HT/TC to process                          |
| Variable-length frames    | Native support (IDLE can trigger)                 | Needs extra mechanism for tail shorter than half-buffer |
| Memory utilization        | Effective capacity `size - 1`                     | Each buffer `N/2`, must wait until full                 |
| Implementation complexity | Must handle wraparound, linear blocks             | Simple logic, HT/TC direct swap                         |

### Q: Which is suitable for which scenario?

**Ring buffer is better for**:

- **Variable-length protocol frames**: Modbus RTU, custom command protocols where frame length is not fixed; IDLE interrupt can immediately detect frame end
- **Low-latency interaction**: Human-machine interaction, AT command response; IDLE interrupt ensures frame tails don't get stuck
- **TX asynchronous transmission**: Zero-copy + chained continuation; application writes anytime, DMA auto-schedules

**Ping-pong buffer is better for**:

- **Fixed-length sample streams**: ADC/I2S audio sampling, high-speed equi-interval sensor data streams, each frame fixed length
- **Batch processing pipelines**: FFT, digital filtering — scenarios requiring a complete block of data before computation
- **Dual-core sharing**: One core writes A, the other processes B; naturally lock-free alternation

### Q: Why is ring buffer the first choice for UART?

Typical characteristics of UART communication:

1. **Variable frame length**: An AT command might be 10 bytes, the next might be 200 bytes
2. **Uncertain inter-frame gaps**: There may be ms-level idle time between communications (detectable by IDLE)
3. **Low-latency response needed**: Commands should be processed as soon as received, not waiting for half-buffer to fill

Ping-pong buffer's HT/TC trigger mechanism requires "half-buffer full before processing." If the frame is shorter than half-buffer, data stays stuck inside. Adding IDLE interrupt as a remedy essentially degenerates into ring buffer logic — better to use ring buffer from the start.

> [!TIP]
> If your UART scenario involves **fixed-length high-speed data streams** (e.g., continuous sensor sampling), ping-pong buffer is simpler to implement; for **variable-length command interaction** (the vast majority of UART scenarios), ring buffer is the more natural choice.

---

## 11. Summary

### Core Design Points

| Point                                 | Reason                                                                                                            |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| RX DMA Circular                       | Never stops, no restart window, no data loss                                                                      |
| TX DMA Normal                         | Send one linear segment at a time, chain-continue                                                                 |
| Triple interrupt (HT+TC+IDLE)         | HT/TC ensures no loss in high-speed continuous streams; IDLE ensures variable-length packet tails don't get stuck |
| DMA counter infers position           | Zero extra hardware, pure software calculation of current write position                                          |
| TX zero-copy                          | DMA reads directly from ring buffer linear block, no intermediate buffer                                          |
| `usart_tx_dma_current_len` mutex flag | Prevents application and interrupt layers from operating DMA simultaneously                                       |
| Same-priority interrupts              | Prevents `usart_rx_check()` from being preempted, avoiding state corruption                                       |

### Dual-Platform Verification Conclusions

STM32 HAL and GD32 Standard Peripheral Library are **architecturally identical** — RX Circular + TX Normal + lwrb zero-copy + triple interrupt. Differences lie only in API calling conventions:

- **HAL thick wrapper**: Callback mechanism manages interrupts uniformly, fast development but deep call stacks during debugging
- **Standard Library thin wrapper**: Manual interrupt flag management, high code transparency but more detail work

Which to choose depends on project requirements — HAL for rapid prototyping, Standard Library for in-depth debugging.

---

## References

- [lwrb — Lightweight ring buffer library](https://github.com/MaJerle/lwrb)
- [stm32-usart-uart-dma-rx-tx — UART DMA RX/TX example](https://github.com/MaJerle/stm32-usart-uart-dma-rx-tx)
- [STM32F407VGT6 UART DMA Efficient RingBuffer — STM32 project for this post](https://github.com/Lingjia007/STM32F407VGT6_UART_DMA_Efficient_RingBuffer)
- [GD32F470ZGT6 UART DMA Efficient RingBuffer — GD32 project for this post](https://github.com/Lingjia007/GD32F470ZGT6_UART_DMA_Efficient_RingBuffer)
- [STM32F407 Reference Manual (RM0090)](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [GD32F4xx User Manual](https://www.gigadevice.com.cn/)
- [The Essence of C: Structs and Pointers](../oop-in-c-embedded/)
- [STM32F407 Secure Bootloader Design](../stm32f407-secure-bootloader/)
