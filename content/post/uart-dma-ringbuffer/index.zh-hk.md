---
title: "UART DMA 高效收發：環形緩衝區零拷貝實戰，STM32 與 GD32 雙晶片驗證"
date: 2026-07-05
description: "基於 lwrb 環形緩衝區函式庫，詳解 UART DMA 循環接收 + 零拷貝發送的完整架構，以 STM32F407 (HAL) 和 GD32F470 (標準外設庫) 雙平台實現驗證，三重中斷觸發、DMA 位置反推、鏈式 TX 發送逐一剖析"
image: UART DMA 零拷贝实战.png
categories:
  - "嵌入式"
tags:
  - "STM32"
  - "GD32"
  - "UART"
  - "DMA"
  - "環形緩衝區"
  - "lwrb"
  - "零拷貝"
math: true
---

## 前言

在嵌入式開發中，UART 串口通信是最基礎也最常用的外設之一。然而，許多開發者仍然在使用中斷逐位元組收發或輪詢方式，導致 CPU 佔用高、資料遺失風險大。**DMA + 環形緩衝區**的組合，才是高效 UART 通信的正確打開方式。

本文基於 [lwrb](https://github.com/MaJerle/lwrb) 輕量級環形緩衝區函式庫和 [stm32-usart-uart-dma-rx-tx](https://github.com/MaJerle/stm32-usart-uart-dma-rx-tx) 參考實現，分別用 **STM32F407VGT6 (HAL 庫)** 和 **GD32F470ZGT6 (標準外設庫)** 完成雙晶片驗證，完整工程見 [STM32F407VGT6 UART DMA](https://github.com/Lingjia007/STM32F407VGT6_UART_DMA_Efficient_RingBuffer) 與 [GD32F470ZGT6 UART DMA](https://github.com/Lingjia007/GD32F470ZGT6_UART_DMA_Efficient_RingBuffer)，以有問有答的方式深入剖析每一個設計決策。

---

## 一、為什麼不用中斷逐位元組收發？

### Q：中斷收發有什麼問題？

中斷方式每收發一個位元組就觸發一次中斷，在高波特率下問題嚴重：

| 波特率  | 每秒位元組數 | 中斷頻率  | 9600bps 時 CPU 開銷 |
| ------- | ------------ | --------- | ------------------- |
| 9600    | 960          | ~960 Hz   | 可忽略              |
| 115200  | 11520        | ~11.5 kHz | ~5-10%              |
| 921600  | 92160        | ~92 kHz   | ~30-50%             |
| 3000000 | 300000       | ~300 kHz  | 幾乎無法工作        |

每個中斷需要保存/恢復上下文（至少 12 個暫存器入堆疊），CPU 大量時間消耗在中斷切換上。更危險的是：**中斷處理稍慢就會溢位 ORE 標誌，導致資料遺失**。

### Q：DMA 能解決什麼？

DMA（Direct Memory Access）讓硬體自動在 USART 資料暫存器和記憶體之間搬運資料，CPU 完全不參與逐位元組的傳輸。對於 115200bps，DMA 方式 CPU 開銷接近 0%。

---

## 二、整體架構：RX 循環 + TX 零拷貝

### Q：RX 和 TX 為什麼用不同的 DMA 模式？

| 通路 | DMA 模式             | 原因                                                                          |
| ---- | -------------------- | ----------------------------------------------------------------------------- |
| RX   | **Circular（循環）** | 接收是被動行為，不知道何時有資料、資料多長。循環模式讓 DMA 自動回繞，永不停止 |
| TX   | **Normal（單次）**   | 發送是主動行為，每次從環形緩衝區取一段線性資料發送。完成後鏈式啟動下一段      |

### Q：整體資料流是怎樣的？

```
RX 通路：
[外部資料] → USART_DR → DMA(Circular) → usart_rx_dma_buffer[64]
                                              ↓
                              IDLE / HT / TC 三重中斷觸發
                                              ↓
                                     usart_rx_check()
                                     (DMA 計數器反推寫入位置)
                                              ↓
                                     usart_process_data()
                                              ↓
                                     lwrb_write() → TX 環形緩衝區

TX 通路：
應用層 lwrb_write() → TX 環形緩衝區
                              ↓
                   usart_start_tx_dma_transfer()
                   (零拷貝: DMA 直接從環形緩衝區線性塊讀取)
                              ↓
                   DMA(Normal) → USART_DR → [外部]
                              ↓
                   TX 完成中斷 → lwrb_skip() + 鏈式啟動下一段
```

---

## 三、RX 通路：DMA Circular + 三重中斷

### Q：為什麼 RX DMA 用 Circular 模式？

Normal 模式下，DMA 傳輸完成後自動停止，必須在中斷中手動重啟。重啟期間的資料就會遺失。Circular 模式下，DMA 寫到緩衝區末尾後自動回到頭部繼續寫，**永不停止**，不存在重啟窗口期。

### Q：三重中斷分別是什麼？為什麼需要三個？

| 中斷                       | 觸發時機                | 作用                                                                    |
| -------------------------- | ----------------------- | ----------------------------------------------------------------------- |
| **HT (Half Transfer)**     | DMA 寫到緩衝區一半時    | 及時處理前半段資料，防止被後半段覆蓋                                    |
| **TC (Transfer Complete)** | DMA 寫完整個緩衝區時    | 及時處理後半段資料，防止被新的前半段覆蓋                                |
| **IDLE Line**              | UART 偵測到匯流排空閒時 | 處理不定長資料的尾部——最後一個封包可能不足半緩衝區，HT 和 TC 都不會觸發 |

**關鍵**：如果只有 HT + TC，當資料量不足半緩衝區時，將永遠不會觸發中斷，資料會「卡」在 DMA 緩衝區中。IDLE 中斷解決了這個問題——匯流排空閒意味著一幀資料結束。

### Q：DMA 計數器怎麼反推寫入位置？

DMA 硬體維護一個遞減計數器 `CNDTR`（GD32 中為 `CNT`），初始值為緩衝區大小，每次傳輸後減 1。當前寫入位置：

$$\text{pos} = N - \text{CNDTR}$$

其中 $N$ 為緩衝區大小。例如 `N=64`，`CNDTR=52` 時，`pos=12`，表示 DMA 已寫入前 12 個位元組。

**STM32 HAL 實現：**

```c
size_t pos = ARRAY_LEN(usart_rx_dma_buffer) - __HAL_DMA_GET_COUNTER(&hdma_usart1_rx);
```

**GD32 標準庫實現：**

```c
size_t pos = USART0_RX_DMA_LENGTH - dma_transfer_number_get(USART0_DMA, USART0_RX_DMA_CH);
```

### Q：回繞時怎麼處理？

DMA Circular 寫到緩衝區末尾會自動回到頭部。`usart_rx_check()` 需要處理兩種情況：

| 情況     | 條件            | 處理                                                  |
| -------- | --------------- | ----------------------------------------------------- |
| 線性寫入 | `pos > old_pos` | 單次呼叫 `usart_process_data(old_pos, pos - old_pos)` |
| 回繞寫入 | `pos < old_pos` | 分兩段：尾部 `[old_pos, N)` + 頭部 `[0, pos)`         |

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
> `pos == old_pos` 表示沒有新資料到來，無需處理。這個判斷不能省略——DMA 計數器可能在兩次檢查之間沒有變化。

### Q：STM32 HAL 的統一回呼是怎麼回事？

STM32 HAL 提供了 `HAL_UARTEx_ReceiveToIdle_DMA()`，它將 HT、TC、IDLE 三種中斷統一到一個回呼 `HAL_UARTEx_RxEventCallback()` 中：

```c
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size) {
    if (huart->Instance == USART1) {
        usart_rx_check();  // 三種中斷最終都呼叫同一個函式
    }
}
```

GD32 方案則需要手動在三個中斷處理函式中分別呼叫 `usart_rx_check()`。

---

## 四、TX 通路：零拷貝 DMA + 鏈式發送

### Q：什麼是零拷貝？為什麼 TX 可以零拷貝？

傳統做法：先把環形緩衝區資料 `memcpy` 到一個線性緩衝區，再把線性緩衝區位址給 DMA。這有一次額外的記憶體拷貝。

**零拷貝**：DMA 直接從環形緩衝區的連續記憶體區域讀取資料，無需中間緩衝區。這利用了 lwrb 的線性塊介面：

```c
size_t len = lwrb_get_linear_block_read_length(&usart_tx_rb);    // 連續可讀長度
void *addr = lwrb_get_linear_block_read_address(&usart_tx_rb);   // 連續可讀起始位址
// DMA 直接從 addr 讀取 len 位元組
```

### Q：環形緩衝區的資料不是可能跨回繞邊界嗎？DMA 只能讀連續記憶體怎麼辦？

確實，環形緩衝區的資料可能跨越緩衝區末尾回到頭部，形成兩段不連續的區域。lwrb 的 `get_linear_block_read_length()` 只返回**從讀指標到寫指標（或緩衝區末尾）的連續長度**，不會跨越回繞邊界。

解決方案：**鏈式發送**。一次 DMA 只發送線性段，傳輸完成中斷中再檢查環形緩衝區是否還有資料，有則啟動下一輪 DMA。如果環形緩衝區資料跨回繞邊界，會自動拆分為兩次 DMA 傳輸：

```
環形緩衝區: [....ABCDE.....FGH....]
                     ↑              ↑
                   讀指標         寫指標

DMA 第 1 次: 發送 "ABCDE" (線性段到緩衝區末尾)
DMA 第 2 次: 發送 "FGH"  (從緩衝區頭部開始的新線性段)
```

### Q：TX DMA 的啟動流程是什麼？

```c
void usart_start_tx_dma_transfer(void) {
    __disable_irq();  // 進入臨界區

    if (usart_tx_dma_current_len == 0) {  // DMA 空閒
        size_t len = lwrb_get_linear_block_read_length(&usart_tx_rb);
        if (len > 0) {
            void *addr = lwrb_get_linear_block_read_address(&usart_tx_rb);
            usart_tx_dma_current_len = len;

            // STM32 HAL:
            // HAL_DMA_Abort(&hdma_usart1_tx);
            // HAL_UART_Transmit_DMA(&huart1, addr, len);

            // GD32 標準庫:
            dma_channel_disable(USART0_DMA, USART0_TX_DMA_CH);
            dma_flag_clear(USART0_DMA, USART0_TX_DMA_CH, ...);
            dma_transfer_number_config(USART0_DMA, USART0_TX_DMA_CH, len);
            dma_memory_address_config(USART0_DMA, USART0_TX_DMA_CH, (uint32_t)addr);
            dma_channel_enable(USART0_DMA, USART0_TX_DMA_CH);
        }
    }

    __enable_irq();  // 退出臨界區
}
```

### Q：為什麼需要 `usart_tx_dma_current_len` 這個全域變數？

它充當**互斥標誌**：值為 0 表示 DMA 空閒，可以啟動新傳輸；非 0 表示 DMA 正在傳輸，不能干擾。應用層寫入環形緩衝區後呼叫 `usart_start_tx_dma_transfer()`，如果 DMA 正忙就返回——資料安全地留在環形緩衝區中，等當前傳輸完成的中斷會自動啟動下一輪。

### Q：TX 完成中斷裡做什麼？

```c
// 1. 跳過已發送的資料（移動環形緩衝區讀指標）
lwrb_skip(&usart_tx_rb, usart_tx_dma_current_len);

// 2. 標記 DMA 空閒
usart_tx_dma_current_len = 0;

// 3. 嘗試鏈式啟動下一輪
usart_start_tx_dma_transfer();
```

`lwrb_skip()` 只移動讀指標，不做任何資料拷貝——這是零拷貝的另一部分。

---

## 五、lwrb 環形緩衝區核心設計

### Q：lwrb 有什麼特別之處？

| 特性       | 說明                                                            |
| ---------- | --------------------------------------------------------------- |
| 輕量       | 標頭檔 + 原始碼檔各一個，無動態分配                             |
| 執行緒安全 | 讀寫指標用 `volatile` 修飾，單寫單讀場景無需鎖                  |
| 零拷貝介面 | `get_linear_block_read_address/length` 直接暴露內部位址給 DMA   |
| 魔術字校驗 | `magic1=0xDEADBEEF, magic2=~0xDEADBEEF`，偵測緩衝區是否已初始化 |
| 實際容量   | `size - 1`，滿條件 `w == r - 1`                                 |

### Q：為什麼緩衝區滿的條件是 `w == r - 1` 而不是 `w == r`？

`w == r` 表示「空」（讀寫指標重合）。如果滿也用 `w == r`，就無法區分空和滿。保留一個位元組不寫（`w` 比 `r` 少走一步），就能透過指標關係判斷狀態：

| 條件                  | 狀態         |
| --------------------- | ------------ |
| `w == r`              | 空           |
| `(w + 1) % size == r` | 滿           |
| 其他                  | 有資料但未滿 |

### Q：lwrb 的線性塊介面為什麼對 DMA 至關重要？

DMA 只能搬運**連續**的記憶體區域。環形緩衝區的資料在回繞時分為兩段不連續區域。線性塊介面返回**從讀指標到緩衝區末尾（或寫指標，取較小者）**的連續區域，DMA 可以直接使用這段位址：

```c
// 返回從讀指標開始的連續可讀長度
size_t lwrb_get_linear_block_read_length(lwrb_t *lwrb) {
    size_t len, w;
    w = lwrb->w;  // 一次性讀取寫指標
    if (w >= lwrb->r) {
        len = w - lwrb->r;  // 線性區域
    } else {
        len = lwrb->size - lwrb->r;  // 讀指標到緩衝區末尾
    }
    return len;
}
```

---

## 六、STM32 vs GD32：雙晶片實現對比

### Q：兩個平台的 DMA 架構有什麼區別？

| 維度       | STM32F407                                   | GD32F470                                                               |
| ---------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| DMA 模型   | **Stream + Channel** (DMA2_Stream2_CH4)     | **Channel + Subperi** (DMA1_CH2_SUB4)                                  |
| RX DMA     | DMA2_Stream2                                | DMA1_Channel2                                                          |
| TX DMA     | DMA2_Stream7                                | DMA1_Channel7                                                          |
| DMA 計數器 | `__HAL_DMA_GET_COUNTER()` 巨集              | `dma_transfer_number_get()` 函式                                       |
| 中斷管理   | HAL 統一回呼 (`HAL_UARTEx_RxEventCallback`) | 手動編寫 IRQ Handler                                                   |
| 抽象層級   | 厚封裝（HAL）                               | 薄封裝（標準外設庫）                                                   |
| GPIO 復用  | `HAL_GPIO_Init()` 一步完成                  | `gpio_af_set()` + `gpio_mode_set()` + `gpio_output_options_set()` 三步 |

### Q：中斷處理方式有什麼不同？

**STM32 HAL 方案**：

```c
// 中斷自動分發到 HAL 回呼
void USART1_IRQHandler(void) { HAL_UART_IRQHandler(&huart1); }
void DMA2_Stream2_IRQHandler(void) { HAL_DMA_IRQHandler(&hdma_usart1_rx); }
void DMA2_Stream7_IRQHandler(void) { HAL_DMA_IRQHandler(&hdma_usart1_tx); }

// 統一回呼
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size) {
    usart_rx_check();  // HT/TC/IDLE 統一入口
}
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart) {
    lwrb_skip(&usart_tx_rb, usart_tx_dma_current_len);
    usart_tx_dma_current_len = 0;
    usart_start_tx_dma_transfer();
}
```

**GD32 標準庫方案**：

```c
// 手動編寫每個中斷處理函式
void USART0_IRQHandler(void) {
    if (RESET != usart_interrupt_flag_get(USART0, USART_INT_FLAG_IDLE)) {
        usart_interrupt_flag_clear(USART0, USART_INT_FLAG_IDLE);
        usart_data_receive(USART0);  // 讀 DR 清除 IDLE
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

### Q：DMA 初始化的流程差異？

**STM32 HAL**（CubeMX 生成）：

```c
// RX DMA - 在 UART 初始化中自動關聯
hdma_usart1_rx.Instance = DMA2_Stream2;
hdma_usart1_rx.Init.Channel = DMA_CHANNEL_4;
hdma_usart1_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
hdma_usart1_rx.Init.Mode = DMA_CIRCULAR;
HAL_DMA_Init(&hdma_usart1_rx);
__HAL_LINKDMA(&huart1, hdmarx, hdma_usart1_rx);  // 關聯到 UART 控制代碼

// 啟動接收
HAL_UARTEx_ReceiveToIdle_DMA(&huart1, usart_rx_dma_buffer, 64);
__HAL_DMA_DISABLE_IT(&hdma_usart1_rx, DMA_IT_HT);  // 可選：關閉 HT 中斷
```

**GD32 標準庫**（手動配置）：

```c
dma_single_data_parameter_struct dma_init;
dma_init.periph_addr = (uint32_t)&USART_DATA(USART0);
dma_init.periph_inc = DMA_PERIPH_INCREASE_DISABLE;
dma_init.memory0_addr = (uint32_t)usart_rx_dma_buffer;
dma_init.memory_inc = DMA_MEMORY_INCREASE_ENABLE;
dma_init.periph_memory_width = DMA_PERIPH_WIDTH_8BIT;
dma_init.circular_mode = DMA_CIRCULAR_MODE_ENABLE;  // 關鍵！
dma_init.direction = DMA_PERIPH_TO_MEMORY;
dma_init.number = USART0_RX_DMA_LENGTH;
dma_init.priority = DMA_PRIORITY_MEDIUM;
dma_single_data_mode_init(USART0_DMA, USART0_RX_DMA_CH, &dma_init);
dma_channel_subperipheral_select(USART0_DMA, USART0_RX_DMA_CH, USART0_DMA_SUBPERI);
dma_channel_enable(USART0_DMA, USART0_RX_DMA_CH);
```

### Q：NVIC 中斷優先級有什麼講究？

所有相關中斷（USART、RX DMA、TX DMA）必須設為**相同優先級**，確保它們不會互相搶佔。這是因為 `usart_rx_check()` 不是可重入的——如果 RX DMA 的 TC 中斷打斷了 IDLE 中斷中的 `usart_rx_check()`，會導致 `old_pos` 狀態混亂。

| 平台  | 優先級設置                    |
| ----- | ----------------------------- |
| STM32 | 全部設為 `(0, 0)`             |
| GD32  | 全部設為 `(2, 2)` 或 `(2, 5)` |

> [!WARNING]
> 如果應用中還有其他更高優先級的中斷（如定時器），UART 相關中斷可以整體降低優先級，但三者之間必須相同。

---

## 七、常見問題與踩坑

### Q：為什麼 DMA 初始化必須在 UART 初始化之前？

DMA 時鐘必須先於 UART 時鐘使能。STM32 中 `MX_DMA_Init()` 必須在 `MX_USART1_UART_Init()` 之前呼叫，否則 HAL 內部關聯 DMA 控制代碼時會失敗。CubeMX 生成的程式碼預設遵循此順序，但手動調整時容易出錯。

### Q：IDLE 中斷標誌怎麼清除？

IDLE 標誌的清除方式比較特殊——需要**先讀狀態暫存器，再讀資料暫存器**：

| 平台      | 清除方式                                                |
| --------- | ------------------------------------------------------- |
| STM32 HAL | `__HAL_UART_CLEAR_IDLEFLAG()` (內部讀 SR + DR)          |
| GD32      | `usart_interrupt_flag_clear()` + `usart_data_receive()` |

如果清除不正確，IDLE 中斷會持續觸發，進入死循環。

### Q：`HAL_UARTEx_ReceiveToIdle_DMA()` 為什麼要每次回呼後重新呼叫？

STM32 HAL 的設計：雖然 RX DMA 配置為 Circular 模式不會停止，但 `ReceiveToIdle_DMA` 內部還管理了 IDLE 中斷的使能。回呼中如果不重新呼叫，IDLE 中斷可能不會再次觸發。HT 和 TC 中斷則由 DMA 硬體自動觸發，不需要重新使能。

GD32 方案不存在這個問題——IDLE 中斷在初始化時一次性使能，不需要每次重啟。

### Q：RX 緩衝區大小怎麼選？

| 緩衝區大小 | HT 觸發間隔 (115200bps) | 適用場景             |
| ---------- | ----------------------- | -------------------- |
| 32 位元組  | ~1.4 ms                 | 低速、短封包         |
| 64 位元組  | ~2.8 ms                 | 通用（推薦）         |
| 128 位元組 | ~5.6 ms                 | 高速、長封包         |
| 256 位元組 | ~11.1 ms                | 超高速或主頻低的 MCU |

原則：**緩衝區大小 ≥ 最大連續資料幀長度 × 2**。HT 和 TC 將緩衝區一分為二，各半必須能容納一幀資料。

### Q：printf 重導向為什麼用阻塞發送？

範例中的 `fputc()` 使用輪詢方式逐位元組發送，不用 DMA。原因是 printf 通常在除錯場景使用，呼叫時需要確保資料完整輸出。如果用 DMA 非同步發送，printf 返回時資料可能還沒發完。實際產品中建議用 `usart_send_string()` 代替 printf。

---

## 八、效能對比：DMA vs 中斷 vs 輪詢

### Q：三種方式的 CPU 開銷對比如何？

以 115200bps、168MHz 主頻為例，接收 1KB 資料：

| 方式 | CPU 時間        | 佔比 | 資料遺失風險      |
| ---- | --------------- | ---- | ----------------- |
| 輪詢 | ~5.6 ms (100%)  | 100% | 無（但 CPU 全佔） |
| 中斷 | ~0.5 ms (9%)    | ~9%  | 中斷延遲時可能    |
| DMA  | ~0.02 ms (0.4%) | < 1% | 幾乎為零          |

DMA 方式的 CPU 時間僅用於 `usart_rx_check()` 中的指標計算和資料搬運（寫入 TX 環形緩衝區），實際 DMA 傳輸過程 CPU 不參與。

### Q：DMA 方式的延遲有多大？

| 事件                               | 最大延遲                              |
| ---------------------------------- | ------------------------------------- |
| 收到資料到 HT/TC 中斷觸發          | 半緩衝區傳輸時間 (64B@115200 ≈ 2.8ms) |
| 收到資料到 IDLE 中斷觸發           | 1 字元時間 + IDLE 偵測時間 (≈ 0.1ms)  |
| TX 資料從寫入環形緩衝區到 DMA 啟動 | 當前 DMA 傳輸完成時間 + 臨界區時間    |

IDLE 中斷是延遲最低的觸發方式，適合對即時性要求高的場景。

---

## 九、移植指南：三步移植到你的平台

### Q：如何將此方案移植到其他 MCU？

**第一步：配置 RX DMA**

1. DMA 配置為 Circular 模式，外設到記憶體
2. 外設位址 = USART 資料暫存器位址
3. 記憶體位址 = `usart_rx_dma_buffer`
4. 使能 HT + TC 中斷
5. 使能 USART IDLE 中斷

**第二步：配置 TX DMA**

1. DMA 配置為 Normal 模式，記憶體到外設
2. 不在初始化時啟動（按需啟動）
3. 使能 TC 中斷

**第三步：實現三個核心函式**

1. `usart_rx_check()` — 讀 DMA 計數器，計算位置，處理資料
2. `usart_start_tx_dma_transfer()` — 臨界區保護，零拷貝啟動
3. TX 完成中斷處理 — `lwrb_skip()` + 鏈式啟動

---

## 十、環形緩衝區 vs 乒乓緩衝區：應用範圍考量

### Q：什麼是乒乓緩衝區（Ping-Pong Buffer）？

乒乓緩衝區使用**兩個等大的線性緩衝區**交替工作：DMA 寫入 A 緩衝區時，CPU 處理 B 緩衝區；DMA 半傳輸完成時交換角色。本質上就是 DMA Circular + HT/TC 中斷的「半緩衝區」機制。

```
緩衝區 A [0, N/2)   ← HT 中斷時 CPU 處理 A，DMA 繼續寫 B
緩衝區 B [N/2, N)   ← TC 中斷時 CPU 處理 B，DMA 繼續寫 A（回繞）
```

### Q：環形緩衝區和乒乓緩衝區的核心區別是什麼？

| 維度       | 環形緩衝區 (lwrb)            | 乒乓緩衝區                         |
| ---------- | ---------------------------- | ---------------------------------- |
| 資料結構   | 一段連續記憶體 + 讀寫指標    | 兩個等大線性緩衝區                 |
| 處理粒度   | 任意長度（按實際寫入量處理） | 固定半緩衝區大小                   |
| 資料延遲   | IDLE 中斷觸發後立即處理      | 必須等 HT/TC 才能處理              |
| 不定長幀   | 天然支援（IDLE 可觸發）      | 需要額外機制處理不足半緩衝區的尾部 |
| 記憶體利用 | 實際容量 `size - 1`          | 兩個緩衝區各 `N/2`，必須等滿才處理 |
| 實現複雜度 | 需要處理回繞、線性塊         | 邏輯簡單，HT/TC 直接切換           |

### Q：各適合什麼場景？

**環形緩衝區更適合**：

- **不定長協定幀**：Modbus RTU、自訂命令協定，幀長度不固定，IDLE 中斷可立即偵測幀結束
- **低延遲互動**：人機互動、AT 指令回應，IDLE 中斷保證幀尾不卡
- **TX 非同步發送**：零拷貝 + 鏈式續傳，應用層隨時寫入，DMA 自動排程

**乒乓緩衝區更適合**：

- **定長取樣流**：ADC/I2S 音訊取樣、感測器高速等間隔資料流，每幀固定長度
- **批次處理管線**：FFT、數位濾波等需要整塊資料才能運算的場景
- **雙核共享**：一個核寫 A，另一個核處理 B，天然無鎖交替

### Q：UART 場景為什麼首選環形緩衝區？

UART 通信的典型特徵：

1. **資料幀長度不固定**：一條 AT 命令可能 10 位元組，下一條可能 200 位元組
2. **幀間隔不確定**：兩次通信之間可能有 ms 級空閒（IDLE 可偵測）
3. **需要低延遲回應**：收到命令後應盡快回覆，不能等半緩衝區填滿

乒乓緩衝區的 HT/TC 觸發機制要求「半緩衝區填滿才處理」，如果幀長不足半緩衝區，資料就會一直「卡」在裡面。雖然可以加 IDLE 中斷來補救，但本質上已經退化成環形緩衝區的處理邏輯了——不如一開始就用環形緩衝區。

> [!TIP]
> 如果你的 UART 場景是**定長高速資料流**（如連續感測器取樣），乒乓緩衝區實現更簡單；如果是**不定長命令互動**（絕大多數 UART 場景），環形緩衝區是更自然的選擇。

---

## 十一、總結

### 核心設計要點

| 要點                                | 原因                                              |
| ----------------------------------- | ------------------------------------------------- |
| RX DMA Circular                     | 永不停止，無重啟窗口期，不遺失資料                |
| TX DMA Normal                       | 每次發一段線性資料，鏈式續傳                      |
| 三重中斷 (HT+TC+IDLE)               | HT/TC 保證高速連續流不丟，IDLE 保證不定長包尾不卡 |
| DMA 計數器反推位置                  | 零額外硬體，純軟體計算當前寫入位置                |
| TX 零拷貝                           | DMA 直接從環形緩衝區線性塊讀取，無中間緩衝區      |
| `usart_tx_dma_current_len` 互斥標誌 | 避免應用層和中斷層同時操作 DMA                    |
| 同優先級中斷                        | 防止 `usart_rx_check()` 被搶佔導致狀態混亂        |

### 雙晶片驗證結論

STM32 HAL 和 GD32 標準外設庫在**架構層面完全一致**——RX Circular + TX Normal + lwrb 零拷貝 + 三重中斷。差異僅在 API 呼叫方式：

- **HAL 厚封裝**：回呼機制統一管理中斷，開發快但除錯時呼叫堆疊深
- **標準庫薄封裝**：手動管理中斷標誌，程式碼透明度高但細節多

選擇哪種取決於專案需求——快速原型選 HAL，深度除錯選標準庫。

---

## 參考資料

- [lwrb — Lightweight ring buffer library](https://github.com/MaJerle/lwrb)
- [stm32-usart-uart-dma-rx-tx — UART DMA RX/TX example](https://github.com/MaJerle/stm32-usart-uart-dma-rx-tx)
- [STM32F407VGT6 UART DMA Efficient RingBuffer — 本文 STM32 工程](https://github.com/Lingjia007/STM32F407VGT6_UART_DMA_Efficient_RingBuffer)
- [GD32F470ZGT6 UART DMA Efficient RingBuffer — 本文 GD32 工程](https://github.com/Lingjia007/GD32F470ZGT6_UART_DMA_Efficient_RingBuffer)
- [STM32F407 Reference Manual (RM0090)](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [GD32F4xx User Manual](https://www.gigadevice.com.cn/)
- [C語言的精髓：結構體與指標](../oop-in-c-embedded/)
- [STM32F407 安全 Bootloader 設計](../stm32f407-secure-bootloader/)
