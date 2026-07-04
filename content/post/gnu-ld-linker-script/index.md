---
title: "GNU LD 链接脚本详解：从 STM32F407 实例剖析嵌入式内存布局与段重定位"
date: 2026-07-02
description: "以 STM32F407VGT6 的 GCC 链接脚本为实例，深入剖析 GNU LD 的 MEMORY 命令、SECTIONS 段分配、VMA/LMA 地址模型、启动代码与链接脚本的协作、CCMRAM 使用、TLS 支持等核心技术"
image: gnu-ld-linker-script.png
categories:
  - "嵌入式"
tags:
  - "STM32"
  - "链接脚本"
  - "GNU-LD"
  - "GCC"
  - "嵌入式"
math: true
---

## 前言

链接脚本（Linker Script, `.ld`）是嵌入式开发中最常被忽视、却又最关键的文件之一。它告诉链接器：程序的代码和数据应该放在内存的什么位置、各个段的排列顺序如何、启动时需要从 Flash 拷贝到 RAM 的数据有哪些。

本文以一个真实项目的 STM32F407VGT6 链接脚本为例，逐行剖析 GNU LD 链接脚本的每一个语法要素，并深入讲解 VMA/LMA 地址模型、启动代码与链接脚本的协作机制。

---

## 一、实例项目背景

本实例来自一个运行在 **STM32F407VGT6** 上的 TensorFlow Lite Micro 端侧 AI 推理项目，使用 CMake + Ninja + GCC Arm Embedded 交叉编译。

芯片资源：

| 资源   | 大小    | 起始地址     |
| ------ | ------- | ------------ |
| Flash  | 1024 KB | `0x08000000` |
| SRAM   | 112 KB  | `0x20000000` |
| CCMRAM | 64 KB   | `0x10000000` |

---

## 二、链接脚本整体结构

一个典型的 GNU LD 链接脚本由三大部分组成：

```
ENTRY(...)         ← 入口点定义

MEMORY { ... }     ← 内存区域声明

SECTIONS { ... }   ← 段分配规则
```

下面逐一剖析。

---

## 三、ENTRY：程序入口点

```ld
ENTRY(Reset_Handler)
```

`ENTRY` 命令告诉链接器程序的入口符号是 `Reset_Handler`。这个值会被写入 ELF 文件的入口点字段，调试器和仿真器据此知道从哪里开始执行。

**在嵌入式裸机中**，真正决定执行起始位置的是中断向量表（`.isr_vector` 段）中的栈指针和 ResetHandler，它们被放在 Flash 的最前面（`0x08000000`），CPU 上电后从这里取指。`ENTRY` 主要影响 ELF 元数据和调试体验。

---

## 四、MEMORY：内存区域声明

```ld
MEMORY
{
RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 112K
CCMRAM (xrw)   : ORIGIN = 0x10000000, LENGTH = 64K
FLASH (rx)      : ORIGIN = 0x08000000, LENGTH = 1024K
}
```

### 4.1 语法

```
<名称> (<属性>) : ORIGIN = <起始地址>, LENGTH = <大小>
```

### 4.2 属性字母

| 属性 | 含义    | 可放入的段               |
| ---- | ------- | ------------------------ |
| `r`  | Read    | 只读数据（.rodata）      |
| `w`  | Write   | 可写数据（.data, .bss）  |
| `x`  | eXecute | 可执行代码（.text）      |
| `a`  | Alloc   | 可分配（默认所有段都有） |
| `i`  | Init    | 已初始化段               |

本实例中：

- **FLASH (rx)**：只读 + 可执行，存放代码和常量
- **RAM (xrw)**：可读 + 可写 + 可执行，存放已初始化数据和 BSS
- **CCMRAM (xrw)**：Core Coupled Memory，与 CPU 内核直接耦合，零等待访问

### 4.3 CCMRAM 的特殊性

STM32F407 的 CCMRAM 是一块**仅 CPU 可访问**的 64KB SRAM，DMA 控制器无法访问。属性设为 `xrw` 允许将代码放入 CCMRAM 执行（性能敏感的算法可受益于零等待），但需注意 DMA 不可触及。

### 4.4 地址空间布局图

| 地址范围                    | 区域                | 内容                                                                  |
| --------------------------- | ------------------- | --------------------------------------------------------------------- |
| `0x00000000`                | Code Region         | 通过 SYSBOOT 映射到 Flash                                             |
| `0x08000000` - `0x080FFFFF` | **FLASH (1024 KB)** | `.isr_vector` → `.text` → `.rodata` → `.data (LMA)` → `.ccmram (LMA)` |
| `0x10000000` - `0x1000FFFF` | **CCMRAM (64 KB)**  | `.ccmram (VMA)` — 启动时从 Flash 拷贝                                 |
| `0x20000000` - `0x2001BFFF` | **RAM (112 KB)**    | `.data (VMA)` → `.bss` → Heap → Stack（栈从高地址向低地址增长）       |

**LMA vs VMA 拷贝关系**：

```
Flash (LMA)                RAM/CCMRAM (VMA)
├── .data初始值    ──────→ .data运行时位置（拷贝）
├── .ccmram初始值  ──────→ .ccmram运行时位置（拷贝）
│
.bss                         ──────→ 启动时清零（不拷贝）
```

---

## 五、VMA 与 LMA：理解双地址模型

这是链接脚本最核心的概念。

- **VMA (Virtual Memory Address)**：程序运行时的地址。CPU 执行时使用的地址。
- **LMA (Load Memory Address)**：程序被加载（烧录）到的地址。Flash 中实际存储的位置。

### 5.1 为什么需要两个地址？

对于 `.text` 和 `.rodata`，VMA = LMA，它们在 Flash 中运行，不需要搬运。

但对于 `.data`（已初始化全局变量），情况不同：

```c
int global_counter = 42;  // 已初始化全局变量
```

这个变量的**初始值 42** 必须存储在 Flash 中（掉电不丢失），但**运行时**变量必须在 RAM 中（可读写）。因此：

- **LMA** = Flash 中的地址（存放初始值 42 的位置）
- **VMA** = RAM 中的地址（程序运行时读写 `global_counter` 的位置）

启动代码的责任就是在 `main()` 之前，将 `.data` 段从 LMA 拷贝到 VMA。

### 5.2 AT> 语法

```ld
.data :
{
    ...
} >RAM AT> FLASH
```

- `>RAM`：指定 VMA 区域为 RAM
- `AT> FLASH`：指定 LMA 区域为 Flash

如果省略 `AT>`，则 LMA = VMA，段既存放又运行在同一区域。

### 5.3 形式化表示

对于段 $S$：

$$\text{VMA}(S) = \text{Addr}(S \text{ in target region})$$
$$\text{LMA}(S) = \text{Addr}(S \text{ in load region})$$

运行时不变量：

$$\forall x \in \text{.data}, \text{Mem}[\text{VMA}(x)] = \text{Mem}[\text{LMA}(x)] \text{ （启动拷贝完成后）}$$

---

## 六、栈与堆：链接时检查

```ld
_estack = ORIGIN(RAM) + LENGTH(RAM);    /* RAM 末尾 */
_sstack = _estack - _Min_Stack_Size;
_Min_Heap_Size = 0x800;      /* 2 KB */
_Min_Stack_Size = 0x800;     /* 2 KB */
```

### 6.1 栈的生长方向

ARM Cortex-M 的栈是**满递减**（Full Descending）：

- 栈指针初始值 = RAM 最高地址（`_estack`）
- 每次 PUSH，SP 先减 4 再写入
- 每次 POP，先读出 SP 再加 4

```
RAM 高地址 ← _estack (SP 初始值)
              │ Stack │  ↓ 向低地址增长
              │       │
              │ Heap  │  ↑ 向高地址增长
RAM 低地址 ← _sdata
```

### 6.2 最小堆栈检查

`._user_heap_stack` 段在链接时确保 RAM 中有足够空间：

```ld
._user_heap_stack (NOLOAD) :
{
    . = ALIGN(8);
    PROVIDE ( end = . );
    PROVIDE ( _end = . );
    . = . + _Min_Heap_Size;    /* 预留 2KB 堆 */
    . = . + _Min_Stack_Size;   /* 预留 2KB 栈 */
    . = ALIGN(8);
} >RAM
```

如果 `.bss` + heap + stack 超出 RAM 容量，链接器会报错：

```
region RAM overflowed by 1234 bytes
```

> [!TIP]
> `_Min_Stack_Size = 0x800` (2KB) 通常是**最小值**，实际栈用量取决于调用深度和局部变量大小。对于使用 RTOS 或中断嵌套的场景，可能需要 4KB 甚至更大。可用 `__stack_limit` 符号或 MPU 进行运行时栈溢出检测。

---

## 七、SECTIONS：段分配详解

### 7.1 中断向量表 (.isr_vector)

```ld
.isr_vector :
{
    . = ALIGN(4);
    KEEP(*(.isr_vector))    /* 强制保留，不被 --gc-sections 优化掉 */
    . = ALIGN(4);
} >FLASH
```

**为什么必须放在 Flash 最前面？**

STM32 上电后，CPU 从 `0x00000000` 开始取指。地址 `0x00000000` 通过 SYSBOOT 配置被映射（alias）到 Flash 起始地址 `0x08000000`。CPU 首先读取：

| 偏移 | 内容             | 含义                     |
| ---- | ---------------- | ------------------------ |
| 0x00 | 初始 SP          | 主栈指针初始值           |
| 0x04 | ResetHandler     | 复位向量，跳转到启动代码 |
| 0x08 | NMIHandler       | 不可屏蔽中断             |
| 0x0C | HardFaultHandler | 硬件错误                 |
| ...  | ...              | 其他中断/异常            |

**`KEEP` 的作用**：`--gc-sections` 优化会删除未被引用的段，但 `.isr_vector` 中的向量表没有显式引用者（CPU 硬件自动跳转）。`KEEP` 防止其被优化掉。

### 7.2 代码段 (.text)

```ld
.text :
{
    . = ALIGN(4);
    *(.text)           /* 所有 .text 段 */
    *(.text*)          /* .text.* 命名段 */
    *(.glue_7)         /* ARM → Thumb 桩代码 */
    *(.glue_7t)        /* Thumb → ARM 桩代码 */
    *(.eh_frame)       /* 异常处理帧（C++） */

    KEEP (*(.init))    /* 构造函数初始化 */
    KEEP (*(.fini))    /* 析构函数清理 */

    . = ALIGN(4);
    _etext = .;        /* 代码段结束符号 */
} >FLASH
```

**`.glue_7` / `.glue_7t`**：ARM 和 Thumb 指令集切换时的跳转桩代码。Cortex-M4 只支持 Thumb-2，但这些段在通用 ARM 工具链中仍会生成。

**`_etext`**：导出符号，标记代码段末尾。启动代码用它来确定 `.data` 段在 Flash 中的起始位置（`_sidata = LOADADDR(.data)` 更精确）。

### 7.3 只读数据段 (.rodata)

```ld
.rodata :
{
    . = ALIGN(4);
    *(.rodata)         /* const 全局变量、字符串字面量 */
    *(.rodata*)        /* .rodata.* 命名段 */
    . = ALIGN(4);
} >FLASH
```

`.rodata` 存放 `const` 全局变量、字符串字面量等。在嵌入式系统中，这些数据留在 Flash 中，不占用 RAM。

**一个常见陷阱**：

```c
const char *msg = "hello";   // "hello" 在 .rodata，msg 指针在 .data
const char msg[] = "hello";  // 整个都在 .rodata，更优
```

### 7.4 C++ 相关段

```ld
.ARM.extab (READONLY) : { ... } >FLASH
.ARM (READONLY) : { ... } >FLASH
.preinit_array (READONLY) : { ... } >FLASH
.init_array (READONLY) : { ... } >FLASH
.fini_array (READONLY) : { ... } >FLASH
```

| 段                    | 用途                     |
| --------------------- | ------------------------ |
| `.ARM.extab` / `.ARM` | C++ 异常处理的展开表     |
| `.preinit_array`      | 最早期初始化函数指针数组 |
| `.init_array`         | C++ 全局构造函数指针数组 |
| `.fini_array`         | C++ 全局析构函数指针数组 |

`READONLY` 关键字是 GCC 11+ 的特性，告诉链接器这些段不可写。使用 GCC 10 或更早版本时需要移除。

**纯 C 项目**：如果没有 C++ 代码，这些段通常为空。但 `KEEP` 确保即使 `--gc-sections` 也不会误删。

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

- `>CCMRAM`：VMA 在 CCMRAM 区域
- `AT> FLASH`：LMA 在 Flash（初始值存储在 Flash 中）
- `_siccmram`：LMA 起始地址，启动代码据此将数据从 Flash 拷贝到 CCMRAM

**使用方式**：在 C 代码中用 `__attribute__` 将变量放入 CCMRAM：

```c
__attribute__((section(".ccmram"))) int fast_buffer[256];
```

> [!WARNING]
> 启动代码必须显式处理 CCMRAM 的拷贝。STM32CubeMX 生成的默认 `startup_*.s` 可能不包含 CCMRAM 拷贝逻辑，需要手动添加。

### 7.6 已初始化数据段 (.data)

```ld
_sidata = LOADADDR(.data);   /* .data 在 Flash 中的起始地址 */

.data :
{
    . = ALIGN(4);
    _sdata = .;               /* VMA 起始 */
    *(.data)
    *(.data*)
    *(.RamFunc)               /* 在 RAM 中执行的函数 */
    *(.RamFunc*)
    . = ALIGN(4);
} >RAM AT> FLASH
```

**导出符号**：

| 符号      | 含义                                        |
| --------- | ------------------------------------------- |
| `_sidata` | `.data` 的 LMA 起始（Flash 中）             |
| `_sdata`  | `.data` 的 VMA 起始（RAM 中）               |
| `_edata`  | `.data` 的 VMA 结束（在 .tdata 段末尾定义） |

**启动拷贝伪代码**：

```c
// startup_stm32f407xx.s 中的逻辑
uint32_t *src = &_sidata;        // Flash 中的源地址
uint32_t *dst = &_sdata;         // RAM 中的目标地址
while (dst < &_edata) {
    *dst++ = *src++;
}
```

**`.RamFunc` 段**：某些场景需要将函数拷贝到 RAM 中执行（如 Flash 擦除/编程时无法从同一 Flash bank 取指）。用 `__attribute__((section(".RamFunc")))` 标记。

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

TLS 是 C11 `_Thread_local` / GCC `__thread` 的底层支持。在裸机嵌入式中使用较少，但在使用 RTOS 时每个线程可能需要独立的变量副本。

**导出符号**：

| 符号                                    | 含义                 |
| --------------------------------------- | -------------------- |
| `__tdata_start` / `__tdata_end`         | `.tdata` 的 VMA 范围 |
| `__tdata_source` / `__tdata_source_end` | `.tdata` 的 LMA 范围 |
| `__tls_size`                            | TLS 总大小           |
| `__tls_align`                           | TLS 对齐要求         |

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

**`(NOLOAD)`**：该段不生成任何输出内容，仅分配地址空间。BSS 段存放未初始化的全局/静态变量，运行时由启动代码清零：

```c
// 启动代码清零 BSS
uint32_t *dst = &_sbss;
while (dst < &_ebss) {
    *dst++ = 0;
}
```

**`(COMMON)`**：C 语言的未初始化全局变量（ tentative definitions）在链接前放在 COMMON 段，链接时合并到 `.bss`。

### 7.9 DISCARD 段

```ld
/DISCARD/ :
{
    libc.a:* ( * )
    libm.a:* ( * )
    libgcc.a:* ( * )
}
```

丢弃标准库的某些内容。在深度嵌入式（newlib-nano、nosys）中，这可以避免链接不需要的库代码，减小固件体积。

---

## 八、ALIGN 与地址对齐

```ld
. = ALIGN(4);
```

- `.` 是**位置计数器**（location counter），表示当前输出偏移
- `ALIGN(4)` 将位置计数器向上对齐到 4 字节边界
- ARM Cortex-M 要求所有访问对齐到自然边界（32 位对 4 字节，64 位对 8 字节）

**为什么 .isr_vector 需要 ALIGN(4)？**

中断向量表的每个条目是 4 字节（32 位地址），必须 4 字节对齐，否则 CPU 取指异常。

---

## 九、PROVIDE 与符号可见性

```ld
PROVIDE( __bss_end = . );
```

`PROVIDE` 定义一个**弱符号**：仅当程序中没有其他地方定义同名符号时才生效。如果 C 代码中定义了 `__bss_end`，链接器使用 C 代码的定义，忽略 `PROVIDE`。

**与直接赋值的区别**：

| 语法                  | 行为                                         |
| --------------------- | -------------------------------------------- |
| `_sbss = .;`          | 强制定义，如果程序中也有定义则报多重定义错误 |
| `PROVIDE(_sbss = .);` | 仅当未定义时才提供，允许程序覆盖             |

---

## 十、KEEP 与 --gc-sections

`--gc-sections`（`-ffunction-sections -fdata-sections` + `-Wl,--gc-sections`）是嵌入式固件瘦身的关键优化：编译器为每个函数和数据对象生成独立段，链接器删除未被引用的段。

**问题**：中断向量表、构造/析构函数数组等没有显式引用者，会被误删。

**解决方案**：`KEEP()` 告诉链接器"保留这个段，即使它看起来未被引用"。

```ld
KEEP(*(.isr_vector))    /* 向量表：CPU 硬件跳转，无软件引用 */
KEEP (*(.init))         /* C++ 初始化 */
KEEP (*(.fini))         /* C++ 清理 */
KEEP (*(SORT(.init_array.*)))  /* 构造函数指针数组 */
```

---

## 十一、启动代码与链接脚本的协作

链接脚本定义了符号，启动代码（`startup_stm32f407xx.s`）使用这些符号完成初始化。两者必须严格配合：

```
┌──────────────────────┐        ┌──────────────────────┐
│     链接脚本 (.ld)    │        │  启动代码 (.s)       │
│                      │        │                      │
│  _estack             │───────→│  初始 SP 值          │
│  _sidata             │───────→│  .data Flash 源地址  │
│  _sdata, _edata     │───────→│  .data RAM 范围      │
│  _sbss, _ebss       │───────→│  .bss RAM 范围       │
│  _siccmram           │───────→│  .ccmram Flash 源   │
│  _sccmram, _eccmram  │───────→│  .ccmram CCMRAM 范围│
└──────────────────────┘        └──────────────────────┘
```

### 11.1 完整启动流程

```asm
Reset_Handler:
    /* 1. 设置栈指针（链接脚本 _estack） */
    ldr  sp, =_estack

    /* 2. 拷贝 .data 从 Flash 到 RAM */
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

    /* 4. 调用 SystemInit() */
    bl   SystemInit

    /* 5. 调用 __libc_init_array()（C++ 构造函数） */
    bl   __libc_init_array

    /* 6. 调用 main() */
    bl   main

    /* 7. main 不应返回，若返回则死循环 */
LoopForever:
    b    LoopForever
```

### 11.2 C++ 构造函数调用

`__libc_init_array()` 遍历 `.init_array` 段中的函数指针并逐一调用。这个段由链接脚本中的 `KEEP (*(SORT(.init_array.*)))` 保证存在且按正确顺序排列（`SORT` 确保按优先级排序）。

---

## 十二、自定义段的实践

### 12.1 将变量放入 CCMRAM

```c
// 链接脚本中已有 .ccmram 段定义
__attribute__((section(".ccmram"))) int ai_tensor_buf[1024];  // AI 推理张量缓冲区
```

AI 推理中的张量缓冲区放入 CCMRAM 可获得零等待访问，提升推理速度。

### 12.2 将函数放入 RAM 执行

```c
// 链接脚本中已有 .RamFunc 段
__attribute__((section(".RamFunc"))) void flash_erase_and_write(void);
```

STM32F407 的 Flash 在擦除/编程期间，同一 bank 无法取指。将 Flash 操作函数放入 RAM 可避免此问题。

### 12.3 NoInit 区域

```c
// 不受启动代码初始化的变量（软复位后保持值）
__attribute__((section(".bss.NoInit"))) volatile uint32_t update_flag;
```

Bootloader 与 App 通过软复位通信时，`NoInit` 变量不被启动代码清零，可以跨复位传递状态。

---

## 十三、常见问题与调试

### 13.1 region overflowed

```
region RAM overflowed by 1234 bytes
```

原因：`.data` + `.bss` + heap + stack 超出 RAM 容量。

排查：

1. 查看 `.map` 文件中各段大小
2. 检查是否有大数组误放 RAM（应考虑用 Flash 或外部 SRAM）
3. 增大 `_Min_Stack_Size` 或优化代码

### 13.2 多重定义错误

```
multiple definition of `_sbss'
```

链接脚本中用 `_sbss = .;` 是强定义，如果 C 代码中也声明了同名符号则冲突。改用 `PROVIDE(_sbss = .)` 或修改 C 符号名。

### 13.3 .data 拷贝不正确

症状：全局变量初始值不是预期的值。

排查：

1. 确认 `_sidata` 与 `_sdata` 的值：`objdump -t firmware.elf | grep -E '_s(data|idata)'`
2. 确认启动代码拷贝逻辑是否覆盖了 `.data` 到 `.tdata` 的完整范围
3. 检查 CCMRAM 拷贝是否被遗漏

### 13.4 查看段布局

```bash
# 查看各段的 VMA 和 LMA
arm-none-eabi-objdump -h firmware.elf

# 查看完整内存映射
arm-none-eabi-objdump -t firmware.elf | sort

# 查看段大小汇总
arm-none-eabi-size firmware.elf
   text    data     bss     dec     hex filename
  45678    1234    8900   55812    d9e4 firmware.elf
```

`size` 输出中：

- `text`：.text + .rodata（Flash 占用）
- `data`：.data（Flash 存初始值 + RAM 运行时占用）
- `bss`：.bss（仅 RAM 占用）
- Flash 总占用 = text + data
- RAM 总占用 = data + bss

---

## 十四、总结

| 概念           | 关键理解                                 |
| -------------- | ---------------------------------------- |
| MEMORY         | 声明芯片的物理内存区域和属性             |
| SECTIONS       | 规定各段的排列顺序和存放位置             |
| VMA / LMA      | 运行时地址 vs 加载时地址，.data 必须拷贝 |
| >RAM AT> FLASH | VMA 在 RAM，LMA 在 Flash                 |
| ALIGN          | 确保硬件要求的地址对齐                   |
| KEEP           | 防止 --gc-sections 误删关键段            |
| PROVIDE        | 弱符号定义，允许程序覆盖                 |
| \_s*/\_e* 符号 | 导出给启动代码使用的边界地址             |
| .bss (NOLOAD)  | 不生成输出，仅分配地址，启动时清零       |
| /DISCARD/      | 丢弃不需要的标准库内容                   |

链接脚本不是"配置好就不用管"的黑箱，而是连接硬件内存拓扑、编译器输出和启动代码的桥梁。理解它，才能在内存优化、多分区 Bootloader、自定义段等场景中游刃有余。

---

## 参考资料

- [GNU LD Manual - Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html)
- [STM32F4xx Reference Manual - Memory Map](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [GCC Attribute Documentation](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html)
- [ARM Cortex-M4 Technical Reference Manual](https://developer.arm.com/documentation/ddi0343/latest/)
