---
title: "STM32F407 汇编啟動檔案深度解析：從復位向量到 C 語言 main() 的完整旅程"
date: 2026-07-11
description: "逐行解析 startup_stm32f407xx.s 汇编啟動檔案，涵蓋處理器指令集宣告、向量表結構、Reset_Handler 啟動流程（棧指標初始化→.data 搬運→.bss 清零→C 庫初始化→main 呼叫）、弱符號中斷機制與 .thumb_set 別名技術"
image: STM32F407汇编启动文件解析.png
categories:
  - "嵌入式"
  - "啟動流程"
tags:
  - "STM32"
  - "汇编"
  - "啟動檔案"
  - "Cortex-M4"
  - "向量表"
  - "GCC"
---

## 前言

每一個 STM32 專案中都有一份 `startup_stm32f407xx.s`，它安靜地躺在工程根目錄，平時幾乎不需要修改。但如果你曾好奇：**晶片上電後第一條指令從哪裡來？全域變數是誰初始化的？中斷服務函式是如何被「註冊」的？**——答案全在這份檔案裡。

本文將以 GCC 版 `startup_stm32f407xx.s` 為藍本，逐段拆解其運作機制，揭開從硬體復位到 C 語言 `main()` 函式執行的完整旅程。

> [!NOTE]
> 本文分析的啟動檔案來自 STM32CubeF4 HAL 庫，適用於 GCC/ARM 工具鏈。Keil 版（ARMASM 語法）和 IAR 版在結構上類似，但汇编語法不同。

---

## 一、檔案總體結構

啟動檔案可以分為四個邏輯區域：

| 區域 | 功能 | 對應行範圍 |
|------|------|-----------|
| 頭部宣告 | 處理器架構、指令集、FPU 類型 | 1-31 行 |
| 鏈結器符號 | .data / .bss 段地址引用 | 35-46 行 |
| Reset_Handler | 啟動初始化核心流程 | 57-102 行 |
| 向量表 + 弱符號 | 中斷向量表與預設處理函式 | 111-508 行 |

---

## 二、頭部宣告：告訴汇编器我們是誰

```asm
.syntax unified
.cpu cortex-m4
.fpu softvfp
.thumb
```

逐行解析：

| 指令 | 含義 | 為什麼需要 |
|------|------|-----------|
| `.syntax unified` | 使用統一汇编語法（ARM + Thumb 指令共用同一套語法） | GCC 預設使用 `divided` 語法，unified 更現代、更簡潔 |
| `.cpu cortex-m4` | 目標處理器為 Cortex-M4 | 影響指令編碼和可用指令集，Cortex-M4 支援 DSP 指令和可選 FPU |
| `.fpu softvfp` | 使用軟體浮點呼叫約定 | 即使硬體 FPU 存在，軟體浮點 ABI 也可確保與軟浮點庫的相容性；切換為 `fpv4-sp-d16` 可啟用硬體浮點 |
| `.thumb` | 使用 Thumb 指令集 | Cortex-M 系列**只支援** Thumb 指令集（16/32 位元混合編碼），這是硬體強制要求 |

> [!TIP]
> 如果你的專案需要使用硬體 FPU（浮點運算單元），需同時修改 `.fpu fpv4-sp-d16` 和鏈結器標誌 `-mfloat-abi=hard`。僅修改啟動檔案是不夠的——編譯器、汇编器和鏈結器必須保持 ABI 一致。

接下來宣告兩個全域符號：

```asm
.global  g_pfnVectors
.global  Default_Handler
```

- `g_pfnVectors`：向量表的起始標籤，鏈結器需要知道它的地址來放置中斷向量
- `Default_Handler`：預設中斷處理函式，所有未實現的中斷都指向它

---

## 三、鏈結器符號：.data 與 .bss 的地址錨點

```asm
.word  _sidata    /* .data 初始值在 Flash 中的起始地址 */
.word  _sdata     /* .data 段在 SRAM 中的起始地址 */
.word  _edata     /* .data 段在 SRAM 中的結束地址 */
.word  _sbss      /* .bss 段在 SRAM 中的起始地址 */
.word  _ebss      /* .bss 段在 SRAM 中的結束地址 */
```

這五個 `.word` 不是程式碼，而是**為 Reset_Handler 預留的資料**——它們透過標籤名引用鏈結器指令稿（`.ld` 檔案）中定義的符號。理解這些符號是理解啟動流程的關鍵。

### 3.1 為什麼 .data 需要搬運？

C 語言中帶初始值的全域變數（如 `int x = 42;`）存放在 `.data` 段。但 `.data` 有兩個「家」：

| 位置 | 內容 | 原因 |
|------|------|------|
| Flash（`_sidata`） | 初始值（42） | Flash 斷電不遺失，存放初始值 |
| SRAM（`_sdata` ~ `_edata`） | 執行時變數 | SRAM 可讀寫，程式執行時使用 |

上電時 SRAM 內容是隨機的，必須從 Flash 將初始值**搬運**到 SRAM，`.data` 段的變數才有正確的初始值。這就是 Reset_Handler 中「Copy Data」迴圈做的事。

### 3.2 為什麼 .bss 只需清零？

C 語言中未初始化的全域變數（如 `int y;`）存放在 `.bss` 段。C 標準規定未初始化的全域變數值為 0。`.bss` 段不佔用 Flash 空間（沒有初始值需要儲存），只需要在啟動時將 SRAM 中對應的區域清零即可。

> [!IMPORTANT]
> 這五個符號的名稱是**約定俗成的**，由鏈結器指令稿定義。如果使用自訂 `.ld` 檔案，必須確保這些符號名與啟動檔案中的引用一致。常見的命名變體包括 `__data_start` / `__data_end` 等。

---

## 四、Reset_Handler：啟動的核心流程

這是整份檔案最重要的部分——晶片上電後執行的第一段有意義的程式碼。

### 4.1 完整程式碼與逐行註解

```asm
    .section  .text.Reset_Handler
  .weak  Reset_Handler
  .type  Reset_Handler, %function
Reset_Handler:
  ldr   sp, =_estack       /* [1] 設定棧指標 */

  bl  SystemInit            /* [2] 呼叫系統初始化 */

  /* [3] 將 .data 段從 Flash 搬運到 SRAM */
  ldr r0, =_sdata           /*   r0 = SRAM 中 .data 起始地址 */
  ldr r1, =_edata           /*   r1 = SRAM 中 .data 結束地址 */
  ldr r2, =_sidata          /*   r2 = Flash 中 .data 初始值起始地址 */
  movs r3, #0               /*   r3 = 偏移量，初始化為 0 */
  b LoopCopyDataInit        /*   跳轉到迴圈判斷 */

CopyDataInit:
  ldr r4, [r2, r3]          /*   從 Flash[r2+r3] 讀取 4 位元組到 r4 */
  str r4, [r0, r3]          /*   將 r4 寫入 SRAM[r0+r3] */
  adds r3, r3, #4           /*   偏移量 += 4（按字搬運） */

LoopCopyDataInit:
  adds r4, r0, r3           /*   r4 = 當前寫入地址 = _sdata + offset */
  cmp r4, r1                /*   當前地址 < _edata ? */
  bcc CopyDataInit          /*   是：繼續搬運（bcc = 無符號小於則跳轉） */

  /* [4] 將 .bss 段清零 */
  ldr r2, =_sbss            /*   r2 = .bss 起始地址 */
  ldr r4, =_ebss            /*   r4 = .bss 結束地址 */
  movs r3, #0               /*   r3 = 0 */
  b LoopFillZerobss         /*   跳轉到迴圈判斷 */

FillZerobss:
  str  r3, [r2]             /*   將 0 寫入 [r2] */
  adds r2, r2, #4           /*   地址 += 4 */

LoopFillZerobss:
  cmp r2, r4                /*   當前地址 < _ebss ? */
  bcc FillZerobss           /*   是：繼續清零 */

  /* [5] 呼叫 C 庫初始化 */
  bl __libc_init_array

  /* [6] 呼叫 main() */
  bl  main
  bx  lr                    /*   如果 main() 返回，跳回呼叫者 */
.size  Reset_Handler, .-Reset_Handler
```

### 4.2 六個步驟詳解

#### 步驟 [1]：設定棧指標

```asm
ldr   sp, =_estack
```

`_estack` 是棧頂地址，定義在鏈結器指令稿中（通常是 SRAM 末尾，如 `0x20020000`）。Cortex-M4 的棧是**滿遞減**（Full Descending）——壓棧時 SP 先減後寫。因此棧頂地址指向 SRAM 最高地址 +1（即 SRAM 末尾的下一個位元組）。

> [!NOTE]
> Cortex-M4 在復位時硬體會自動從向量表偏移 0 處載入 SP（即 `_estack`），所以這行程式碼看似冗餘。但它是防禦性程式設計：確保 SP 值正確，即使向量表配置有誤。

#### 步驟 [2]：呼叫 SystemInit

```asm
bl  SystemInit
```

`SystemInit()` 通常在 `system_stm32f4xx.c` 中定義，負責：

- 配置 Flash 等待週期（LATENCY）
- 啟用 PLL 並切換系統時脈到最高頻率（STM32F407 通常為 168MHz）
- 配置 AHB/APB1/APB2 分頻

`bl`（Branch with Link）指令在跳轉的同時將返回地址儲存到 `lr` 暫存器，函式返回時透過 `bx lr` 回到呼叫點。

#### 步驟 [3]：搬運 .data 段

這是啟動檔案中最精巧的程式碼，使用**偏移量定址**實現從 Flash 到 SRAM 的資料搬運：

| 暫存器 | 作用 | 值 |
|--------|------|---|
| r0 | SRAM 目標起始地址 | `_sdata` |
| r1 | SRAM 目標結束地址 | `_edata` |
| r2 | Flash 來源起始地址 | `_sidata` |
| r3 | 偏移量（每輪 +4） | 0 → 4 → 8 → ... |
| r4 | 臨時暫存器 | 當前操作地址/資料 |

核心迴圈邏輯：

```asm
CopyDataInit:
  ldr r4, [r2, r3]    /* 來源讀取：Flash[_sidata + offset] */
  str r4, [r0, r3]    /* 目標寫入：SRAM[_sdata + offset] */
  adds r3, r3, #4     /* 偏移量遞增 */
LoopCopyDataInit:
  adds r4, r0, r3     /* r4 = _sdata + offset = 當前寫入地址 */
  cmp r4, r1          /* 與 _edata 比較 */
  bcc CopyDataInit    /* 未達到結束地址則繼續 */
```

> [!TIP]
> 為什麼用偏移量 `[r2, r3]` 而非直接遞增指標？因為來源（Flash）和目標（SRAM）的基地址不同，但偏移量是相同的。用同一個偏移量 `r3` 同時索引來源和目標，程式碼最簡潔，只需一個計數器。

#### 步驟 [4]：清零 .bss 段

與 .data 搬運類似，但更簡單——不需要從 Flash 讀取資料，直接寫入 0：

```asm
FillZerobss:
  str  r3, [r2]       /* r3 = 0，寫入 [r2] */
  adds r2, r2, #4     /* 地址遞增 */
LoopFillZerobss:
  cmp r2, r4          /* 當前地址 < _ebss ? */
  bcc FillZerobss     /* 繼續清零 */
```

這裡直接遞增 `r2`（當前地址指標），與 .data 搬運的偏移量方式不同。因為 .bss 清零只有一個指標需要移動，不需要同時維護來源和目標兩個地址。

#### 步驟 [5]：呼叫 C 庫初始化

```asm
bl __libc_init_array
```

這一步容易被忽視，但對 C++ 和某些 C 程式碼至關重要。`__libc_init_array` 執行兩個關鍵操作：

1. **呼叫 `.init_array` 段中的函式指標**——C++ 全域物件的建構函式（`__libc_init_array` 內部先呼叫 `.preinit_array`，再呼叫 `.init_array`）
2. **呼叫 `__attribute__((constructor))` 修飾的函式**

> [!IMPORTANT]
> 如果你的專案是純 C 且沒有使用 `__attribute__((constructor))`，理論上可以跳過此步驟。但標準做法是始終呼叫，因為：
> - 部分 C 庫的初始化邏輯依賴它
> - 未來如果引入 C++ 程式碼，不會因遺漏而出錯
> - 呼叫開銷極小（`.init_array` 為空時幾乎是空操作）

#### 步驟 [6]：呼叫 main()

```asm
bl  main
bx  lr
```

經過以上所有準備，C 語言執行環境終於就緒：棧已設定、全域變數已初始化、C 庫已就緒。現在可以安全地呼叫 `main()` 了。

如果 `main()` 竟然返回了（嵌入式程式中通常不應該），`bx lr` 會跳回呼叫者——但此時呼叫者是 Reset_Handler 本身，`lr` 的值未定義，行為不可預測。在健壯的系統中，`main()` 應該是一個無限迴圈，永遠不返回。

### 4.3 啟動流程全景圖

```
上電/復位
  │
  ▼
硬體自動從向量表載入 SP = _estack  ← 向量表偏移 0
硬體自動從向量表載入 PC = Reset_Handler  ← 向量表偏移 4
  │
  ▼
Reset_Handler 執行
  ├─ [1] ldr sp, =_estack          重新設定棧指標（防禦性）
  ├─ [2] bl SystemInit             配置系統時脈
  ├─ [3] Copy .data: Flash → SRAM  初始化全域變數
  ├─ [4] Zero .bss: SRAM 清零      初始化未賦初值的全域變數
  ├─ [5] bl __libc_init_array      C++ 建構函式 / constructor
  └─ [6] bl main                   進入使用者程式
```

---

## 五、Default_Handler：未預期中斷的安全網

```asm
    .section  .text.Default_Handler,"ax",%progbits
Default_Handler:
Infinite_Loop:
  b  Infinite_Loop
  .size  Default_Handler, .-Default_Handler
```

如果 CPU 進入了一個未實現的中斷處理函式，就會執行 `Default_Handler`——一個無限迴圈。它的目的是**讓程式停在原地**，方便除錯器捕獲現場：

- 中斷返回地址（`lr`）仍然儲存在暫存器中
- 除錯器可以看到程式卡在 `Default_Handler`，立即知道發生了未處理中斷
- 暫存器和棧的內容未被破壞，可以分析中斷來源

`.section` 屬性說明：

| 屬性 | 含義 |
|------|------|
| `"ax"` | allocatable + executable，可分配且可執行 |
| `%progbits` | 段包含實際資料（而非 BSS 式的零填充） |

> [!WARNING]
> 在生產環境中，`Default_Handler` 的無限迴圈會導致裝置「假死」——看起來執行但不再回應。更好的做法是在 `Default_Handler` 中執行軟復位（`NVIC_SystemReset()`），或至少透過日誌輸出異常資訊。

---

## 六、向量表：中斷的路由表

### 6.1 向量表結構

向量表是 Cortex-M4 啟動機制的核心，它必須放置在 Flash 的起始地址（`0x08000000`）。

```asm
   .section  .isr_vector,"a",%progbits
  .type  g_pfnVectors, %object

g_pfnVectors:
  .word  _estack               /* 偏移 0x00: 棧頂地址 */
  .word  Reset_Handler         /* 偏移 0x04: 復位向量 */
  .word  NMI_Handler           /* 偏移 0x08: 不可遮蔽中斷 */
  .word  HardFault_Handler     /* 偏移 0x0C: 硬體錯誤 */
  .word  MemManage_Handler     /* 偏移 0x10: 記憶體管理錯誤 */
  .word  BusFault_Handler      /* 偏移 0x14: 匯流排錯誤 */
  .word  UsageFault_Handler    /* 偏移 0x18: 用法錯誤 */
  .word  0                     /* 偏移 0x1C: 保留 */
  .word  0                     /* 偏移 0x20: 保留 */
  .word  0                     /* 偏移 0x24: 保留 */
  .word  0                     /* 偏移 0x28: 保留 */
  .word  SVC_Handler           /* 偏移 0x2C: 系統服務呼叫 */
  .word  DebugMon_Handler      /* 偏移 0x30: 除錯監控 */
  .word  0                     /* 偏移 0x34: 保留 */
  .word  PendSV_Handler        /* 偏移 0x38: PendSV */
  .word  SysTick_Handler       /* 偏移 0x3C: SysTick */
  /* 外部中斷從偏移 0x40 開始... */
```

### 6.2 Cortex-M4 異常向量詳解

前 16 個向量是 ARM Cortex-M4 架構定義的**系統異常**，所有 M4 晶片都一樣：

| 偏移 | 異常號 | 名稱 | 優先級 | 典型觸發場景 |
|------|--------|------|--------|-------------|
| 0x00 | - | 初始 SP | - | 硬體自動載入 |
| 0x04 | 1 | Reset | -3（最高） | 上電/復位 |
| 0x08 | 2 | NMI | -2 | 外部 NMI 引腳 / 看門狗 |
| 0x0C | 3 | HardFault | -1 | 其他異常無法處理時的兜底 |
| 0x10 | 4 | MemManage | 可配置 | 存取 MPU 保護區域 |
| 0x14 | 5 | BusFault | 可配置 | 存取無效地址（如未對映外設） |
| 0x18 | 6 | UsageFault | 可配置 | 執行未定義指令 / 除零 |
| 0x2C | 11 | SVCall | 可配置 | 執行 `SVC` 指令（RTOS 系統呼叫） |
| 0x30 | 12 | Debug Monitor | 可配置 | 除錯事件 |
| 0x38 | 14 | PendSV | 可配置 | RTOS 上下文切換 |
| 0x3C | 15 | SysTick | 可配置 | SysTick 計時器溢位 |

> [!NOTE]
> 優先級數值越小優先級越高。Reset（-3）、NMI（-2）、HardFault（-1）是固定的，不可配置。其他異常的優先級可透過 NVIC 暫存器程式設計。

### 6.3 STM32F407 外部中斷向量

從偏移 0x40 開始是晶片特定的外部中斷，STM32F407 共有 82 個：

| 異常號 | 中斷名 | 外設 | 說明 |
|--------|--------|------|------|
| 16 | WWDG_IRQHandler | WWDG | 視窗看門狗 |
| 17 | PVD_IRQHandler | PVD | 電源電壓偵測 |
| 18 | TAMP_STAMP_IRQHandler | RTC | 侵入與時間戳 |
| ... | ... | ... | ... |
| 25 | EXTI0_IRQHandler | EXTI | 外部中斷線 0 |
| ... | ... | ... | ... |
| 54 | USART1_IRQHandler | USART1 | 序列埠 1 |
| 67 | ETH_IRQHandler | Ethernet | 乙太網路 |
| 97 | FPU_IRQHandler | FPU | 浮點單元 |

部分位置填 0（保留位），如偏移 0x1C-0x28 和 CRYP 位置。寫入保留位的 0 確保向量表大小和偏移正確。

### 6.4 向量表的硬體載入機制

Cortex-M4 復位時的硬體行為（純硬體，無需軟體參與）：

1. 從地址 `0x00000000`（對映到 `0x08000000`）讀取 4 位元組 → 載入到 **SP**（主棧指標）
2. 從地址 `0x00000004`（對映到 `0x08000000+4`）讀取 4 位元組 → 載入到 **PC**（程式計數器）
3. CPU 開始從 PC 指向的地址執行程式碼——即 `Reset_Handler`

> [!TIP]
> STM32 的 Flash 起始地址 `0x08000000` 會被別名對映到 `0x00000000`（透過 BOOT 接腳配置）。這就是為什麼向量表放在 Flash 開頭，但硬體從 `0x00000000` 讀取——它們指向同一塊實體儲存。

---

## 七、弱符號機制：中斷處理函式的「佔位符」

### 7.1 .weak + .thumb_set 的運作原理

啟動檔案的最後部分是一長串重複的模式：

```asm
   .weak      NMI_Handler
   .thumb_set NMI_Handler,Default_Handler

   .weak      HardFault_Handler
   .thumb_set HardFault_Handler,Default_Handler

   /* ... 80+ 個中斷 ... */

   .weak      FPU_IRQHandler
   .thumb_set FPU_IRQHandler,Default_Handler
```

這兩行指令構成了一套精巧的**預設值機制**：

| 指令 | 含義 |
|------|------|
| `.weak NMI_Handler` | 宣告 `NMI_Handler` 為弱符號——如果其他檔案定義了同名符號，弱定義被覆蓋 |
| `.thumb_set NMI_Handler, Default_Handler` | 將 `NMI_Handler` 的值設為 `Default_Handler` 的地址（Thumb 函式別名） |

### 7.2 弱符號的鏈結行為

```
場景 1：使用者未實現 NMI_Handler
  → 鏈結器使用弱定義 → NMI_Handler = Default_Handler → 死迴圈

場景 2：使用者在 stm32f4xx_it.c 中實現了 NMI_Handler
  → 鏈結器選擇強定義 → NMI_Handler = 使用者函式 → 正常執行
```

這是嵌入式開發中最常用的中斷註冊方式——**無需修改啟動檔案，只需在 C 程式碼中定義同名函式即可**：

```c
// stm32f4xx_it.c
void NMI_Handler(void) {
    // 使用者自訂的 NMI 處理邏輯
}
```

鏈結器看到強符號 `NMI_Handler`，自動忽略啟動檔案中的弱定義，向量表中的對應條目自動指向使用者函式。

### 7.3 .thumb_set vs .set

`.thumb_set` 是 `.set` 的 Thumb 專用變體，它會自動設定地址的 bit[0] = 1（Thumb 間跳轉標誌）：

| 指令 | 地址 bit[0] | 用途 |
|------|-------------|------|
| `.set` | 不修改 | ARM 模式（Cortex-M 不適用） |
| `.thumb_set` | 自動置 1 | Thumb 模式（Cortex-M 必須使用） |

Cortex-M4 只支援 Thumb 指令，中斷向量中的地址 bit[0] 必須為 1，否則觸發 HardFault。使用 `.thumb_set` 確保了這一點。

> [!IMPORTANT]
> 如果你手動撰寫汇编中斷處理函式，必須確保函式符號的 bit[0] 為 1。GCC 編譯的 C 函式會自動處理，但手寫汇编需要用 `.thumb_func` 宣告或 `.thumb_set` 設定。

---

## 八、啟動檔案與鏈結器指令稿的協作

啟動檔案不是獨立運作的，它依賴鏈結器指令稿（`.ld`）定義的符號，也依賴 C 執行時提供的函式。完整的依賴關係：

| 啟動檔案中的引用 | 定義位置 | 類型 |
|-----------------|---------|------|
| `_estack` | 鏈結器指令稿 | 棧頂地址 |
| `_sidata`, `_sdata`, `_edata` | 鏈結器指令稿 | .data 段地址 |
| `_sbss`, `_ebss` | 鏈結器指令稿 | .bss 段地址 |
| `SystemInit` | `system_stm32f4xx.c` | 時脈初始化函式 |
| `__libc_init_array` | C 執行時庫 (libc) | 建構函式呼叫 |
| `main` | 使用者程式碼 | 程式入口 |

### 8.1 鏈結器指令稿關鍵片段

```ld
/* 典型的 STM32F407 鏈結器指令稿 */
MEMORY {
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 1024K
  RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS {
  .isr_vector : {
    . = ALIGN(4);
    KEEP(*(.isr_vector))    /* 向量表，不能被最佳化刪除 */
    . = ALIGN(4);
  } > FLASH

  .text : {
    *(.text*)
    *(.rodata*)
  } > FLASH

  _sidata = LOADADDR(.data);  /* .data 在 Flash 中的載入地址 */

  .data : {
    _sdata = .;
    *(.data*)
    _edata = .;
  } > RAM AT > FLASH          /* 執行在 RAM，載入在 Flash */

  .bss : {
    _sbss = .;
    *(.bss*)
    *(COMMON)
    _ebss = .;
  } > RAM

  _estack = ORIGIN(RAM) + LENGTH(RAM);  /* 棧頂 = RAM 末尾 */
}
```

> [!NOTE]
> `.data : { ... } > RAM AT > FLASH` 是關鍵語法——表示 `.data` 段的**執行地址**在 RAM（`> RAM`），但**載入地址**在 Flash（`AT > FLASH`）。`LOADADDR(.data)` 取的是載入地址（即 Flash 中的位置），這就是 `_sidata` 的值。

---

## 九、常見問題與排錯

### 9.1 HardFault 死迴圈

**現象**：程式卡在 `Default_Handler`（實際是 `HardFault_Handler`）。

**排查步驟**：

1. 在除錯器中暫停，檢視 `PC` 和 `LR` 暫存器
2. 檢視 `SCB->HFSR`、`SCB->CFSR`、`SCB->MMFAR`/`SCB->BFAR` 暫存器
3. 常見原因：
   - 空指標解引用（`SCB->MMFAR` 指向非法地址）
   - 棧溢位（SP 超出 SRAM 範圍）
   - 向量表中函式地址的 bit[0] = 0（非 Thumb 地址）

### 9.2 全域變數初始值錯誤

**現象**：帶初始值的全域變數值不對，或隨重啟變化。

**原因**：Reset_Handler 中的 .data 搬運被跳過或執行不正確。

**排查**：

- 檢查鏈結器指令稿是否正確定義了 `_sidata`、`_sdata`、`_edata`
- 檢查是否有自訂的 Reset_Handler 覆蓋了弱符號但未實現搬運邏輯
- 確認向量表 `.isr_vector` 段使用了 `KEEP()` 防止被最佳化

### 9.3 C++ 全域物件未建構

**現象**：C++ 全域物件的建構函式未被呼叫。

**原因**：`__libc_init_array` 被遺漏。某些精簡的啟動檔案（或自訂 Reset_Handler）可能沒有呼叫它。

### 9.4 中斷處理函式不生效

**現象**：定義了中斷處理函式但未被呼叫，總是進入死迴圈。

**排查**：

- 確認 C 檔案中的函式名與啟動檔案中的弱符號名**完全一致**（區分大小寫）
- 確認編譯時包含了 `stm32f4xx_it.c` 檔案
- 檢查 NVIC 中斷使能是否已配置

---

## 十、啟動檔案的修改指南

### 10.1 何時需要修改啟動檔案

| 場景 | 修改內容 |
|------|---------|
| 使用外部 SDRAM 擴充記憶體 | 在 Reset_Handler 中新增 SDRAM 初始化程式碼 |
| 修改堆疊大小 | 修改鏈結器指令稿中的 `_estack` 或增加 `_Min_Heap_Size`/`_Min_Stack_Size` |
| 新增自訂中斷 | 向量表中一般無需修改——弱符號機制自動處理 |
| Bootloader 跳轉場景 | 調整向量表偏移（`SCB->VTOR`），啟動檔案本身通常不需要改 |
| 使用硬體 FPU | 修改 `.fpu` 為 `fpv4-sp-d16`，並在 SystemInit 中啟用 FPU |

### 10.2 新增自訂復位初始化邏輯

如果需要在進入 `main()` 之前執行額外的初始化（如看門狗餵狗、外設早期配置），有兩種方式：

**方式一：修改 SystemInit（推薦）**

在 `system_stm32f4xx.c` 的 `SystemInit()` 中新增程式碼，無需修改汇编檔案。

**方式二：在 Reset_Handler 中插入汇编程式碼**

在 `bl SystemInit` 之後、`bl main` 之前插入自訂汇编呼叫。這種方式需要理解 ARM 呼叫約定（AAPCS），確保不破壞暫存器狀態。

---

## 十一、總結

`startup_stm32f407xx.s` 雖然只有約 500 行，卻承載了從硬體復位到 C 語言執行環境建立的全部責任。核心要點：

1. **頭部宣告**確定了目標架構和指令集——Cortex-M4 只支援 Thumb，這是硬體強制約束
2. **鏈結器符號**是啟動檔案與鏈結器指令稿的橋樑——`.data` 搬運和 `.bss` 清零依賴這些符號
3. **Reset_Handler** 是啟動流程的核心——SP 初始化 → 時脈配置 → 資料搬運 → BSS 清零 → C 庫初始化 → main()
4. **弱符號機制**使得中斷註冊無需修改啟動檔案——只需在 C 程式碼中定義同名函式
5. **向量表**是硬體與軟體的契約——復位時硬體自動載入 SP 和 PC，後續中斷也透過它路由

理解啟動檔案，就理解了嵌入式系統「從零到 main()」的全部過程。

---

## 參考資料

- [ARM Cortex-M4 Technical Reference Manual](https://developer.arm.com/documentation/ddi0439/latest/)
- [STM32F4xx Reference Manual (RM0090)](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [ARM Architecture Procedure Call Standard (AAPCS)](https://github.com/ARM-software/abi-aa/blob/main/aapcs32/aapcs32.rst)
- [GNU Assembler Manual](https://sourceware.org/binutils/docs/as/)
- [Cortex-M4 Devices Generic User Guide](https://developer.arm.com/documentation/dui0553/latest/)
