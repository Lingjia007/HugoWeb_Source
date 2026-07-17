---
title: "STM32F407 + TensorFlow-Lite-Micro 流式语音关键词检测全链路实战：从诡异 Bug 到模块化架构"
date: 2026-07-17
description: "在 STM32F407VGT6 上跑通 micro_speech 流式关键词检测的完整踩坑录：INMP441 I2S MEMS 麦克风、DMA ping-pong、环形缓冲、MFCC 滑动窗口、int8 量化推理、KWS 消抖冷却后处理。详解 stride 帧布局陷阱、volatile 可见性、栈溢出、DWT 失效等六大坑，以及最终的 audio/capture+frontend+kws 模块化分层架构"
image: STM32F407-TFLM-流式语音识别.png
categories:
  - "嵌入式"
  - "边缘AI"
tags:
  - "STM32"
  - "TensorFlow-Lite-Micro"
  - "micro_speech"
  - "INMP441"
  - "I2S"
  - "DMA"
  - "关键词检测"
  - "KWS"
  - "volatile"
  - "模块化"
math: true
---

## 前言

故事要从一句诡异的吐槽说起。

我对着麦克风字正腔圆地喊了一声 "yes"，串口屏上却跳出来一行：

```
[KWS 0] up (96.7%, score=18, hits=3/5)
```

`up`？我明明说的是 `yes`。更离谱的是，连续说 "one / two / three"，识别结果在 `cat`、`right`、`up` 之间随机横跳，置信度只有 14~30。而当我把模型直接喂整段录音测试时，明明能稳定识别。**流式推理到底哪里出了问题？**

接下来是一连串更诡异的现象：

- 删掉一行 `printf`，整个系统就再无输出，加上又"复活"了
- 把大数组从栈移到 `static` 后崩溃消失
- DWT 周期计数器读出来永远是 0
- 噪音被识别成 `up`，分数竟然和真说 "up" 时差不多

这篇文章记录的就是把 [STM32F407VGT6_TensorFlow-Lite-Micro](file:///d:/edgedownload/STM32F407VGT6_TensorFlow-Lite-Micro) 这个工程从"能跑但乱识别"调到"流式稳定识别 yes/no/up/down/..."的完整过程。每个坑都附了**根因分析**和**修复方案**，希望后来人能少绕几圈。

> [!NOTE]
> 本文不是保姆级教程，假设读者已熟悉 STM32 HAL、I2S、DMA、TFLite 基础。重点在**踩坑与架构**，不是从零搭建。模型训练部分见前作 [WSL2 + Docker + TensorFlow 教程](../wsl2-docker-tensorflow/)。

---

## 一、工程总览

### 1.1 硬件配置

| 部件     | 型号 / 配置                                                                  |
| -------- | ---------------------------------------------------------------------------- |
| 主控     | STM32F407VGT6（Cortex-M4F，168 MHz，1 MB Flash，192 KB RAM 含 64 KB CCMRAM） |
| 麦克风   | INMP441 I2S MEMS（24-bit，L/R 引脚接地 = 左通道）                            |
| I2S 接口 | I2S2，Philips 标准，16B_EXTENDED，16 kHz 立体声                              |
| DMA      | DMA1_Stream3 / Channel0，Circular，半字对齐                                  |
| 调试器   | J-Link                                                                       |
| 模型     | micro_speech int8 量化（36 标签）                                            |
| 推理框架 | TensorFlow-Lite-Micro + X-CUBE-AI 集成                                       |

### 1.2 软件数据流

整个流式 KWS 链路分成五层，从麦克风到关键词输出：

```
[INMP441]
   │ I2S2 24-bit @ 16 kHz
   ▼

audio/capture/                       ← 第 1 层：I2S DMA ping-pong
  • DMA 双缓冲 (640 halfwords)
  • Half/Full IRQ 各 10 ms
  • 左通道去交错 [L,R] → [L]
  • 按 block 减去均值去 DC
   ▼ feed (160 samples / 10 ms)

audio/capture/                       ← 第 2 层：环形缓冲
  AudioCaptureRingBuff_t
  4 × 3200 = 12800 samples
  LDREX/STREX 原子计数器
  volatile 保证 ISR/main 可见
   ▼ consume (3200 samples / 200 ms)

audio/frontend/                      ← 第 3 层：MFCC 滑动窗口
  • 1 秒窗口 (16000 samples)
  • SLIDE_COLS=10 (200 ms)
  • CMSIS-DSP RFFT Q15
  • 40 通道 Mel 滤波器组
  • 存放在 CCMRAM 节省 RAM
   ▼ 49×40 int8 特征

X-CUBE-AI                            ← 第 4 层：int8 推理
  micro_speech 模型 (~50 ms)
   ▼ 36 个 int8 logits

audio/kws/                           ← 第 5 层：KWS 后处理
  • 过滤 _silence_/_unknown_
  • N-in-M 消抖 (3/5)
  • 全局冷却 (5 帧 ≈ 1 s)
  • score 阈值 (>= 20)
  • softmax 概率显示
   ▼

[KWS N] yes (100.0%, score=83)
```

每一层都是一个独立的 `.c/.h` 模块，最后通过 `X-CUBE-AI/App/app_x-cube-ai.c` 这个 CubeMX 模板文件串起来。

> [!TIP]
> 为什么要分层？因为流式 KWS 的 bug 80% 出在"层与层的接口"上：DMA 帧布局错了、环形缓冲可见性错了、滑动窗口步长错了……分层之后每层都能独立验证，定位 bug 时按层排查即可。

---

## 二、第 1 层：I2S DMA ping-pong 采集

### 2.1 为什么用 16B_EXTENDED 而不是 24B/32B？

INMP441 是 24-bit MEMS 麦克风，按理说应该用 24 位或 32 位 I2S 格式才"对味"。但实际上，micro_speech 模型只需要 16-bit 输入，而 STM32F4 的 I2S 24B/32B 模式会让 DMA 传输 4 个半字（每通道 2 个），多搬一半数据，浪费带宽和 CPU 去交错时间。

`16B_EXTENDED` 模式则是一个折中：**32-bit 时隙 + 16-bit 数据**，每个通道只搬 1 个半字，DMA 拿到的就是已经对齐的 16-bit 样本。

> [!WARNING]
> 这就是第一个大坑的发源地——STM32N6 原工程**根本不用 I2S**（用的是 MDF 数字麦克风接口 + BSP 单声道抽象），所以没有去交错逻辑可供参考。在 STM32F4 上新写去交错代码时，对 `16B_EXTENDED` 帧布局的误解导致 stride=4 错误。详见 [5.1 节](#51-stride-bug16b_extended-帧布局陷阱)。

### 2.2 DMA 双缓冲与回调

```c
/* DMA 双缓冲：必须放在普通 RAM，DMA 访问不到 CCMRAM */
static uint16_t s_dma_buf[AS_DMA_BUF_HALFWORDS];  /* 640 halfwords */

/* 启动循环模式 DMA：会触发 Half / Full 两个回调 */
HAL_I2S_Receive_DMA(&hi2s2, (uint16_t *)s_dma_buf, AS_DMA_BUF_HALFWORDS);
```

`AS_DMA_HALF_FRAMES = 160`，意味着每个半缓冲装 10 ms 的单声道样本（16000 Hz × 10 ms = 160）。`HAL_I2S_RxHalfCpltCallback` 在 DMA 写到一半时触发，`HAL_I2S_RxCpltCallback` 在写完整圈时触发，两个回调分别处理 `s_dma_buf` 的前半段和后半段，形成无缝的 ping-pong。

### 2.3 去交错与 DC 去除

```c
static void AudioStream_process_half(uint8_t half_id)
{
  const uint16_t *src = s_dma_buf + half_id * AS_DMA_HALFWORDS;

  /* STM32F4 16B_EXTENDED: 每帧 2 halfwords [L, R]
   * INMP441 L/R=GND → 左通道，AUDIO_CHANNEL_INDEX=0 */
  int32_t sum = 0;
  for (uint32_t i = 0; i < AS_DMA_HALF_FRAMES; i++)
  {
    int16_t v = (int16_t)src[i * 2 + AUDIO_CHANNEL_INDEX];
    s_mono[i] = v;
    sum += v;
  }

  /* 按 block 减均值去 DC：MEMS 麦克风有几十~几百的 DC 偏置 */
  int16_t dc = (int16_t)(sum / (int32_t)AS_DMA_HALF_FRAMES);
  for (uint32_t i = 0; i < AS_DMA_HALF_FRAMES; i++)
    s_mono[i] = (int16_t)(s_mono[i] - dc);

  AudioCaptureRingBuff_feed(&s_ring, (const uint8_t *)s_mono,
                            (uint16_t)AS_DMA_HALF_FRAMES);
}
```

**为什么要去 DC？** INMP441 上电后输出会有一个固定的 DC 偏置（实测几十到几百），如果不去掉，下游 MFCC 的能量谱会整体抬高，模型量化后的 int8 特征会偏移。每 10 ms 一个 block 减去均值，既简单又稳定，比一阶 IIR 高通滤波器便宜得多。

---

## 三、第 2 层：环形缓冲（lock-free）

### 3.1 为什么要环形缓冲？

DMA 中断每 10 ms 产生 160 个样本，但消费端（MFCC）每 200 ms 才需要 3200 个样本。生产者和消费者速率严重不匹配，需要一个 FIFO 解耦。

| 通路                 | 频率   | 速率            |
| -------------------- | ------ | --------------- |
| Producer (DMA IRQ)   | 100 Hz | 160 samples/次  |
| Consumer (main loop) | 5 Hz   | 3200 samples/次 |

### 3.2 原子计数器与可见性

环形缓冲的核心字段是 `availableSamples`，被 ISR 写、main 循环读。这两个上下文是真正的并发，必须解决两个问题：

1. **原子性**：32 位读写本身在 Cortex-M4 上是原子的，但"读-改-写"（`+=`）不是。用 `LDREX/STREX` 独占访问解决：

   ```c
   static inline void atomic_add(volatile uint32_t *p32, int32_t inc)
   {
     do { } while (__STREXW(__LDREXW(p32) + inc, p32));
   }
   ```

2. **可见性**：这是第二个大坑（详见 [5.2 节](#52-volatile-缺失atomic--visibility)）。简而言之，必须加 `volatile`：

   ```c
   typedef struct {
     uint8_t  *pData;
     ...
     volatile uint32_t availableSamples;  /* ISR/main 共享，必须 volatile */
   } AudioCaptureRingBuff_t;
   ```

### 3.3 消费端的简易临界区

```c
uint8_t *AudioCaptureRingBuff_consume(uint8_t *pData,
                                      AudioCaptureRingBuff_t *pHdle,
                                      uint32_t nbSamples)
{
  /* 消费端在 main 循环，brief IRQ masking 保持索引一致 */
  __disable_irq();
  if (pHdle->availableSamples >= (int32_t)nbSamples) {
    /* ... 两段拷贝处理回绕 ... */
    atomic_add(&pHdle->availableSamples, -(int32_t)nbSamples);
  }
  __enable_irq();
  return pData;
}
```

生产端不能用 `__disable_irq()`（会抖动 DMA 时序），所以用 LDREX/STREX；消费端在 main 循环里，关中断几微秒无所谓，直接关 IRQ 更简单。

> [!NOTE]
> 这种"生产端原子、消费端关中断"的不对称设计，是 bare-metal 环形缓冲的经典模式。如果换成 FreeRTOS，应该用 `xStreamBuffer` 或带互斥锁的队列。

---

## 四、第 3 层：MFCC 滑动窗口

### 4.1 关键参数

```c
#define SAMPLE_RATE          16000     // 采样率
#define FRAME_LEN            480       // 30 ms 窗长 = 480 samples
#define WINDOW_STRIDE        320       // 20 ms 步长 = 320 samples
#define FFT_LENGTH           512       // 下一个 2^n >= 480
#define NUM_CHANNELS         40        // Mel 滤波器组通道数
#define SPECTROGRAM_LENGTH   49        // (16000-480)/320 + 1

#define SLIDE_COLS           10        // 每次滑动 10 个频谱列
#define SLIDE_SAMPLES        3200      // = 10 × 320 = 200 ms
```

### 4.2 滑动窗口机制

micro_speech 模型的输入是 1 秒（16000 样本）的频谱图，形状为 `49 × 40`。如果每次都从头算 1 秒窗口的 MFCC，CPU 开销太大。

**滑动窗口策略**：维护一个 1 秒的环形音频窗口，每来 200 ms 新音频（3200 样本），窗口滑动 10 个频谱列，只计算这 10 个新列，老的 39 列复用。这样每次推理前的预处理开销从 ~200 ms 降到 ~30 ms。

```
窗口状态 (49 列频谱图):
┌──────────────────────────────────────────────────────┐
│ [老 39 列]                              │ [新 10 列] │
└──────────────────────────────────────────────────────┘
                                                  ▲
                                  200 ms 后：      │
                                                  ▼
┌──────────────────────────────────────────────────────┐
│          │ [老 39 列]                    │ [新 10 列] │
└──────────────────────────────────────────────────────┘
   ← 丢弃
```

### 4.3 CCMRAM：把 1 秒窗口塞进去

`AudioFrontendStream` 结构体里包含 16000 个 `int16_t` 的音频窗口（32 KB）加上频谱缓存，总共 ~37 KB。STM32F407 主 RAM 只有 128 KB（含栈和堆），而 CCMRAM 有 64 KB 专门给数据用，DMA 又访问不到 CCMRAM。

所以策略是：**DMA 缓冲放主 RAM，前端窗口放 CCMRAM**。

```c
/* 1 秒窗口 + 特征缓存放 CCMRAM，主 RAM 留给栈/堆/模型激活 */
static AudioFrontendStream g_stream __attribute__((section(".ccmram")));
```

最终内存占用：

```
RAM:    82320 B / 112 KB  (71.78%)
CCMRAM: 37544 B / 64 KB   (57.29%)
FLASH:  348684 B / 1 MB   (33.25%)
```

> [!TIP]
> CCMRAM (Core Coupled Memory) 是 Cortex-M4 的一块 0 等待周期 SRAM，访问速度比主 RAM 快得多，特别适合放频繁访问的大数组。但代价是 DMA 不能用，所以选放什么进去很有讲究。

---

## 五、踩坑实录（重头戏）

下面六个坑，每一个都让我花了至少一小时定位。按踩中顺序排列。

### 5.1 stride bug：16B_EXTENDED 帧布局陷阱

#### 症状

流式推理识别结果全错，对着麦克风说 "yes" 识别成 "up"，说 "one" 识别成 "cat"，score 只有 14~30。但用同一份模型直接喂整段录音测试，识别完全正常。

#### 根因

代码是从 STM32N6 工程移植过来的。STM32N6 的 I2S 在 16B_EXTENDED 模式下，每个立体声帧传输 **4 个半字** `[L_pad, L, R_pad, R]`（24-bit 槽位 + padding）。而 STM32F4 的 16B_EXTENDED 是 **2 个半字** `[L, R]`（纯 16-bit 数据，无 padding）。

我移植时没改 stride，DMA 去交错代码写成：

```c
/* ❌ 错误：STM32N6 的 stride=4，在 STM32F4 上等效降采样 */
int16_t v = (int16_t)src[i * 4 + 2];  /* 取 R 通道 */
```

在 STM32F4 上 `src[i*4+2]` 实际跨越了 2 个立体声帧，等于**只取了一半的左通道样本**。等效于把 16 kHz 采样率降到了 8 kHz，所有频率成分都错位了一半。模型看到 "yes" 的频谱图实际是 "yes" 降采样后的扭曲版本，自然识别错。

#### 修复

```c
/* ✅ 正确：STM32F4 16B_EXTENDED 每帧 2 halfwords [L, R] */
#define AS_DMA_HALFWORDS (AS_DMA_HALF_FRAMES * 2)   /* 320, 不是 640 */

int16_t v = (int16_t)src[i * 2 + AUDIO_CHANNEL_INDEX];  /* stride=2 */
```

修复后 score 从 14~30 飙升到 **42~100**，"yes/no/up/down" 全部正确识别。

> [!WARNING]
> 这个 stride bug 的真正根源不是"配置不同"，而是 **STM32N6 和 STM32F4 用了完全不同的音频外设架构**——N6 原工程根本不用 I2S！移植时数据获取层必须重写，不能照搬。

**STM32N6 原工程 vs STM32F407 本工程的音频路径对比：**

| 维度                | STM32N6（原工程）                                        | STM32F407（本工程）                        |
| ------------------- | -------------------------------------------------------- | ------------------------------------------ |
| 音频外设            | **MDF**（Multi-function Digital Filter，数字麦克风接口） | **I2S2**（SPI2 复用）                      |
| 驱动抽象            | `BSP_AUDIO_IN_*` 高层 API                                | 裸 `HAL_I2S_*`                             |
| 通道配置            | `ChannelsNbr = 1`（单声道）                              | I2S 协议**强制立体声**（WS 在 L/R 间切换） |
| DMA buffer 内容     | 纯 mono int16 序列                                       | `[L, R, L, R, ...]` 立体声交错             |
| 16B_EXTENDED 帧布局 | 不适用（MDF 不走 I2S 帧格式）                            | 每帧 2 halfwords `[L, R]`                  |
| 去交错代码          | **不需要**（BSP 已配 mono，直接喂 ring）                 | **必须新写**（从 `[L,R]` 提取 L）          |

N6 工程的初始化代码长这样：

```c
AudioInit.Device        = AUDIO_IN_DEVICE_DIGITAL_MIC;  /* MDF 接数字麦克风 */
AudioInit.BitsPerSample = AUDIO_RESOLUTION_16B;
AudioInit.ChannelsNbr   = 1;                            /* 单声道，关键！ */
BSP_AUDIO_IN_Init(1, &AudioInit);
BSP_AUDIO_IN_Record(1, (uint8_t *)acq_buf, CAPTURE_BUFFER_SIZE * sizeof(int16_t));
```

DMA 完成中断里直接 `AudioCapture_ring_buff_feed(acq_buf, ring, len)`，**中间没有任何 stride 处理**——因为 buffer 本来就是单声道。

而 STM32F4 这边，I2S 协议物理上就是立体声（WS=LOW 传左、WS=HIGH 传右），INMP441 即使只有一个麦克风也输出立体声流，必须按 `[L,R]` 接收再丢弃一半。**N6 工程没有这段逻辑可供参考**，移植时新写的去交错代码就因为对 `16B_EXTENDED` 帧布局的误解踩了 stride=4 的坑。

> [!TIP]
> 移植经验：跨芯片移植音频代码时，**先确认两边的音频外设类型**（I2S / SAI / MDF / SPDIFRX），而不是直接看采样率和位宽。外设类型不同意味着 DMA buffer 布局可能完全不同，去交错逻辑必须基于目标芯片的参考手册重写。可以用逻辑分析仪抓 WS + SD 时序，直接数一个 WS 周期内有几个 16-bit slot，比看手册更快。

### 5.2 volatile 缺失：atomic ≠ visibility

#### 症状

修复 stride bug 后识别正常。我觉得代码稳定了，把诊断用的 `printf` 全删掉，结果——**系统再无任何输出**，只剩启动横幅。

更诡异的是，加回一行无关的 `printf`（每 2 秒打印一次 ring 可用样本数），系统又恢复正常。删掉，又死。这个 `printf` 跟识别逻辑毫无关系，但它"修复"了 bug。

#### 根因

`availableSamples` 字段缺 `volatile`：

```c
/* ❌ 错误：缺 volatile */
typedef struct {
  ...
  uint32_t availableSamples;
} AudioCaptureRingBuff_t;
```

在 `-O3 -flto` 优化下，编译器把 main 循环里的 `while (ring->availableSamples < SLIDE_SAMPLES) break;` 内联后，**把 `availableSamples` 缓存到了寄存器**，再也不会从内存重读。ISR 写入的新值 main 循环永远看不到，于是 `availableSamples` 在 main 看来永远是 0，循环一直 break，推理永远不执行。

那为什么加 `printf` 又"复活"了？因为 `printf` 是函数调用，形成**编译屏障（compiler barrier）**，强制编译器在调用前把寄存器写回内存、调用后从内存重读。所以 `printf` 阴差阳错地绕过了可见性问题。

#### 修复

```c
/* ✅ 正确：ISR/main 共享变量必须 volatile */
typedef struct {
  ...
  volatile uint32_t availableSamples;
} AudioCaptureRingBuff_t;

/* atomic_add 参数也要同步改 */
static inline void atomic_add(volatile uint32_t *p32, int32_t inc)
{
  do { } while (__STREXW(__LDREXW(p32) + inc, p32));
}
```

> [!IMPORTANT]
> 这是嵌入式 C 最经典的坑之一：**LDREX/STREX 只保证原子性，不保证可见性**。原子操作解决的是"读-改-写不可分割"，可见性解决的是"修改对其他上下文可见"。两者是正交的概念，缺一不可。
>
> 在 Linux 内核里 `atomic_t` 内部封装了 memory barrier，所以用户不用关心。但 bare-metal 上自己写原子操作，**volatile 仍然必须加**。

### 5.3 栈溢出：8 KB 都不够

#### 症状

调试前期，系统运行几秒后莫名重启（看门狗复位）。打开调试器发现是 HardFault，跳到 `MemManage_Handler`，栈指针跑飞。

#### 根因

`audio_frontend_process_frame()` 函数里声明了几个大数组作为局部变量：

```c
/* ❌ 错误：几百 KB 的局部数组放栈上 */
void audio_frontend_process_frame(...)
{
  int16_t window[FRAME_LEN];           /* 480 × 2 = 960 B */
  int16_t fft_input[FFT_LENGTH];       /* 512 × 2 = 1024 B */
  int16_t fft_output[FFT_LENGTH * 2];  /* 2048 B */
  q15_t mel_output[NUM_CHANNELS];      /* 80 B */
  /* ... 还有几个 ... */
}
```

加起来每帧调用 ~5 KB 栈，加上中断嵌套和其他函数的栈帧，STM32F4 默认的 2 KB 栈瞬间爆掉。

#### 修复

两步走：

1. **大数组改 `static`**：移到 BSS 段，函数不可重入但本来就是单线程调用，没问题。

   ```c
   void audio_frontend_process_frame(...)
   {
     static int16_t window[FRAME_LEN];
     static int16_t fft_input[FFT_LENGTH];
     static int16_t fft_output[FFT_LENGTH * 2];
     /* ... */
   }
   ```

2. **栈大小从 0x800 提到 0x2000**：在 `STM32F407XX_FLASH.ld` 里：

   ```
   _Min_Stack_Size = 0x2000 ;  /* 8 KB，原来 0x800 = 2 KB */
   ```

> [!TIP]
> 嵌入式 C 经验法则：**任何超过 256 字节的局部数组都应该改 `static`**。Cortex-M4 的栈本来就小，函数嵌套几层就紧张。`static` 的代价是失去可重入性，但 90% 的场景下函数根本不会被并发调用。

### 5.4 DWT CYCCNT 失效：被 CMSIS-DSP 静默禁用

#### 症状

想测试预处理和推理各花多少时间。按标准方法使能 DWT 周期计数器：

```c
CoreDebug->DEMCR |= (1u << 24);    /* TRCENA */
DWT->CTRL |= (1u << 0);            /* CYCCNTENA */
DWT->CYCCNT = 0;
/* ... 跑一段代码 ... */
uint32_t ticks = DWT->CYCCNT;      /* 读出来永远是 0 或固定值 */
```

读出来永远是 0，或者一个不变的固定值。

#### 根因

CMSIS-DSP 的某些函数（比如 `arm_rfft_q15`）**内部会修改 DWT CTRL 寄存器**来给自己做性能分析，调用结束后**不会恢复 CYCCNTENA 位**，把计数器静默禁用了。

所以你的代码先使能 CYCCNT，然后调用 MFCC（里面调了 CMSIS-DSP），计数器就被关掉了，再读 CYCCNT 当然是 0 或固定值。

#### 修复

放弃 DWT，改用 TIM5（32-bit 定时器，APB1 时钟 84 MHz）：

```c
#define TIM5_TIMER_CLOCK 84000000U

static void timing_init(void)
{
  __HAL_RCC_TIM5_CLK_ENABLE();
  TIM5->PSC = 0;            /* 不分频，84 MHz 计数 */
  TIM5->ARR = 0xFFFFFFFFU;  /* 32-bit 最大周期 */
  TIM5->CNT = 0;
  TIM5->CR1 = TIM_CR1_CEN;
}

static inline float ticks_to_us(uint32_t ticks)
{
  return (float)ticks * 1000000.0f / (float)TIM5_TIMER_CLOCK;
}

/* 用法 */
uint32_t t0 = TIM5->CNT;
run_inference();
uint32_t t1 = TIM5->CNT;
printf("inf_us = %.1f\r\n", ticks_to_us(t1 - t0));
```

TIM5 是独立的硬件定时器，CMSIS-DSP 改不到它。

> [!WARNING]
> 不要相信"标准"性能测量方法在所有工程里都有效。一旦引入了第三方库（特别是 CMSIS-DSP 这种"上帝库"），它可能改你不知道的寄存器。实测永远比记忆可靠。

### 5.5 KWS 误报：up 的 int8 score 天生偏低

#### 症状

环境噪音被识别成 `up`，score 15~18，连续多次触发。但真说 "up" 时 score 也只有 22~23，和噪音误报重叠，单纯提高阈值没法区分。

更恶心的是，第一版 KWS 后处理里 cooldown 是 per-word 的，结果中间穿插的 `wow`、`follow` 误识别会重置 `up` 的冷却计数器，让 `up` 连续触发 9 次。

#### 根因

1. **模型本身问题**：micro_speech 的 int8 量化让 `up` 这个词的 logit 天生偏低（22~23），与噪音误报（15~18）只差 5。这种重叠是模型量化损失，软件层面无解。

2. **后处理设计缺陷**：per-word cooldown 假设不同词之间独立，但实际上一个长音节可能跨越多个滑动窗口，被模型分段识别成不同词（"foll" → follow，"ow" → wow），这些中间词会重置目标词的冷却。

#### 修复

KWS 后处理四件套：

```c
#define KWS_HISTORY_M   5    /* 回看最近 5 帧推理（≈ 1 s）*/
#define KWS_REQUIRED_N  3    /* 同一标签在窗口内出现 >= 3 次才确认 */
#define KWS_COOLDOWN    5    /* 任意触发后全局抑制 5 帧（≈ 1 s）*/
#define KWS_MIN_SCORE   20   /* 低于 20 直接丢弃（噪音通常 15~18）*/

int kws_postprocess_feed(kws_state_t *s,
                         const int8_t *logits,
                         uint8_t num_labels,
                         const char *const *label_names)
{
  /* 1) argmax */
  int max_idx = argmax(logits, num_labels);

  /* 2) 丢弃 _silence_/_unknown_（标签 0/1）*/
  if (max_idx <= 1) return 0;

  /* 3) 记录历史 */
  s->hist[s->hist_idx] = (int8_t)max_idx;
  s->hist_idx = (s->hist_idx + 1) % KWS_HISTORY_M;

  /* 4) 全局冷却（不是 per-word！）*/
  if (s->cooldown_remaining > 0) {
    s->cooldown_remaining--;
    return 0;
  }

  /* 5) score 阈值 */
  if (max_val < KWS_MIN_SCORE) return 0;

  /* 6) N-in-M 消抖 */
  uint8_t hits = count_hits(s->hist, max_idx);
  if (hits < KWS_REQUIRED_N) return 0;

  /* 7) 触发：softmax 打印 + 设全局冷却 + 清空历史 */
  print_softmax(logits, num_labels, label_names, max_idx);
  s->cooldown_remaining = KWS_COOLDOWN;
  for (int i = 0; i < KWS_HISTORY_M; i++) s->hist[i] = -1;
  return 1;
}
```

关键改进：

- **全局冷却** 替代 per-word 冷却：任意关键词触发后，**所有**关键词都被抑制 1 秒，杜绝一个词连续刷屏。
- **N-in-M 消抖**：要求 1 秒内同一标签出现至少 3 次，过滤掉单帧的瞬时误识别。
- **过滤 silence/unknown**：这两个标签会刷屏（噪音环境下 score 50+），对用户没用，直接丢弃。

> [!NOTE]
> 模型本身的 score 重叠问题，软件没法完全解决。如果想根治，要么换更大的模型，要么做 fine-tune 时加更多 `up` 的负样本（噪音）。本文方案是把误报频率从"每秒 5 次"降到"每几秒一次"，工程上可接受。

### 5.6 kernel_util.cpp 的隐藏依赖

#### 症状

链接时报 `undefined reference to tflite::...` 一堆错误，全是 `kernel_util.cpp` 里的符号。

#### 根因

TFLite Micro 的 `kernel_util.cpp` 依赖 upstream TensorFlow Lite 的 `array.h` 和 `array.cc`，这两个文件在 TFLM 的 `tensorflow/lite/` 目录下，但默认不会被 TFLM 的 CMakeLists 收集进来。

#### 修复

在 CMakeLists.txt 显式加上：

```cmake
set(TFLM_SOURCES
    ...
    # TFLite core (kernel_util.cpp 的隐藏依赖)
    ${TFLM_BASE}/tensorflow/lite/array.cc
    ${TFLM_BASE}/tensorflow/lite/core/c/common.cpp
    ...
    ${TFLM_BASE}/tensorflow/lite/kernels/kernel_util.cpp
    ...
)
```

> [!TIP]
> TFLM 的官方 CMake 模板不完整，自己拼装工程时遇到 `undefined reference` 先去 `tensorflow/lite/` 找同名 `.cc` 文件。它的目录结构是 TFLM (`tensorflow/lite/micro/`) 调用 TFLite core (`tensorflow/lite/`)，后者经常漏文件。

---

## 六、第 4+5 层：推理与 KWS 后处理

### 6.1 推理入口

`X-CUBE-AI` 生成的 `app_x-cube-ai.c` 是 CubeMX 模板文件，**不能移动位置**（移动会破坏 CubeMX 重新生成能力）。所以这个文件留在 `X-CUBE-AI/App/`，但里面的 KWS 后处理逻辑被拆到了独立的 `audio/kws/` 模块。

```c
void MX_X_CUBE_AI_Process(void)
{
  int res;
  do {
    /* 1) 从环形缓冲取 200 ms 数据 + 滑动窗口 + MFCC */
    res = acquire_and_process_data(data_ins);
    if (res == -2) break;  /* 数据不够或窗口未填满 */

    /* 2) 推理 */
    if (res == 0)
      res = ai_run();

    /* 3) KWS 后处理（debounce + cooldown + softmax 打印）*/
    if (res == 0)
      res = post_process(data_outs);
  } while (res == 0);
}
```

### 6.2 post_process 的最终形态

经过模块化重构后，`post_process` 从 75 行缩成了 1 行调用：

```c
int post_process(ai_i8 *data[])
{
  const int8_t *out = (const int8_t *)data[0];
  g_inference_cnt++;

  /* KWS 逻辑全部封装在 audio/kws/kws_postprocess.c */
  (void)kws_postprocess_feed(&g_kws, out, NUM_LABELS, kLabelNames);
  return 0;
}
```

---

## 七、模块化重构：audio/ 三层目录

踩完坑之后，代码散落在 `audio_app/`、`preprocess/`、`app_x-cube-ai.c` 三个地方，KWS 逻辑和模板代码混在一起。重构后的目录结构：

```
audio/
├── capture/                       ← 第 1+2 层
│   ├── audio_capture_ring_buff.c  │  环形缓冲（lock-free, volatile）
│   ├── audio_capture_ring_buff.h  │
│   ├── audio_stream.c             │  I2S DMA ping-pong + 去交错 + 去 DC
│   └── audio_stream.h             │
│
├── frontend/                      ← 第 3 层
│   ├── audio_frontend.c           │  MFCC 滑动窗口（CCMRAM）
│   ├── audio_frontend.h           │
│   ├── audio_params.h             │  采样率/窗长/步长/量化参数
│   └── mel_filterbank_data.h      │  40 通道 Mel 滤波器组系数表
│
└── kws/                           ← 第 5 层
    ├── kws_postprocess.c          │  消抖 + 冷却 + score 阈值 + softmax
    └── kws_postprocess.h          │
```

`X-CUBE-AI/App/app_x-cube-ai.c` 保持原位（CubeMX 约束），但只保留模型 IO 模板和顶层调度，KWS 逻辑全部走 `kws_postprocess_feed()` 调用。

对应的 CMakeLists.txt：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    # 音频前端预处理（MFCC / 滑动窗口特征）
    ${CMAKE_SOURCE_DIR}/audio/frontend/audio_frontend.c
    # 音频流采集（I2S DMA + 环形缓冲）
    ${CMAKE_SOURCE_DIR}/audio/capture/audio_capture_ring_buff.c
    ${CMAKE_SOURCE_DIR}/audio/capture/audio_stream.c
    # KWS 后处理（消抖 + 冷却）
    ${CMAKE_SOURCE_DIR}/audio/kws/kws_postprocess.c
    ...
)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    ...
    # 音频模块（采集 / 前端 / KWS）
    ${CMAKE_SOURCE_DIR}/audio/capture
    ${CMAKE_SOURCE_DIR}/audio/frontend
    ${CMAKE_SOURCE_DIR}/audio/kws
    ...
)
```

> [!TIP]
> 模块化分层的好处不只是"代码好看"。这次重构后，我后来想试 `yes/no` 二分类的小模型，**只改了 `app_x-cube-ai.c` 里的 `NUM_LABELS` 和 `kLabelNames`**，capture 和 frontend 完全不动。模块边界清晰后，迭代速度明显快了。

---

## 八、性能与延迟分析

### 8.1 流式 vs 直接输入的延迟差

很多读者问：流式推理比直接喂整段录音慢多少？实测分解：

| 阶段             | 延迟           | 说明                                                |
| ---------------- | -------------- | --------------------------------------------------- |
| DMA 半缓冲       | 10 ms          | 160 样本 / 16 kHz                                   |
| 滑动窗口等待     | 0~200 ms       | SLIDE_SAMPLES=3200（200 ms）；首次还需填满 1 秒窗口 |
| MFCC 特征提取    | 30~50 ms       | CMSIS-DSP RFFT Q15 + Mel 滤波                       |
| int8 推理        | ~50 ms         | micro_speech 模型                                   |
| KWS N-in-M 消抖  | +200~600 ms    | 需要 3/5 命中才确认                                 |
| **端到端总滞后** | **500~850 ms** |                                                     |

直接喂整段录音测试时只有"推理 50 ms"，所以**流式比直接测试慢约 10~17 倍**。主要消耗在滑动窗口填充和 KWS 消抖上——这是流式 KWS 的**固有代价**：用滞后换稳定性。

### 8.2 优化机会

如果觉得 800 ms 太慢，可以调：

- **`SLIDE_COLS` 从 10 改到 5**：滑动间隔 200 ms → 100 ms，但 CPU 开销翻倍
- **`KWS_REQUIRED_N` 从 3 改到 2**：消抖更宽松，但误报增多
- **`KWS_COOLDOWN` 从 5 改到 3**：冷却更短，但同一词可能连发
- **换更小的模型**：SVDF 或 DS-CNN 的精简版，推理 20 ms 以内

工程永远是 trade-off。

### 8.3 编译选项

整个工程用 `-O3 -flto -funroll-loops` 全局优化，链接 `-u _printf_float` 启用浮点 printf：

```cmake
add_compile_options(
    -O3                  # 最高优化级别
    -flto                # 链接时优化（跨模块内联）
    -funroll-loops       # 显式循环展开
)
target_link_options(${CMAKE_PROJECT_NAME} PRIVATE -flto -u _printf_float)
```

注意 `-O3 -flto` 也是 5.2 节 volatile 坑的"元凶"——优化越激进，对可见性要求越严格。但只要正确加了 `volatile`，激进优化是好事，MFCC 预处理时间从 -O2 的 60 ms 降到 -O3 的 35 ms。

---

## 九、踩坑总结表

把六个坑汇总成一张表，方便回顾：

| #   | 坑名                   | 症状                               | 根因                                          | 修复                                          |
| --- | ---------------------- | ---------------------------------- | --------------------------------------------- | --------------------------------------------- |
| 1   | stride bug             | 识别全错，score 14~30              | STM32F4 16B_EXTENDED 每帧 2 halfwords，不是 4 | `i*4+2` → `i*2+idx`，DMA size 减半            |
| 2   | volatile 缺失          | 删 printf 系统死机，加 printf 复活 | `-O3 -flto` 缓存 ISR 共享变量到寄存器         | `availableSamples` 加 `volatile`              |
| 3   | 栈溢出                 | HardFault / 看门狗复位             | MFCC 大数组放栈上，2 KB 栈不够                | 数组改 `static`，栈 2 KB → 8 KB               |
| 4   | DWT CYCCNT 失效        | 计数器读出固定值                   | CMSIS-DSP 内部禁用了 CYCCNTENA                | 改用 TIM5 (32-bit, 84 MHz)                    |
| 5   | KWS 误报               | 噪音识别成 up，连续触发            | per-word 冷却被中间词重置 + score 重叠        | 全局冷却 + N-in-M + score 阈值 + 过滤 silence |
| 6   | kernel_util.cpp 链接错 | `undefined reference`              | TFLM 漏了 `array.cc` 隐藏依赖                 | CMakeLists 显式加 `tensorflow/lite/array.cc`  |

---

## 十、经验沉淀

写完这个工程，有几条经验值得记下来：

1. **移植代码永远不要假设"应该一样"**。即使是同一家厂商（ST）的同名外设（I2S），不同芯片代际的细节也可能不同。手册优先于记忆。

2. **原子操作 ≠ 可见性**。LDREX/STREX 解决原子性，`volatile` 解决可见性，两者正交。bare-metal 上自己写并发原语时，**两个都要**。Linux 内核的 `atomic_t` 之所以不用加 volatile，是因为内部封装了 memory barrier。

3. **printf 能"修"bug，说明有可见性问题**。如果加一行无关 printf 让 bug 消失或出现，第一时间怀疑编译器优化 + 共享变量可见性，而不是怀疑 printf 本身。

4. **大数组默认 `static`**。嵌入式 C 的黄金法则，能避免 80% 的栈溢出问题。

5. **不要相信"标准"性能测量**。DWT、SysTick、PMU 这些"标准"计数器，在引入第三方库后可能被静默修改。硬件定时器（TIMx）是独立外设，最可靠。

6. **模型量化损失无法用软件完全弥补**。int8 量化后某些类的 score 天生偏低（如本文的 `up`），与噪音误报重叠。KWS 后处理只能降低误报频率，不能根治。要根治得换模型或加 fine-tune 负样本。

7. **模块化不是为了好看，是为了可替换**。本次重构后，想换模型只需改 `app_x-cube-ai.c`，采集和前端完全不动。如果将来要支持 PDM 麦克风，只改 `audio/capture/`，前端和 KWS 不动。**模块边界 = 替换边界**。

8. **CubeMX 生成文件不移动**。`app_x-cube-ai.c` 这种 CubeMX 模板文件，移动会破坏 CubeMX 重新生成能力。保持原位，把业务逻辑拆到独立模块调用即可。

---

## 结语

这个工程从"能跑但乱识别"到"流式稳定识别"，前前后后踩了六个坑，每个坑都是一两个小时。但回头看，每个坑都教会了我一条**底层原理**——I2S 帧布局、内存可见性、栈管理、外设冲突、模型量化、依赖管理。这些原理换芯片、换工程都用得上。

写这篇文章的初衷，就是希望后来人能少绕几圈。如果按本文的架构和注意事项来，从零搭一个 STM32F4 + TFLM 流式 KWS，应该半天就能跑通。

完整工程文件结构：

```
STM32F407VGT6_TensorFlow-Lite-Micro/
├── audio/                  # ← 模块化音频链路
│   ├── capture/            #   I2S DMA + 环形缓冲
│   ├── frontend/           #   MFCC 滑动窗口
│   └── kws/                #   KWS 后处理
├── X-CUBE-AI/App/          # ← CubeMX 模板（推理入口）
├── Core/                   # ← CubeMX 生成
├── Drivers/                # ← HAL + CMSIS
├── stm32f4_tflm_project/   # ← TFLM 源码
└── CMakeLists.txt          # ← 构建配置
```

感谢阅读。如果踩过相同的坑，欢迎交流；如果正在踩坑，希望本文能给你一点启发。
