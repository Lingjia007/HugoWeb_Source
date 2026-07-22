---
title: "STM32F407 流式語音關鍵詞檢測：全鏈路設計與五大踩坑解析"
date: 2026-07-17
description: "在 STM32F407VGT6 上跑通 micro_speech 流式關鍵詞檢測的完整實錄：INMP441 I2S MEMS 麥克風、DMA ping-pong、環形緩衝、MFCC 滑動窗口、int8 量化推理、KWS 智能合併後處理。詳解 stride 幀佈局陷阱、volatile 可見性、棧餘量優化、KWS 誤報、隱藏依賴等五大坑，以及最終的 audio/capture+frontend+kws 模組化分層架構"
image: STM32F407-TFLM-流式语音识别.png
categories:
  - "嵌入式"
  - "邊緣AI"
tags:
  - "STM32"
  - "TensorFlow-Lite-Micro"
  - "micro_speech"
  - "INMP441"
  - "I2S"
  - "DMA"
  - "關鍵詞檢測"
  - "KWS"
  - "volatile"
  - "模組化"
math: true
---

## 前言

這個工程的目標：在 STM32F407VGT6 上用 INMP441 I2S MEMS 麥克風跑通 micro_speech 模型的**流式**關鍵詞檢測——用戶隨時說話，系統即時識別 yes/no/up/down/...，不需要提前錄音再整段餵給模型。

參考工程是 ST 官方的 [STM32N6-GettingStarted-Audio](file:///d:/edgedownload/STM32N6-GettingStarted-Audio)，它已經實現了流式採集 + 推理的完整鏈路。但把它移植到 STM32F407 + INMP441 時，遇到了五個坑，每個症狀都不一樣：

| #   | 坑            | 症狀                           | 根因                                         |
| --- | ------------- | ------------------------------ | -------------------------------------------- |
| 1   | stride 錯誤   | 識別結果隨機橫跳，score 14~30  | STM32F4 I2S 16B_EXTENDED 幀佈局與假設不符    |
| 2   | volatile 缺失 | 刪掉一行 printf 系統就無輸出   | ISR/main 共享變數被 `-O3 -flto` 快取到暫存器 |
| 3   | 棧餘量不足    | 預防性優化（未觀察到明確症狀） | 大陣列放放棧上，預設 2KB 棧餘量危險            |
| 4   | KWS 誤報      | 噪音被識別成 up，連續觸發      | int8 量化分數區分度低 + per-word 冷卻被重置  |
| 5   | 隱藏依賴      | 編譯報錯找不到 array.h         | kernel_util.cpp 依賴上游 TFL 的 array.cc     |

本文先按資料流自上而下講架構，再按踩坑時間線講除錯。每個坑附**症狀 → 根因 → 修復**三段式分析，希望遇到類似問題的人能直接對號入座。

> [!NOTE]
> 本文不是保姆級教程，假設讀者已熟悉 STM32 HAL、I2S、DMA、TFLite 基礎。重點在**踩坑與架構**，不是從零搭建。模型訓練部分見前作 [WSL2 + Docker + TensorFlow 教程](../wsl2-docker-tensorflow/)。

---

## 一、工程總覽

### 1.1 硬體配置

| 部件     | 型號 / 配置                                                                  |
| -------- | ---------------------------------------------------------------------------- |
| 主控     | STM32F407VGT6（Cortex-M4F，168 MHz，1 MB Flash，192 KB RAM 含 64 KB CCMRAM） |
| 麥克風   | INMP441 I2S MEMS（24-bit，L/R 引腳接地 = 左通道）                            |
| I2S 介面 | I2S2，Philips 標準，16B_EXTENDED，16 kHz 立體聲                              |
| DMA      | DMA1_Stream3 / Channel0，Circular，半字對齊                                  |
| 除錯器   | J-Link                                                                       |
| 模型     | micro_speech int8 量化（36 標籤）                                            |
| 推理框架 | TensorFlow-Lite-Micro + X-CUBE-AI 整合                                       |

### 1.2 軟體資料流

整個流式 KWS 鏈路分成五層，從麥克風到關鍵詞輸出：

```
[INMP441]
   │ I2S2 24-bit @ 16 kHz
   ▼

audio/capture/                       ← 第 1 層：I2S DMA ping-pong
  • DMA 雙緩衝 (640 halfwords)
  • Half/Full IRQ 各 10 ms
  • 左通道去交錯 [L,R] → [L]
  • 按 block 減去均值去 DC
   ▼ feed (160 samples / 10 ms)

audio/capture/                       ← 第 2 層：環形緩衝
  AudioCaptureRingBuff_t
  4 × 3200 = 12800 samples（25.6 KB，存 CCMRAM）
  LDREX/STREX 原子計數器
  volatile 保證 ISR/main 可見
   ▼ consume (3200 samples / 200 ms)

audio/frontend/                      ← 第 3 層：MFCC 滑動窗口
  • 1 秒窗口 (16000 samples)
  • SLIDE_COLS=10 (200 ms)
  • CMSIS-DSP RFFT Q15
  • 40 通道 Mel 濾波器組
  • 存放在 CCMRAM（連同環形緩衝）
   ▼ 49×40 int8 特徵

X-CUBE-AI                            ← 第 4 層：int8 推理
  micro_speech 模型 (~50 ms)
   ▼ 36 個 int8 logits

audio/kws/                           ← 第 5 層：KWS 後處理
  • 過濾 _silence_/_unknown_
  • N-in-M 消抖 (3/5)
  • 全域冷卻 (5 幀 ≈ 1 s)
  • score 閾值 (>= 20)
  • softmax 機率顯示
   ▼

[KWS N] yes (100.0%, score=83)
```

每一層都是一個獨立的 `.c/.h` 模組，最後透過 `X-CUBE-AI/App/app_x-cube-ai.c` 這個 CubeMX 模板檔案串起來。

> [!TIP]
> 為什麼要分層？因為流式 KWS 的 bug 80% 出在「層與層的介面」上：DMA 幀佈局錯了、環形緩衝可見性錯了、滑動窗口步長錯了……分層之後每層都能獨立驗證，定位 bug 時按層排查即可。

---

## 二、第 1 層：I2S DMA ping-pong 採集

### 2.1 為什麼用 16B_EXTENDED 而不是 24B/32B？

INMP441 是 24-bit MEMS 麥克風，按理說應該用 24 位或 32 位 I2S 格式才「對味」。但實際上，micro_speech 模型只需要 16-bit 輸入，而 STM32F4 的 I2S 24B/32B 模式會讓 DMA 傳輸 4 個半字（每通道 2 個），多搬一半資料，浪費頻寬和 CPU 去交錯時間。

`16B_EXTENDED` 模式則是一個折中：**32-bit 時隙 + 16-bit 資料**，每個通道只搬 1 個半字，DMA 拿到的就是已經對齊的 16-bit 樣本。

> [!WARNING]
> 這就是第一個大坑的發源地——STM32N6 原工程**根本不用 I2S**（用的是 MDF 數位麥克風介面 + BSP 單聲道抽象），所以沒有去交錯邏輯可供參考。在 STM32F4 上新寫去交錯程式碼時，對 `16B_EXTENDED` 幀佈局的誤解導致 stride=4 錯誤。詳見 [5.1 節](#51-stride-bug16b_extended-幀佈局陷阱)。

### 2.2 DMA 雙緩衝與回呼

```c
/* DMA 雙緩衝：必須放在普通 RAM，DMA 訪問不到 CCMRAM */
static uint16_t s_dma_buf[AS_DMA_BUF_HALFWORDS];  /* 640 halfwords */

/* 啟動循環模式 DMA：會觸發 Half / Full 兩個回呼 */
HAL_I2S_Receive_DMA(&hi2s2, (uint16_t *)s_dma_buf, AS_DMA_BUF_HALFWORDS);
```

`AS_DMA_HALF_FRAMES = 160`，意味著每個半緩衝裝 10 ms 的單聲道樣本（16000 Hz × 10 ms = 160）。`HAL_I2S_RxHalfCpltCallback` 在 DMA 寫到一半時觸發，`HAL_I2S_RxCpltCallback` 在寫完整圈時觸發，兩個回呼分別處理 `s_dma_buf` 的前半段和後半段，形成無縫的 ping-pong。

### 2.3 去交錯與 DC 去除

```c
static void AudioStream_process_half(uint8_t half_id)
{
  const uint16_t *src = s_dma_buf + half_id * AS_DMA_HALFWORDS;

  /* STM32F4 16B_EXTENDED: 每幀 2 halfwords [L, R]
   * INMP441 L/R=GND → 左通道，AUDIO_CHANNEL_INDEX=0 */
  int32_t sum = 0;
  for (uint32_t i = 0; i < AS_DMA_HALF_FRAMES; i++)
  {
    int16_t v = (int16_t)src[i * 2 + AUDIO_CHANNEL_INDEX];
    s_mono[i] = v;
    sum += v;
  }

  /* 按 block 減均值去 DC：MEMS 麥克風有幾十~幾百的 DC 偏置 */
  int16_t dc = (int16_t)(sum / (int32_t)AS_DMA_HALF_FRAMES);
  for (uint32_t i = 0; i < AS_DMA_HALF_FRAMES; i++)
    s_mono[i] = (int16_t)(s_mono[i] - dc);

  AudioCaptureRingBuff_feed(&s_ring, (const uint8_t *)s_mono,
                            (uint16_t)AS_DMA_HALF_FRAMES);
}
```

**為什麼要去 DC？** INMP441 上電後輸出會有一個固定的 DC 偏置（實測幾十到幾百），如果不去掉，下游 MFCC 的能量頻譜會整體抬高，模型量化後的 int8 特徵會偏移。每 10 ms 一個 block 減去均值，既簡單又穩定，比一階 IIR 高通濾波器便宜得多。

---

## 三、第 2 層：環形緩衝（lock-free）

### 3.1 為什麼要環形緩衝？

DMA 中斷每 10 ms 產生 160 個樣本，但消費端（MFCC）每 200 ms 才需要 3200 個樣本。生產者和消費者速率嚴重不匹配，需要一個 FIFO 解耦。

| 通路                 | 頻率   | 速率            |
| -------------------- | ------ | --------------- |
| Producer (DMA IRQ)   | 100 Hz | 160 samples/次  |
| Consumer (main loop) | 5 Hz   | 3200 samples/次 |

### 3.2 原子計數器與可見性

環形緩衝的核心欄位是 `availableSamples`，被 ISR 寫、main 循環讀。這兩個上下文是真正的併發，必須解決兩個問題：

1. **原子性**：32 位讀寫本身在 Cortex-M4 上是原子的，但「讀-改-寫」（`+=`）不是。用 `LDREX/STREX` 獨佔訪問解決：

   ```c
   static inline void atomic_add(volatile uint32_t *p32, int32_t inc)
   {
     do { } while (__STREXW(__LDREXW(p32) + inc, p32));
   }
   ```

2. **可見性**：這是第二個大坑（詳見 [5.2 節](#52-volatile-缺失atomic--visibility)）。簡而言之，必須加 `volatile`：

   ```c
   typedef struct {
     uint8_t  *pData;
     ...
     volatile uint32_t availableSamples;  /* ISR/main 共享，必須 volatile */
   } AudioCaptureRingBuff_t;
   ```

### 3.3 消費端的簡易臨界區

```c
uint8_t *AudioCaptureRingBuff_consume(uint8_t *pData,
                                      AudioCaptureRingBuff_t *pHdle,
                                      uint32_t nbSamples)
{
  /* 消費端在 main 循環，brief IRQ masking 保持索引一致 */
  __disable_irq();
  if (pHdle->availableSamples >= (int32_t)nbSamples) {
    /* ... 兩段拷貝處理回繞 ... */
    atomic_add(&pHdle->availableSamples, -(int32_t)nbSamples);
  }
  __enable_irq();
  return pData;
}
```

生產端不能用 `__disable_irq()`（會抖動 DMA 時序），所以用 LDREX/STREX；消費端在 main 循環裡，關中斷幾微秒無所謂，直接關 IRQ 更簡單。

> [!NOTE]
> 這種「生產端原子、消費端關中斷」的不對稱設計，是 bare-metal 環形緩衝的經典模式。如果換成 FreeRTOS，應該用 `xStreamBuffer` 或帶互斥鎖的佇列。

---

## 四、第 3 層：MFCC 滑動窗口

### 4.1 關鍵參數

```c
#define SAMPLE_RATE          16000     // 採樣率
#define FRAME_LEN            480       // 30 ms 窗長 = 480 samples
#define WINDOW_STRIDE        320       // 20 ms 步長 = 320 samples
#define FFT_LENGTH           512       // 下一個 2^n >= 480
#define NUM_CHANNELS         40        // Mel 濾波器組通道數
#define SPECTROGRAM_LENGTH   49        // (16000-480)/320 + 1

#define SLIDE_COLS           10        // 每次滑動 10 個頻譜列
#define SLIDE_SAMPLES        3200      // = 10 × 320 = 200 ms
```

### 4.2 滑動窗口機制

micro_speech 模型的輸入是 1 秒（16000 樣本）的頻譜圖，形狀為 `49 × 40`。如果每次都從頭算 1 秒窗口的 MFCC，CPU 開銷太大。

**滑動窗口策略**：維護一個 1 秒的環形音頻窗口，每來 200 ms 新音頻（3200 樣本），窗口滑動 10 個頻譜列，只計算這 10 個新列，老的 39 列復用。這樣每次推理前的預處理開銷從 ~200 ms 降到 ~30 ms。

```
窗口狀態 (49 列頻譜圖):
┌──────────────────────────────────────────────────────┐
│ [老 39 列]                              │ [新 10 列] │
└──────────────────────────────────────────────────────┘
                                                  ▲
                                  200 ms 後：      │
                                                  ▼
┌──────────────────────────────────────────────────────┐
│          │ [老 39 列]                    │ [新 10 列] │
└──────────────────────────────────────────────────────┘
   ← 丟棄
```

### 4.3 CCMRAM：把 1 秒窗口塞進去

`AudioFrontendStream` 結構體裡包含 16000 個 `int16_t` 的音頻窗口（32 KB）加上頻譜快取，總共 ~37 KB。STM32F407 主 RAM 只有 128 KB（含棧和堆），而 CCMRAM 有 64 KB 專門給資料用，DMA 又訪問不到 CCMRAM。

所以策略是：**DMA 緩衝放主 RAM，前端窗口放 CCMRAM**。

```c
/* 1 秒窗口 + 特徵快取放 CCMRAM，主 RAM 留給棧/堆/模型啟用 */
static AudioFrontendStream g_stream __attribute__((section(".ccmram")));
```

最終記憶體佔用：

```
RAM:    82320 B / 112 KB  (71.78%)
CCMRAM: 37544 B / 64 KB   (57.29%)
FLASH:  348684 B / 1 MB   (33.25%)
```

> [!TIP]
> CCMRAM (Core Coupled Memory) 是 Cortex-M4 的一塊 0 等待週期 SRAM，訪問速度比主 RAM 快得多，特別適合放頻繁訪問的大陣列。但代價是 DMA 不能用，所以選放什麼進去很有講究。

---

## 五、踩坑實錄（重頭戲）

下面五個坑，每一個都讓我花了至少一小時定位。按踩中順序排列。

### 5.1 stride bug：16B_EXTENDED 幀佈局陷阱

#### 症狀

流式推理識別結果全錯，對著麥克風說 "yes" 識別成 "up"，說 "one" 識別成 "cat"，score 只有 14~30。但用同一份模型直接餵整段錄音測試，識別完全正常。

#### 根因

程式碼是從 STM32N6 工程移植過來的。STM32N6 的 I2S 在 16B_EXTENDED 模式下，每個立體聲幀傳輸 **4 個半字** `[L_pad, L, R_pad, R]`（24-bit 槽位 + padding）。而 STM32F4 的 16B_EXTENDED 是 **2 個半字** `[L, R]`（純 16-bit 資料，無 padding）。

我移植時沒改 stride，DMA 去交錯程式碼寫成：

```c
/* ❌ 錯誤：STM32N6 的 stride=4，在 STM32F4 上等效降取樣 */
int16_t v = (int16_t)src[i * 4 + 2];  /* 取 R 通道 */
```

在 STM32F4 上 `src[i*4+2]` 實際跨越了 2 個立體聲幀，等於**只取了一半的左通道樣本**。等效於把 16 kHz 採樣率降到了 8 kHz，所有頻率成分都錯位了一半。模型看到 "yes" 的頻譜圖實際是 "yes" 降取樣後的扭曲版本，自然識別錯。

#### 修復

```c
/* ✅ 正確：STM32F4 16B_EXTENDED 每幀 2 halfwords [L, R] */
#define AS_DMA_HALFWORDS (AS_DMA_HALF_FRAMES * 2)   /* 320, 不是 640 */

int16_t v = (int16_t)src[i * 2 + AUDIO_CHANNEL_INDEX];  /* stride=2 */
```

修復後 score 從 14~30 飆升到 **42~100**，"yes/no/up/down" 全部正確識別。

> [!WARNING]
> 這個 stride bug 的真正根源不是「配置不同」，而是 **STM32N6 和 STM32F4 用了完全不同的音頻外設架構**——N6 原工程根本不用 I2S！移植時資料獲取層必須重寫，不能照搬。

**STM32N6 原工程 vs STM32F407 本工程的音頻路徑對比：**

| 維度                | STM32N6（原工程）                                        | STM32F407（本工程）                        |
| ------------------- | -------------------------------------------------------- | ------------------------------------------ |
| 音頻外設            | **MDF**（Multi-function Digital Filter，數位麥克風介面） | **I2S2**（SPI2 復用）                      |
| 驅動抽象            | `BSP_AUDIO_IN_*` 高層 API                                | 裸 `HAL_I2S_*`                             |
| 通道配置            | `ChannelsNbr = 1`（單聲道）                              | I2S 協議**強制立體聲**（WS 在 L/R 間切換） |
| DMA buffer 內容     | 純 mono int16 序列                                       | `[L, R, L, R, ...]` 立體聲交錯             |
| 16B_EXTENDED 幀佈局 | 不適用（MDF 不走 I2S 幀格式）                            | 每幀 2 halfwords `[L, R]`                  |
| 去交錯程式碼        | **不需要**（BSP 已配 mono，直接餵 ring）                 | **必須新寫**（從 `[L,R]` 提取 L）          |

N6 工程的初始化程式碼長這樣：

```c
AudioInit.Device        = AUDIO_IN_DEVICE_DIGITAL_MIC;  /* MDF 接數位麥克風 */
AudioInit.BitsPerSample = AUDIO_RESOLUTION_16B;
AudioInit.ChannelsNbr   = 1;                            /* 單聲道，關鍵！ */
BSP_AUDIO_IN_Init(1, &AudioInit);
BSP_AUDIO_IN_Record(1, (uint8_t *)acq_buf, CAPTURE_BUFFER_SIZE * sizeof(int16_t));
```

DMA 完成中斷裡直接 `AudioCapture_ring_buff_feed(acq_buf, ring, len)`，**中間沒有任何 stride 處理**——因為 buffer 本來就是單聲道。

而 STM32F4 這邊，I2S 協議物理上就是立體聲（WS=LOW 傳左、WS=HIGH 傳右），INMP441 即使只有一個麥克風也輸出立體聲流，必須按 `[L,R]` 接收再丟棄一半。**N6 工程沒有這段邏輯可供參考**，移植時新寫的去交錯程式碼就因為對 `16B_EXTENDED` 幀佈局的誤解踩了 stride=4 的坑。

> [!TIP]
> 移植經驗：跨晶片移植音頻程式碼時，**先確認兩邊的音頻外設類型**（I2S / SAI / MDF / SPDIFRX），而不是直接看採樣率和位寬。外設類型不同意味著 DMA buffer 佈局可能完全不同，去交錯邏輯必須基於目標晶片的參考手冊重寫。可以用邏輯分析儀抓 WS + SD 時序，直接數一個 WS 週期內有幾個 16-bit slot，比看手冊更快。

### 5.2 volatile 缺失：atomic ≠ visibility

#### 症狀

修復 stride bug 後識別正常。我覺得程式碼穩定了，把診斷用的 `printf` 全刪掉，結果——**系統再無任何輸出**，只剩啟動橫幅。

更詭異的是，加回一行無關的 `printf`（每 2 秒列印一次 ring 可用樣本數），系統又恢復正常。刪掉，又死。這個 `printf` 跟識別邏輯毫無關係，但它「修復」了 bug。

#### 根因

`availableSamples` 欄位缺 `volatile`：

```c
/* ❌ 錯誤：缺 volatile */
typedef struct {
  ...
  uint32_t availableSamples;
} AudioCaptureRingBuff_t;
```

在 `-O3 -flto` 優化下，編譯器把 main 循環裡的 `while (ring->availableSamples < SLIDE_SAMPLES) break;` 內聯後，**把 `availableSamples` 快取到了暫存器**，再也不會從記憶體重讀。ISR 寫入的新值 main 循環永遠看不到，於是 `availableSamples` 在 main 看來永遠是 0，循環一直 break，推理永遠不執行。

那為什麼加 `printf` 又「復活」了？因為 `printf` 是函式呼叫，形成**編譯屏障（compiler barrier）**，強制編譯器在呼叫前把暫存器寫回記憶體、呼叫後從記憶體重讀。所以 `printf` 陰差陽錯地繞過了可見性問題。

#### 修復

```c
/* ✅ 正確：ISR/main 共享變數必須 volatile */
typedef struct {
  ...
  volatile uint32_t availableSamples;
} AudioCaptureRingBuff_t;

/* atomic_add 參數也要同步改 */
static inline void atomic_add(volatile uint32_t *p32, int32_t inc)
{
  do { } while (__STREXW(__LDREXW(p32) + inc, p32));
}
```

> [!IMPORTANT]
> 這是嵌入式 C 最經典的坑之一：**LDREX/STREX 只保證原子性，不保證可見性**。原子操作解決的是「讀-改-寫不可分割」，可見性解決的是「修改對其他上下文可見」。兩者是正交的概念，缺一不可。
>
> 在 Linux 核心裡 `atomic_t` 內部封裝了 memory barrier，所以用戶不用關心。但 bare-metal 上自己寫原子操作，**volatile 仍然必須加**。

### 5.3 預防性棧優化：大陣列改 static + 棧擴容

#### 背景

volatile 修復後系統能跑了。但回頭看 `audio_frontend_process_frame()` 的程式碼，發現裡面聲明了幾個大陣列作為區域變數：

```c
void audio_frontend_process_frame(...)
{
  int16_t window[FRAME_LEN];           /* 480 × 2 = 960 B */
  int16_t fft_input[FFT_LENGTH];       /* 512 × 2 = 1024 B */
  int16_t fft_output[FFT_LENGTH * 2];  /* 2048 B */
  q15_t mel_output[NUM_CHANNELS];      /* 80 B */
  /* ... 還有幾個 ... */
}
```

加起來每幀呼叫 ~5 KB 棧。STM32F4 預設棧只有 2 KB，雖然沒觀察到明確的棧溢位症狀（系統無輸出是 volatile 缺失導致的，不是棧），但這個餘量太危險——中斷巢套幾層就可能爆。作為預防性優化，做了兩處修改。

#### 修復

1. **大陣列改 `static`**：移到 BSS 段，函式不可重入但本來就是單執行緒呼叫，沒問題。

   ```c
   void audio_frontend_process_frame(...)
   {
     static int16_t window[FRAME_LEN];
     static int16_t fft_input[FFT_LENGTH];
     static int16_t fft_output[FFT_LENGTH * 2];
     /* ... */
   }
   ```

2. **棧大小從 0x800 提到 0x2000**：在 `STM32F407XX_FLASH.ld` 裡：

   ```
   _Min_Stack_Size = 0x2000 ;  /* 8 KB，原來 0x800 = 2 KB */
   ```

> [!TIP]
> 嵌入式 C 經驗法則：**任何超過 256 位元組的區域陣列都應該改 `static`**。Cortex-M4 的棧本來就小，函式巢套幾層就緊張。`static` 的代價是失去可重入性，但 90% 的場景下函式根本不會被併發呼叫。

### 5.4 KWS 誤報：up 的 int8 score 天生偏低

#### 症狀

![電腦風扇噪聲被誤識別為 up](测试结果.png)

上圖是系統穩定執行後的串列輸出範例。即使沒有說話，電腦風扇或環境噪聲也會觸發誤識別（這裡是 `up`），score 在 15~18 之間。

環境噪音被識別成 `up`，score 15-18，連續多次觸發。但真說 "up" 時 score 也只有 22-23，和噪音誤報重疊，單純提高閾值沒法區分。

更噁心的是，第一版 KWS 後處理裡 cooldown 是 per-word 的，結果中間穿插的 `wow`、`follow` 誤識別會重置 `up` 的冷卻計數器，讓 `up` 連續觸發 9 次。

#### 根因

1. **模型本身問題**：micro_speech 的 int8 量化讓 `up` 這個詞的 logit 天生偏低（22-23），與噪音誤報（15-18）只差 5。這種重疊是模型量化損失，軟體層面無解。

2. **後處理設計缺陷**：per-word cooldown 假設不同詞之間獨立，但實際上一個長音節可能跨越多個滑動窗口，被模型分段識別成不同詞（"foll" → follow，"ow" → wow），這些中間詞會重置目標詞的冷卻。

#### 修復

KWS 後處理四件套：

```c
#define KWS_HISTORY_M   5    /* 回看最近 5 幀推理（≈ 1 s）*/
#define KWS_REQUIRED_N  3    /* 同一標籤在窗口內出現 >= 3 次才確認 */
#define KWS_COOLDOWN    5    /* 任意觸發後全域抑制 5 幀（≈ 1 s）*/
#define KWS_MIN_SCORE   20   /* 低於 20 直接丟棄（噪音通常 15~18）*/

int kws_postprocess_feed(kws_state_t *s,
                         const int8_t *logits,
                         uint8_t num_labels,
                         const char *const *label_names)
{
  /* 1) argmax */
  int max_idx = argmax(logits, num_labels);

  /* 2) 丟棄 _silence_/_unknown_（標籤 0/1）*/
  if (max_idx <= 1) return 0;

  /* 3) 記錄歷史 */
  s->hist[s->hist_idx] = (int8_t)max_idx;
  s->hist_idx = (s->hist_idx + 1) % KWS_HISTORY_M;

  /* 4) 全域冷卻（不是 per-word！）*/
  if (s->cooldown_remaining > 0) {
    s->cooldown_remaining--;
    return 0;
  }

  /* 5) score 閾值 */
  if (max_val < KWS_MIN_SCORE) return 0;

  /* 6) N-in-M 消抖 */
  uint8_t hits = count_hits(s->hist, max_idx);
  if (hits < KWS_REQUIRED_N) return 0;

  /* 7) 觸發：softmax 列印 + 設全域冷卻 + 清空歷史 */
  print_softmax(logits, num_labels, label_names, max_idx);
  s->cooldown_remaining = KWS_COOLDOWN;
  for (int i = 0; i < KWS_HISTORY_M; i++) s->hist[i] = -1;
  return 1;
}
```

關鍵改進：

- **全域冷卻** 替代 per-word 冷卻：任意關鍵詞觸發後，**所有**關鍵詞都被抑制 1 秒，杜絕一個詞連續刷屏。
- **N-in-M 消抖**：要求 1 秒內同一標籤出現至少 3 次，過濾掉單幀的瞬時誤識別。
- **過濾 silence/unknown**：這兩個標籤會刷屏（噪音環境下 score 50+），對用戶沒用，直接丟棄。

> [!NOTE]
> 模型本身的 score 重疊問題，軟體沒法完全解決。如果想根治，要麼換更大的模型，要麼做 fine-tune 時加更多 `up` 的負樣本（噪音）。本文方案是把誤報頻率從「每秒 5 次」降到「每幾秒一次」，工程上可接受。

### 5.5 kernel_util.cpp 的隱藏依賴

#### 症狀

連結時報 `undefined reference to tflite::...` 一堆錯誤，全是 `kernel_util.cpp` 裡的符號。

#### 根因

TFLite Micro 的 `kernel_util.cpp` 依賴 upstream TensorFlow Lite 的 `array.h` 和 `array.cc`，這兩個檔案在 TFLM 的 `tensorflow/lite/` 目錄下，但預設不會被 TFLM 的 CMakeLists 收集進來。

#### 修復

在 CMakeLists.txt 顯式加上：

```cmake
set(TFLM_SOURCES
    ...
    # TFLite core (kernel_util.cpp 的隱藏依賴)
    ${TFLM_BASE}/tensorflow/lite/array.cc
    ${TFLM_BASE}/tensorflow/lite/core/c/common.cpp
    ...
    ${TFLM_BASE}/tensorflow/lite/kernels/kernel_util.cpp
    ...
)
```

> [!TIP]
> TFLM 的官方 CMake 模板不完整，自己拼裝工程時遇到 `undefined reference` 先去 `tensorflow/lite/` 找同名 `.cc` 檔案。它的目錄結構是 TFLM (`tensorflow/lite/micro/`) 呼叫 TFLite core (`tensorflow/lite/`)，後者經常漏檔案。

---

## 六、第 4+5 層：推理與 KWS 後處理

### 6.1 推理入口

`X-CUBE-AI` 生成的 `app_x-cube-ai.c` 是 CubeMX 模板檔案，**不能移動位置**（移動會破壞 CubeMX 重新生成能力）。所以這個檔案留在 `X-CUBE-AI/App/`，但裡面的 KWS 後處理邏輯被拆到了獨立的 `audio/kws/` 模組。

```c
void MX_X_CUBE_AI_Process(void)
{
  int res;
  do {
    /* 1) 從環形緩衝取 200 ms 資料 + 滑動窗口 + MFCC */
    res = acquire_and_process_data(data_ins);
    if (res == -2) break;  /* 資料不夠或窗口未填滿 */

    /* 2) 推理 */
    if (res == 0)
      res = ai_run();

    /* 3) KWS 後處理（debounce + cooldown + softmax 列印）*/
    if (res == 0)
      res = post_process(data_outs);
  } while (res == 0);
}
```

### 6.2 post_process 的最終形態

經過模組化重構後，`post_process` 從 75 行縮成了 1 行呼叫：

```c
int post_process(ai_i8 *data[])
{
  const int8_t *out = (const int8_t *)data[0];
  g_inference_cnt++;

  /* KWS 邏輯全部封裝在 audio/kws/kws_postprocess.c */
  (void)kws_postprocess_feed(&g_kws, out, NUM_LABELS, kLabelNames);
  return 0;
}
```

---

## 七、模組化重構：audio/ 三層目錄

踩完坑之後，程式碼散落在 `audio_app/`、`preprocess/`、`app_x-cube-ai.c` 三個地方，KWS 邏輯和模板程式碼混在一起。重構後的目錄結構：

```
audio/
├── capture/                       ← 第 1+2 層
│   ├── audio_capture_ring_buff.c  │  環形緩衝（lock-free, volatile）
│   ├── audio_capture_ring_buff.h  │
│   ├── audio_stream.c             │  I2S DMA ping-pong + 去交錯 + 去 DC
│   └── audio_stream.h             │
│
├── frontend/                      ← 第 3 層
│   ├── audio_frontend.c           │  MFCC 滑動窗口（CCMRAM）
│   ├── audio_frontend.h           │
│   ├── audio_params.h             │  採樣率/窗長/步長/量化參數
│   └── mel_filterbank_data.h      │  40 通道 Mel 濾波器組係數表
│
└── kws/                           ← 第 5 層
    ├── kws_postprocess.c          │  消抖 + 冷卻 + score 閾值 + softmax
    └── kws_postprocess.h          │
```

`X-CUBE-AI/App/app_x-cube-ai.c` 保持原位（CubeMX 約束），但只保留模型 IO 模板和頂層調度，KWS 邏輯全部走 `kws_postprocess_feed()` 呼叫。

對應的 CMakeLists.txt：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    # 音頻前端預處理（MFCC / 滑動窗口特徵）
    ${CMAKE_SOURCE_DIR}/audio/frontend/audio_frontend.c
    # 音頻流採集（I2S DMA + 環形緩衝）
    ${CMAKE_SOURCE_DIR}/audio/capture/audio_capture_ring_buff.c
    ${CMAKE_SOURCE_DIR}/audio/capture/audio_stream.c
    # KWS 後處理（消抖 + 冷卻）
    ${CMAKE_SOURCE_DIR}/audio/kws/kws_postprocess.c
    ...
)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    ...
    # 音頻模組（採集 / 前端 / KWS）
    ${CMAKE_SOURCE_DIR}/audio/capture
    ${CMAKE_SOURCE_DIR}/audio/frontend
    ${CMAKE_SOURCE_DIR}/audio/kws
    ...
)
```

> [!TIP]
> 模組化分層的好處不只是「程式碼好看」。這次重構後，我後來想試 `yes/no` 二分類的小模型，**只改了 `app_x-cube-ai.c` 裡的 `NUM_LABELS` 和 `kLabelNames`**，capture 和 frontend 完全不動。模組邊界清晰後，迭代速度明顯快了。

---

## 八、效能與延遲分析

### 8.1 流式 vs 直接輸入的延遲差

很多讀者問：流式推理比直接餵整段錄音慢多少？實測分解：

| 階段             | 延遲           | 說明                                                |
| ---------------- | -------------- | --------------------------------------------------- |
| DMA 半緩衝       | 10 ms          | 160 樣本 / 16 kHz                                   |
| 滑動窗口等待     | 0~200 ms       | SLIDE_SAMPLES=3200（200 ms）；首次還需填滿 1 秒窗口 |
| MFCC 特徵提取    | 30~50 ms       | CMSIS-DSP RFFT Q15 + Mel 濾波                       |
| int8 推理        | ~50 ms         | micro_speech 模型                                   |
| KWS N-in-M 消抖  | +200~600 ms    | 需要 3/5 命中才確認                                 |
| **端到端總滯後** | **500~850 ms** |                                                     |

直接餵整段錄音測試時只有「推理 50 ms」，所以**流式比直接測試慢約 10~17 倍**。主要消耗在滑動窗口填充和 KWS 消抖上——這是流式 KWS 的**固有代價**：用滯後換穩定性。

### 8.2 優化機會

如果覺得 800 ms 太慢，可以調：

- **`SLIDE_COLS` 從 10 改到 5**：滑動間隔 200 ms → 100 ms，但 CPU 開銷翻倍
- **`KWS_REQUIRED_N` 從 3 改到 2**：消抖更寬鬆，但誤報增多
- **`KWS_COOLDOWN` 從 5 改到 3**：冷卻更短，但同一詞可能連發
- **換更小的模型**：SVDF 或 DS-CNN 的精簡版，推理 20 ms 以內

工程永遠是 trade-off。

### 8.3 編譯選項

整個工程用 `-O3 -flto -funroll-loops` 全域優化，連結 `-u _printf_float` 啟用浮點 printf：

```cmake
add_compile_options(
    -O3                  # 最高優化級別
    -flto                # 連結時優化（跨模組內聯）
    -funroll-loops       # 顯式循環展開
)
target_link_options(${CMAKE_PROJECT_NAME} PRIVATE -flto -u _printf_float)
```

注意 `-O3 -flto` 也是 5.2 節 volatile 坑的「元凶」——優化越激進，對可見性要求越嚴格。但只要正確加了 `volatile`，激進優化是好事，MFCC 預處理時間從 -O2 的 60 ms 降到 -O3 的 35 ms。

### 8.4 記憶體佈局優化：CCMRAM 充分利用

STM32F407 除了 128 KB 主 RAM，還有 64 KB CCMRAM（Core Coupled Memory，CPU 專用緊耦合記憶體）。CCMRAM 的特點：

- **零等待訪問**：CPU 專用，無 DMA 仲裁延遲，訪問速度更快
- **DMA 無法訪問**：不能放 DMA 緩衝，但純 CPU 資料隨便放
- **獨立的位址空間**：0x10000000 開始，不佔用主 RAM 空間

本工程把兩個大緩衝都放進了 CCMRAM：

```c
/* 環形緩衝 backing store（CPU-only，25.6 KB）*/
static int16_t s_ring_backing[AS_RING_NB_SAMPLES * AS_RING_NB_FRAMES]
    __attribute__((section(".ccmram")));

/* MFCC 滑動窗口上下文（CPU-only，~8 KB）*/
static AudioFrontendStream g_stream __attribute__((section(".ccmram")));
```

編譯後的記憶體分佈：

| 區域       | 使用量   | 總量   | 佔比      |
| ---------- | -------- | ------ | --------- |
| **RAM**    | 50576 B  | 112 KB | **44.1%** |
| **CCMRAM** | 63144 B  | 64 KB  | **96.4%** |
| FLASH      | 348684 B | 1 MB   | 33.2%     |

**RAM 佔用大頭（50.6 KB）：**

| 資料結構                  | 大小    | 佔比  |
| ------------------------- | ------- | ----- |
| `pool0`（模型啟用緩衝）   | 25.4 KB | 50.3% |
| `g_slide_buf`（滑動緩衝） | 6.4 KB  | 12.7% |
| 其他 .bss/.data           | 18.8 KB | 37.0% |

**CCMRAM 佔用大頭（63.1 KB）：**

| 資料結構                     | 大小        | 說明                         |
| ---------------------------- | ----------- | ---------------------------- |
| `s_ring_backing`（環形緩衝） | **25.6 KB** | 從 RAM 移入，節省 22.6% RAM  |
| `g_stream`（滑動窗口）       | ~8 KB       | 1 秒音頻窗口 + MFCC 中間緩衝 |
| `s_mono`（DMA 臨時）         | 320 B       | 單聲道去交錯緩衝             |
| 其他緩衝                     | ~29 KB      | 音頻前端靜態陣列等           |

> [!TIP]
> 環形緩衝移到 CCMRAM 是後期優化。初期為了避免踩坑，先放主 RAM，等系統穩定後再搬遷。搬遷時只需加一行 `__attribute__((section(".ccmram")))`，零風險——因為環形緩衝只有 CPU 訪問，DMA 從不碰它。

如果把環形緩衝留在主 RAM，RAM 佔用會是 66.7%（76.2 KB），留給棧/堆的空間更緊張。搬到 CCMRAM 後，RAM 餘量從 35.8 KB 提升到 61.4 KB，為將來加模型或功能留出更多空間。

---

## 九、踩坑總結表

把五個坑彙總成一張表，方便回顧：

| #   | 坑名                   | 症狀                               | 根因                                          | 修復                                          |
| --- | ---------------------- | ---------------------------------- | --------------------------------------------- | --------------------------------------------- |
| 1   | stride bug             | 識別全錯，score 14~30              | STM32F4 16B_EXTENDED 每幀 2 halfwords，不是 4 | `i*4+2` → `i*2+idx`，DMA size 減半            |
| 2   | volatile 缺失          | 刪 printf 系統死機，加 printf 復活 | `-O3 -flto` 快取 ISR 共享變數到暫存器         | `availableSamples` 加 `volatile`              |
| 3   | 棧餘量不足             | 預防性優化（未觀察到明確症狀）     | MFCC 大陣列放棧上，預設 2 KB 棧餘量危險       | 陣列改 `static`，棧 2 KB → 8 KB               |
| 4   | KWS 誤報               | 噪音識別成 up，連續觸發            | per-word 冷卻被中間詞重置 + score 重疊        | 全域冷卻 + N-in-M + score 閾值 + 過濾 silence |
| 5   | kernel_util.cpp 連結錯 | `undefined reference`              | TFLM 漏了 `array.cc` 隱藏依賴                 | CMakeLists 顯式加 `tensorflow/lite/array.cc`  |

---

## 十、經驗沉驗沉澱

寫完這個工程，有幾條經驗值得記下來：

1. **移植程式碼永遠不要假設「應該一樣」**。即使是同一家廠商（ST）的同名外設（I2S），不同晶片代際的細節也可能不同。手冊優先於記憶。

2. **原子操作 ≠ 可見性**。LDREX/STREX 解決原子性，`volatile` 解決可見性，兩者正交。bare-metal 上自己寫併發原語時，**兩個都要**。Linux 核心的 `atomic_t` 之所以不用加 volatile，是因為內部封裝了 memory barrier。

3. **printf 能「修」bug，說明有可見性問題**。如果加一行無關 printf 讓 bug 消失或出現，第一時間懷疑編譯器優化 + 共享變數可見性，而不是懷疑 printf 本身。

4. **大陣列預設 `static`**。嵌入式 C 的黃金法則，能避免 80% 的棧溢位問題。

5. **模型量化損失無法用軟體完全彌補**。int8 量化後某些類的 score 天生偏低（如本文的 `up`），與噪音誤報重疊。KWS 後處理只能降低誤報頻率，不能根治。要根治得換模型或加 fine-tune 負樣本。

6. **模組化不是為了好看，是為了可替換**。本次重構後，想換模型只需改 `app_x-cube-ai.c`，採集和前端完全不動。如果將來要支援 PDM 麥克風，只改 `audio/capture/`，前端和 KWS 不動。**模組邊界 = 替換邊界**。

7. **CubeMX 生成檔案不移動**。`app_x-cube-ai.c` 這種 CubeMX 模板檔案，移動會破壞 CubeMX 重新生成能力。保持原位，把業務邏輯拆到獨立模組呼叫即可。

---

## 結語

這個工程從「能跑但亂識別」到「流式穩定識別」，前前後後踩了五個坑，每個坑都是一兩個小時。但回頭看，每個坑都教會了我一條**底層原理**——I2S 幀佈局、記憶體可見性、棧管理、模型量化、依賴管理。這些原理換晶片、換工程都用得上。

寫這篇文章的初衷，就是希望後來人能少繞幾圈。如果按本文的架構和注意事項來，從零搭一個 STM32F4 + TFLM 流式 KWS，應該半天就能跑通。

完整工程檔案結構：

{{< details "專案樹（點擊展開）" >}}

```
STM32F407VGT6_TensorFlow-Lite-Micro/
├── audio/
│   ├── capture/
│   │   ├── audio_capture_ring_buff.c
│   │   ├── audio_capture_ring_buff.h
│   │   ├── audio_stream.c
│   │   └── audio_stream.h
│   ├── frontend/
│   │   ├── audio_frontend.c
│   │   ├── audio_frontend.h
│   │   ├── audio_params.h
│   │   └── mel_filterbank_data.h
│   └── kws/
│       ├── kws_postprocess.c
│       └── kws_postprocess.h
├── build/
│   └── Debug/
│       ├── STM32F407VGT6_TensorFlow-Lite-Micro.bin
│       ├── STM32F407VGT6_TensorFlow-Lite-Micro.hex
│       └── ... (編譯產物)
├── cmake/
│   └── stm32cubemx/
│       ├── CMakeLists.txt
│       └── ... (CubeMX 生成)
├── Core/
│   ├── Inc/
│   │   ├── main.h
│   │   ├── gpio.h
│   │   ├── i2s.h
│   │   ├── usart.h
│   │   └── ... (CubeMX 生成)
│   └── Src/
│       ├── main.c
│       ├── gpio.c
│       ├── i2s.c
│       ├── usart.c
│       ├── stm32f4xx_it.c
│       └── ... (CubeMX 生成)
├── Drivers/
│   ├── CMSIS/
│   ├── STM32F4xx_HAL_Driver/
│   └── ... (HAL + CMSIS)
├── stm32f4_tflm_project/
│   ├── tensorflow/
│   │   └── lite/
│   │       ├── micro/
│   │       │   ├── kernels/
│   │       │   ├── arena_allocator/
│   │       │   ├── memory_planner/
│   │       │   └── ... (TFLM 核心)
│   │       └── core/
│   ├── signal/
│   │   ├── micro/kernels/
│   │   └── src/
│   ├── third_party/
│   │   ├── cmsis_nn/
│   │   ├── kissfft/
│   │   └── flatbuffers/
│   └── ... (TFLM 原始碼)
├── X-CUBE-AI/
│   └── App/
│       ├── app_x-cube-ai.c
│       ├── app_x-cube-ai.h
│       └── micro_speech.h
├── CMakeLists.txt
├── STM32F407XX_FLASH.ld
├── stm32f4_tflm_project.md
└── ... (配置檔案)
```

{{< /details >}}

感謝閱讀。如果踩過相同的坑，歡迎交流；如果正在踩坑，希望本文能給你一點啟發。
