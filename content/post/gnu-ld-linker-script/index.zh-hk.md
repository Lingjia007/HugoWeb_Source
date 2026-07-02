---
title: "GNU LD 鏈結腳本詳解：從 STM32F407 實例剖析嵌入式記憶體佈局與段重定位"
date: 2026-07-02
description: "以 STM32F407VGT6 的 GCC 鏈結腳本為實例，深入剖析 GNU LD 的 MEMORY 命令、SECTIONS 段分配、VMA/LMA 位址模型、啟動程式碼與鏈結腳本的協作、CCMRAM 使用、TLS 支援等核心技術"
categories:
  - "嵌入式"
tags:
  - "STM32"
  - "鏈結腳本"
  - "GNU-LD"
  - "GCC"
  - "嵌入式"
math: true
---

## 前言

鏈結腳本（Linker Script, `.ld`）是嵌入式開發中最常被忽視、卻又最關鍵的檔案之一。它告訴鏈結器：程式的程式碼和資料應該放在記憶體的什麼位置、各個段的排列順序如何、啟動時需要從 Flash 拷貝到 RAM 的資料有哪些。

本文以一個真實專案的 STM32F407VGT6 鏈結腳本為例，逐行剖析 GNU LD 鏈結腳本的每一個語法要素，並深入講解 VMA/LMA 位址模型、啟動程式碼與鏈結腳本的協作機制。

---

## 一、實例專案背景

本實例來自一個運行在 **STM32F407VGT6** 上的 TensorFlow Lite Micro 端側 AI 推理專案，使用 CMake + Ninja + GCC Arm Embedded 交叉編譯。

晶片資源：

| 資源 | 大小 | 起始位址 |
|------|------|----------|
| Flash | 1024 KB | `0x08000000` |
| SRAM | 112 KB | `0x20000000` |
| CCMRAM | 64 KB | `0x10000000` |

---

## 二、鏈結腳本整體結構

一個典型的 GNU LD 鏈結腳本由三大部分組成：

```
ENTRY(...)         ← 入口點定義

MEMORY { ... }     ← 記憶體區域宣告

SECTIONS { ... }   ← 段分配規則
```

下面逐一剖析。

---

## 三、ENTRY：程式入口點

```ld
ENTRY(Reset_Handler)
```

`ENTRY` 命令告訴鏈結器程式的入口符號是 `Reset_Handler`。這個值會被寫入 ELF 檔案的入口點欄位，除錯器和模擬器據此知道從哪裡開始執行。

**在嵌入式裸機中**，真正決定執行起始位置的是中斷向量表（`.isr_vector` 段）中的堆疊指標和 ResetHandler，它們被放在 Flash 的最前面（`0x08000000`），CPU 上電後從這裡取指。`ENTRY` 主要影響 ELF 元資料和除錯體驗。

---

## 四、MEMORY：記憶體區域宣告

```ld
MEMORY
{
RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 112K
CCMRAM (xrw)   : ORIGIN = 0x10000000, LENGTH = 64K
FLASH (rx)      : ORIGIN = 0x08000000, LENGTH = 1024K
}
```

### 4.1 語法

```
<名稱> (<屬性>) : ORIGIN = <起始位址>, LENGTH = <大小>
```

### 4.2 屬性字母

| 屬性 | 含義 | 可放入的段 |
|------|------|-----------|
| `r` | Read | 唯讀資料（.rodata） |
| `w` | Write | 可寫資料（.data, .bss） |
| `x` | eXecute | 可執行程式碼（.text） |
| `a` | Alloc | 可分配（預設所有段都有） |
| `i` | Init | 已初始化段 |

本實例中：
- **FLASH (rx)**：唯讀 + 可執行，存放程式碼和常數
- **RAM (xrw)**：可讀 + 可寫 + 可執行，存放已初始化資料和 BSS
- **CCMRAM (xrw)**：Core Coupled Memory，與 CPU 內核直接耦合，零等待存取

### 4.3 CCMRAM 的特殊性

STM32F407 的 CCMRAM 是一塊**僅 CPU 可存取**的 64KB SRAM，DMA 控制器無法存取。屬性設為 `xrw` 允許將程式碼放入 CCMRAM 執行（效能敏感的演算法可受益於零等待），但需注意 DMA 不可觸及。

### 4.4 位址空間佈局圖

| 位址範圍 | 區域 | 內容 |
|----------|------|------|
| `0x00000000` | Code Region | 透過 SYSBOOT 映射到 Flash |
| `0x08000000` - `0x080FFFFF` | **FLASH (1024 KB)** | `.isr_vector` → `.text` → `.rodata` → `.data (LMA)` → `.ccmram (LMA)` |
| `0x10000000` - `0x1000FFFF` | **CCMRAM (64 KB)** | `.ccmram (VMA)` — 啟動時從 Flash 拷貝 |
| `0x20000000` - `0x2001BFFF` | **RAM (112 KB)** | `.data (VMA)` → `.bss` → Heap → Stack（堆疊從高位址向低位址增長） |

**LMA vs VMA 拷貝關係**：

```
Flash (LMA)                RAM/CCMRAM (VMA)
|-- .data初始值    ─-----> .data執行時位置（拷貝）
|-- .ccmram初始值  ─-----> .ccmram執行時位置（拷貝）
|
.bss                        ─-----> 啟動時清零（不拷貝）
```

---

## 五、VMA 與 LMA：理解雙位址模型

這是鏈結腳本最核心的概念。

- **VMA (Virtual Memory Address)**：程式執行時的位址。CPU 執行時使用的位址。
- **LMA (Load Memory Address)**：程式被載入（燒錄）到的位址。Flash 中實際儲存的位置。

### 5.1 為什麼需要兩個位址？

對於 `.text` 和 `.rodata`，VMA = LMA，它們在 Flash 中執行，不需要搬運。

但對於 `.data`（已初始化全域變數），情況不同：

```c
int global_counter = 42;  // 已初始化全域變數
```

這個變數的**初始值 42** 必須儲存在 Flash 中（掉電不遺失），但**執行時**變數必須在 RAM 中（可讀寫）。因此：

- **LMA** = Flash 中的位址（存放初始值 42 的位置）
- **VMA** = RAM 中的位址（程式執行時讀寫 `global_counter` 的位置）

啟動程式碼的責任就是在 `main()` 之前，將 `.data` 段從 LMA 拷貝到 VMA。

### 5.2 AT> 語法

```ld
.data :
{
    ...
} >RAM AT> FLASH
```

- `>RAM`：指定 VMA 區域為 RAM
- `AT> FLASH`：指定 LMA 區域為 Flash

如果省略 `AT>`，則 LMA = VMA，段既存放又執行在同一區域。

### 5.3 形式化表示

對於段 $S$：

$$\text{VMA}(S) = \text{Addr}(S \text{ in target region})$$
$$\text{LMA}(S) = \text{Addr}(S \text{ in load region})$$

執行時不變量：

$$\forall x \in \text{.data}, \text{Mem}[\text{VMA}(x)] = \text{Mem}[\text{LMA}(x)] \text{ （啟動拷貝完成後）}$$

---

## 六、堆疊與堆積：鏈結時檢查

```ld
_estack = ORIGIN(RAM) + LENGTH(RAM);    /* RAM 末尾 */
_sstack = _estack - _Min_Stack_Size;
_Min_Heap_Size = 0x800;      /* 2 KB */
_Min_Stack_Size = 0x800;     /* 2 KB */
```

### 6.1 堆疊的生長方向

ARM Cortex-M 的堆疊是**滿遞減**（Full Descending）：

- 堆疊指標初始值 = RAM 最高位址（`_estack`）
- 每次 PUSH，SP 先減 4 再寫入
- 每次 POP，先讀出 SP 再加 4

```
RAM 高位址 ← _estack (SP 初始值)
              │ Stack │  ↓ 向低位址增長
              │       │
              │ Heap  │  ↑ 向高位址增長
RAM 低位址 ← _sdata
```

### 6.2 最小堆疊檢查

`._user_heap_stack` 段在鏈結時確保 RAM 中有足夠空間：

```ld
._user_heap_stack (NOLOAD) :
{
    . = ALIGN(8);
    PROVIDE ( end = . );
    PROVIDE ( _end = . );
    . = . + _Min_Heap_Size;    /* 預留 2KB 堆積 */
    . = . + _Min_Stack_Size;   /* 預留 2KB 堆疊 */
    . = ALIGN(8);
} >RAM
```

如果 `.bss` + heap + stack 超出 RAM 容量，鏈結器會報錯：

```
region RAM overflowed by 1234 bytes
```

> [!TIP]
> `_Min_Stack_Size = 0x800` (2KB) 通常是**最小值**，實際堆疊用量取決於呼叫深度和區域變數大小。對於使用 RTOS 或中斷巢狀的場景，可能需要 4KB 甚至更大。可用 `__stack_limit` 符號或 MPU 進行執行時堆疊溢出偵測。

---

## 七、SECTIONS：段分配詳解

### 7.1 中斷向量表 (.isr_vector)

```ld
.isr_vector :
{
    . = ALIGN(4);
    KEEP(*(.isr_vector))    /* 強制保留，不被 --gc-sections 最佳化掉 */
    . = ALIGN(4);
} >FLASH
```

**為什麼必須放在 Flash 最前面？**

STM32 上電後，CPU 從 `0x00000000` 開始取指。位址 `0x00000000` 透過 SYSBOOT 配置被映射（alias）到 Flash 起始位址 `0x08000000`。CPU 首先讀取：

| 偏移 | 內容 | 含義 |
|------|------|------|
| 0x00 | 初始 SP | 主堆疊指標初始值 |
| 0x04 | ResetHandler | 復位向量，跳轉到啟動程式碼 |
| 0x08 | NMIHandler | 不可遮罩中斷 |
| 0x0C | HardFaultHandler | 硬體錯誤 |
| ... | ... | 其他中斷/異常 |

**`KEEP` 的作用**：`--gc-sections` 最佳化會刪除未被引用的段，但 `.isr_vector` 中的向量表沒有顯式引用者（CPU 硬體自動跳轉）。`KEEP` 防止其被最佳化掉。

### 7.2 程式碼段 (.text)

```ld
.text :
{
    . = ALIGN(4);
    *(.text)           /* 所有 .text 段 */
    *(.text*)          /* .text.* 命名段 */
    *(.glue_7)         /* ARM → Thumb 樁程式碼 */
    *(.glue_7t)        /* Thumb → ARM 樁程式碼 */
    *(.eh_frame)       /* 異常處理幀（C++） */

    KEEP (*(.init))    /* 建構函式初始化 */
    KEEP (*(.fini))    /* 解構函式清理 */

    . = ALIGN(4);
    _etext = .;        /* 程式碼段結束符號 */
} >FLASH
```

**`.glue_7` / `.glue_7t`**：ARM 和 Thumb 指令集切換時的跳轉樁程式碼。Cortex-M4 僅支援 Thumb-2，但這些段在通用 ARM 工具鏈中仍會產生。

**`_etext`**：匯出符號，標記程式碼段末尾。啟動程式碼用它來確定 `.data` 段在 Flash 中的起始位置（`_sidata = LOADADDR(.data)` 更精確）。

### 7.3 唯讀資料段 (.rodata)

```ld
.rodata :
{
    . = ALIGN(4);
    *(.rodata)         /* const 全域變數、字串字面量 */
    *(.rodata*)        /* .rodata.* 命名段 */
    . = ALIGN(4);
} >FLASH
```

`.rodata` 存放 `const` 全域變數、字串字面量等。在嵌入式系統中，這些資料留在 Flash 中，不佔用 RAM。

**一個常見陷阱**：

```c
const char *msg = "hello";   // "hello" 在 .rodata，msg 指標在 .data
const char msg[] = "hello";  // 整個都在 .rodata，更優
```

### 7.4 C++ 相關段

```ld
.ARM.extab (READONLY) : { ... } >FLASH
.ARM (READONLY) : { ... } >FLASH
.preinit_array (READONLY) : { ... } >FLASH
.init_array (READONLY) : { ... } >FLASH
.fini_array (READONLY) : { ... } >FLASH
```

| 段 | 用途 |
|----|------|
| `.ARM.extab` / `.ARM` | C++ 異常處理的展開表 |
| `.preinit_array` | 最早期初始化函式指標陣列 |
| `.init_array` | C++ 全域建構函式指標陣列 |
| `.fini_array` | C++ 全域解構函式指標陣列 |

`READONLY` 關鍵字是 GCC 11+ 的特性，告訴鏈結器這些段不可寫。使用 GCC 10 或更早版本時需要移除。

**純 C 專案**：如果沒有 C++ 程式碼，這些段通常為空。但 `KEEP` 確保即使 `--gc-sections` 也不會誤刪。

### 7.5 CCMRAM 段

```ld
_siccmram = LOADADDR(.ccmram);

.ccmram :
{
    . = ALIGN(4);
    _sccmram = .;
    *(.ccmram)
    *(.ccmram*)
    . = ALIGN(4);
    _eccmram = .;
} >CCMRAM AT> FLASH
```

- `>CCMRAM`：VMA 在 CCMRAM 區域
- `AT> FLASH`：LMA 在 Flash（初始值儲存在 Flash 中）
- `_siccmram`：LMA 起始位址，啟動程式碼據此將資料從 Flash 拷貝到 CCMRAM

**使用方式**：在 C 程式碼中用 `__attribute__` 將變數放入 CCMRAM：

```c
__attribute__((section(".ccmram"))) int fast_buffer[256];
```

> [!WARNING]
> 啟動程式碼必須顯式處理 CCMRAM 的拷貝。STM32CubeMX 產生的預設 `startup_*.s` 可能不包含 CCMRAM 拷貝邏輯，需要手動新增。

### 7.6 已初始化資料段 (.data)

```ld
_sidata = LOADADDR(.data);   /* .data 在 Flash 中的起始位址 */

.data :
{
    . = ALIGN(4);
    _sdata = .;               /* VMA 起始 */
    *(.data)
    *(.data*)
    *(.RamFunc)               /* 在 RAM 中執行的函式 */
    *(.RamFunc*)
    . = ALIGN(4);
} >RAM AT> FLASH
```

**匯出符號**：

| 符號 | 含義 |
|------|------|
| `_sidata` | `.data` 的 LMA 起始（Flash 中） |
| `_sdata` | `.data` 的 VMA 起始（RAM 中） |
| `_edata` | `.data` 的 VMA 結束（在 .tdata 段末尾定義） |

**啟動拷貝偽程式碼**：

```c
// startup_stm32f407xx.s 中的邏輯
uint32_t *src = &_sidata;        // Flash 中的來源位址
uint32_t *dst = &_sdata;         // RAM 中的目標位址
while (dst < &_edata) {
    *dst++ = *src++;
}
```

**`.RamFunc` 段**：某些場景需要將函式拷貝到 RAM 中執行（如 Flash 擦除/程式化時無法從同一 Flash bank 取指）。用 `__attribute__((section(".RamFunc")))` 標記。

### 7.7 TLS (Thread-Local Storage) 段

```ld
.tdata : ALIGN(4)
{
    *(.tdata .tdata.* .gnu.linkonce.td.*)
    . = ALIGN(4);
    _edata = .;
    PROVIDE(__data_end = .);
    PROVIDE(__tdata_end = .);
} >RAM AT> FLASH

PROVIDE( __tdata_start = ADDR(.tdata) );
PROVIDE( __tdata_size = __tdata_end - __tdata_start );
PROVIDE( __tdata_source = LOADADDR(.tdata) );
PROVIDE( __tdata_source_end = LOADADDR(.tdata) + SIZEOF(.tdata) );

.tbss (NOLOAD) : ALIGN(4)
{
    _sbss = .;
    __bss_start__ = _sbss;
    *(.tbss .tbss.*)
    . = ALIGN(4);
    PROVIDE( __tbss_end = . );
} >RAM
```

TLS 是 C11 `_Thread_local` / GCC `__thread` 的底層支援。在裸機嵌入式中使用較少，但在使用 RTOS 時每個執行緒可能需要獨立的變數副本。

**匯出符號**：

| 符號 | 含義 |
|------|------|
| `__tdata_start` / `__tdata_end` | `.tdata` 的 VMA 範圍 |
| `__tdata_source` / `__tdata_source_end` | `.tdata` 的 LMA 範圍 |
| `__tls_size` | TLS 總大小 |
| `__tls_align` | TLS 對齊要求 |

### 7.8 BSS 段 (.bss)

```ld
.bss (NOLOAD) : ALIGN(4)
{
    *(.bss)
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;
    __bss_end__ = _ebss;
    PROVIDE( __bss_end = .);
} >RAM
```

**`(NOLOAD)`**：該段不產生任何輸出內容，僅分配位址空間。BSS 段存放未初始化的全域/靜態變數，執行時由啟動程式碼清零：

```c
// 啟動程式碼清零 BSS
uint32_t *dst = &_sbss;
while (dst < &_ebss) {
    *dst++ = 0;
}
```

**`(COMMON)`**：C 語言的未初始化全域變數（tentative definitions）在鏈結前放在 COMMON 段，鏈結時合併到 `.bss`。

### 7.9 DISCARD 段

```ld
/DISCARD/ :
{
    libc.a:* ( * )
    libm.a:* ( * )
    libgcc.a:* ( * )
}
```

丟棄標準函式庫的某些內容。在深度嵌入式（newlib-nano、nosys）中，這可以避免鏈結不需要的函式庫程式碼，減小韌體體積。

---

## 八、ALIGN 與位址對齊

```ld
. = ALIGN(4);
```

- `.` 是**位置計數器**（location counter），表示當前輸出偏移
- `ALIGN(4)` 將位置計數器向上對齊到 4 位元組邊界
- ARM Cortex-M 要求所有存取對齊到自然邊界（32 位元對 4 位元組，64 位元對 8 位元組）

**為什麼 .isr_vector 需要 ALIGN(4)？**

中斷向量表的每個條目是 4 位元組（32 位元位址），必須 4 位元組對齊，否則 CPU 取指異常。

---

## 九、PROVIDE 與符號可見性

```ld
PROVIDE( __bss_end = . );
```

`PROVIDE` 定義一個**弱符號**：僅當程式中沒有其他地方定義同名符號時才生效。如果 C 程式碼中定義了 `__bss_end`，鏈結器使用 C 程式碼的定義，忽略 `PROVIDE`。

**與直接賦值的區別**：

| 語法 | 行為 |
|------|------|
| `_sbss = .;` | 強制定義，如果程式中也有定義則報多重定義錯誤 |
| `PROVIDE(_sbss = .);` | 僅當未定義時才提供，允許程式覆寫 |

---

## 十、KEEP 與 --gc-sections

`--gc-sections`（`-ffunction-sections -fdata-sections` + `-Wl,--gc-sections`）是嵌入式韌體瘦身的關鍵最佳化：編譯器為每個函式和資料物件產生獨立段，鏈結器刪除未被引用的段。

**問題**：中斷向量表、建構/解構函式陣列等沒有顯式引用者，會被誤刪。

**解決方案**：`KEEP()` 告訴鏈結器「保留這個段，即使它看起來未被引用」。

```ld
KEEP(*(.isr_vector))    /* 向量表：CPU 硬體跳轉，無軟體引用 */
KEEP (*(.init))         /* C++ 初始化 */
KEEP (*(.fini))         /* C++ 清理 */
KEEP (*(SORT(.init_array.*)))  /* 建構函式指標陣列 */
```

---

## 十一、啟動程式碼與鏈結腳本的協作

鏈結腳本定義了符號，啟動程式碼（`startup_stm32f407xx.s`）使用這些符號完成初始化。兩者必須嚴格配合：

```
┌──────────────────────┐        ┌──────────────────────┐
│     鏈結腳本 (.ld)    │        │  啟動程式碼 (.s)     │
│                      │        │                      │
│  _estack             │───────→│  初始 SP 值          │
│  _sidata             │───────→│  .data Flash 來源位址│
│  _sdata, _edata     │───────→│  .data RAM 範圍      │
│  _sbss, _ebss       │───────→│  .bss RAM 範圍       │
│  _siccmram           │───────→│  .ccmram Flash 來源  │
│  _sccmram, _eccmram  │───────→│  .ccmram CCMRAM 範圍│
└──────────────────────┘        └──────────────────────┘
```

### 11.1 完整啟動流程

```asm
Reset_Handler:
    /* 1. 設定堆疊指標（鏈結腳本 _estack） */
    ldr  sp, =_estack

    /* 2. 拷貝 .data 從 Flash 到 RAM */
    ldr  r0, =_sdata      /* dst start */
    ldr  r1, =_edata      /* dst end */
    ldr  r2, =_sidata     /* src start (LMA) */
CopyData:
    cmp  r0, r1
    ittt lt
    ldrlt r3, [r2], #4
    strlt r3, [r0], #4
    blt  CopyData

    /* 3. 清零 .bss */
    ldr  r0, =_sbss
    ldr  r1, =_ebss
    movs r2, #0
ClearBSS:
    cmp  r0, r1
    itt  lt
    strlt r2, [r0], #4
    blt  ClearBSS

    /* 4. 呼叫 SystemInit() */
    bl   SystemInit

    /* 5. 呼叫 __libc_init_array()（C++ 建構函式） */
    bl   __libc_init_array

    /* 6. 呼叫 main() */
    bl   main

    /* 7. main 不應返回，若返回則死循環 */
LoopForever:
    b    LoopForever
```

### 11.2 C++ 建構函式呼叫

`__libc_init_array()` 遍歷 `.init_array` 段中的函式指標並逐一呼叫。這個段由鏈結腳本中的 `KEEP (*(SORT(.init_array.*)))` 保證存在且按正確順序排列（`SORT` 確保按優先級排序）。

---

## 十二、自訂段的實踐

### 12.1 將變數放入 CCMRAM

```c
// 鏈結腳本中已有 .ccmram 段定義
__attribute__((section(".ccmram"))) int ai_tensor_buf[1024];  // AI 推理張量緩衝區
```

AI 推理中的張量緩衝區放入 CCMRAM 可獲得零等待存取，提升推理速度。

### 12.2 將函式放入 RAM 執行

```c
// 鏈結腳本中已有 .RamFunc 段
__attribute__((section(".RamFunc"))) void flash_erase_and_write(void);
```

STM32F407 的 Flash 在擦除/程式化期間，同一 bank 無法取指。將 Flash 操作函式放入 RAM 可避免此問題。

### 12.3 NoInit 區域

```c
// 不受啟動程式碼初始化的變數（軟復位後保持值）
__attribute__((section(".bss.NoInit"))) volatile uint32_t update_flag;
```

Bootloader 與 App 透過軟復位通訊時，`NoInit` 變數不被啟動程式碼清零，可以跨復位傳遞狀態。

---

## 十三、常見問題與除錯

### 13.1 region overflowed

```
region RAM overflowed by 1234 bytes
```

原因：`.data` + `.bss` + heap + stack 超出 RAM 容量。

排查：
1. 檢視 `.map` 檔案中各段大小
2. 檢查是否有大陣列誤放 RAM（應考慮用 Flash 或外部 SRAM）
3. 增大 `_Min_Stack_Size` 或最佳化程式碼

### 13.2 多重定義錯誤

```
multiple definition of `_sbss'
```

鏈結腳本中用 `_sbss = .;` 是強定義，如果 C 程式碼中也宣告了同名符號則衝突。改用 `PROVIDE(_sbss = .)` 或修改 C 符號名。

### 13.3 .data 拷貝不正確

症狀：全域變數初始值不是預期的值。

排查：
1. 確認 `_sidata` 與 `_sdata` 的值：`objdump -t firmware.elf | grep -E '_s(data|idata)'`
2. 確認啟動程式碼拷貝邏輯是否覆蓋了 `.data` 到 `.tdata` 的完整範圍
3. 檢查 CCMRAM 拷貝是否被遺漏

### 13.4 檢視段佈局

```bash
# 檢視各段的 VMA 和 LMA
arm-none-eabi-objdump -h firmware.elf

# 檢視完整記憶體映射
arm-none-eabi-objdump -t firmware.elf | sort

# 檢視段大小彙總
arm-none-eabi-size firmware.elf
   text    data     bss     dec     hex filename
  45678    1234    8900   55812    d9e4 firmware.elf
```

`size` 輸出中：
- `text`：.text + .rodata（Flash 佔用）
- `data`：.data（Flash 存初始值 + RAM 執行時佔用）
- `bss`：.bss（僅 RAM 佔用）
- Flash 總佔用 = text + data
- RAM 總佔用 = data + bss

---

## 十四、總結

| 概念 | 關鍵理解 |
|------|----------|
| MEMORY | 宣告晶片的實體記憶體區域和屬性 |
| SECTIONS | 規定各段的排列順序和存放位置 |
| VMA / LMA | 執行時位址 vs 載入時位址，.data 必須拷貝 |
| >RAM AT> FLASH | VMA 在 RAM，LMA 在 Flash |
| ALIGN | 確保硬體要求的位址對齊 |
| KEEP | 防止 --gc-sections 誤刪關鍵段 |
| PROVIDE | 弱符號定義，允許程式覆寫 |
| _s*/_e* 符號 | 匯出給啟動程式碼使用的邊界位址 |
| .bss (NOLOAD) | 不產生輸出，僅分配位址，啟動時清零 |
| /DISCARD/ | 丟棄不需要的標準函式庫內容 |

鏈結腳本不是「設定好就不用管」的黑箱，而是連接硬體記憶體拓撲、編譯器輸出和啟動程式碼的橋樑。理解它，才能在記憶體最佳化、多分割區 Bootloader、自訂段等場景中遊刃有餘。

---

## 參考資料

- [GNU LD Manual - Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html)
- [STM32F4xx Reference Manual - Memory Map](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [GCC Attribute Documentation](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html)
- [ARM Cortex-M4 Technical Reference Manual](https://developer.arm.com/documentation/ddi0343/latest/)
