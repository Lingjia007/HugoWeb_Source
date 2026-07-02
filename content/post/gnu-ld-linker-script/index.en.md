---
title: "GNU LD Linker Script in Depth: Dissecting Embedded Memory Layout and Section Relocation via an STM32F407 Example"
date: 2026-07-02
description: "Using the GCC linker script for STM32F407VGT6 as a real-world example, this post provides an in-depth analysis of GNU LD's MEMORY command, SECTIONS allocation, VMA/LMA address model, the cooperation between startup code and linker scripts, CCMRAM usage, TLS support, and other core technologies"
categories:
  - "Embedded"
tags:
  - "STM32"
  - "Linker-Script"
  - "GNU-LD"
  - "GCC"
  - "Embedded"
math: true
---

## Introduction

A linker script (`.ld`) is one of the most overlooked yet critical files in embedded development. It tells the linker where the program's code and data should be placed in memory, the arrangement order of each section, and which data needs to be copied from Flash to RAM at startup.

This article uses a real project's STM32F407VGT6 linker script as an example, walking through every syntactic element of a GNU LD linker script line by line, and providing an in-depth explanation of the VMA/LMA address model and the cooperation mechanism between startup code and the linker script.

---

## 1. Example Project Background

This example comes from a TensorFlow Lite Micro edge AI inference project running on **STM32F407VGT6**, built with CMake + Ninja + GCC Arm Embedded cross-compiler.

Chip resources:

| Resource | Size | Start Address |
|----------|------|---------------|
| Flash | 1024 KB | `0x08000000` |
| SRAM | 112 KB | `0x20000000` |
| CCMRAM | 64 KB | `0x10000000` |

---

## 2. Overall Structure of a Linker Script

A typical GNU LD linker script consists of three major parts:

```
ENTRY(...)         <- Entry point definition

MEMORY { ... }     <- Memory region declarations

SECTIONS { ... }   <- Section allocation rules
```

Let's examine each one.

---

## 3. ENTRY: Program Entry Point

```ld
ENTRY(Reset_Handler)
```

The `ENTRY` command tells the linker that the program's entry symbol is `Reset_Handler`. This value is written into the ELF file's entry point field, allowing debuggers and simulators to know where execution begins.

**In bare-metal embedded systems**, what truly determines the execution start point is the stack pointer and Reset Handler in the interrupt vector table (`.isr_vector` section), which are placed at the very beginning of Flash (`0x08000000`). After power-on, the CPU begins fetching instructions from there. `ENTRY` primarily affects ELF metadata and the debugging experience.

---

## 4. MEMORY: Memory Region Declarations

```ld
MEMORY
{
RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 112K
CCMRAM (xrw)   : ORIGIN = 0x10000000, LENGTH = 64K
FLASH (rx)      : ORIGIN = 0x08000000, LENGTH = 1024K
}
```

### 4.1 Syntax

```
<name> (<attributes>) : ORIGIN = <start address>, LENGTH = <size>
```

### 4.2 Attribute Letters

| Attribute | Meaning | Sections that can be placed |
|-----------|---------|----------------------------|
| `r` | Read | Read-only data (.rodata) |
| `w` | Write | Writable data (.data, .bss) |
| `x` | eXecute | Executable code (.text) |
| `a` | Alloc | Allocatable (default for all sections) |
| `i` | Init | Initialized sections |

In this example:
- **FLASH (rx)**: Read-only + executable, stores code and constants
- **RAM (xrw)**: Readable + writable + executable, stores initialized data and BSS
- **CCMRAM (xrw)**: Core Coupled Memory, directly coupled to the CPU core, zero-wait-state access

### 4.3 Special Characteristics of CCMRAM

The CCMRAM on STM32F407 is a 64KB SRAM **accessible only by the CPU**; the DMA controller cannot access it. Setting its attribute to `xrw` allows code to be placed in CCMRAM for execution (performance-sensitive algorithms benefit from zero-wait states), but note that DMA cannot reach this memory.

### 4.4 Address Space Layout Diagram

| Address Range | Region | Contents |
|---------------|--------|----------|
| `0x00000000` | Code Region | Aliased to Flash via SYSBOOT |
| `0x08000000` - `0x080FFFFF` | **FLASH (1024 KB)** | `.isr_vector` → `.text` → `.rodata` → `.data (LMA)` → `.ccmram (LMA)` |
| `0x10000000` - `0x1000FFFF` | **CCMRAM (64 KB)** | `.ccmram (VMA)` — Copied from Flash at startup |
| `0x20000000` - `0x2001BFFF` | **RAM (112 KB)** | `.data (VMA)` → `.bss` → Heap → Stack (stack grows from high to low addresses) |

**LMA vs VMA Copy Relationships**:

```
Flash (LMA)                 RAM/CCMRAM (VMA)
|-- .data init values  ----> .data runtime location (copy)
|-- .ccmram init values ----> .ccmram runtime location (copy)
|
.bss                        ----> Zeroed at startup (no copy)
```

---

## 5. VMA and LMA: Understanding the Dual Address Model

This is the most core concept of linker scripts.

- **VMA (Virtual Memory Address)**: The address at runtime. The address used by the CPU during execution.
- **LMA (Load Memory Address)**: The address where the program is loaded (flashed). The actual storage location in Flash.

### 5.1 Why Do We Need Two Addresses?

For `.text` and `.rodata`, VMA = LMA; they execute in Flash and do not need to be moved.

But for `.data` (initialized global variables), the situation is different:

```c
int global_counter = 42;  // Initialized global variable
```

The **initial value 42** of this variable must be stored in Flash (non-volatile), but at **runtime** the variable must reside in RAM (readable and writable). Therefore:

- **LMA** = address in Flash (where the initial value 42 is stored)
- **VMA** = address in RAM (where `global_counter` is read/written at runtime)

The startup code's responsibility is to copy the `.data` section from LMA to VMA before `main()`.

### 5.2 AT> Syntax

```ld
.data :
{
    ...
} >RAM AT> FLASH
```

- `>RAM`: specifies VMA region as RAM
- `AT> FLASH`: specifies LMA region as Flash

If `AT>` is omitted, then LMA = VMA, and the section is both stored and runs in the same region.

### 5.3 Formal Representation

For section $S$:

$$\text{VMA}(S) = \text{Addr}(S \text{ in target region})$$
$$\text{LMA}(S) = \text{Addr}(S \text{ in load region})$$

Runtime invariant:

$$\forall x \in \text{.data}, \text{Mem}[\text{VMA}(x)] = \text{Mem}[\text{LMA}(x)] \text{ (after startup copy completes)}$$

---

## 6. Stack and Heap: Link-Time Checking

```ld
_estack = ORIGIN(RAM) + LENGTH(RAM);    /* End of RAM */
_sstack = _estack - _Min_Stack_Size;
_Min_Heap_Size = 0x800;      /* 2 KB */
_Min_Stack_Size = 0x800;     /* 2 KB */
```

### 6.1 Stack Growth Direction

The ARM Cortex-M stack is **Full Descending**:

- Stack pointer initial value = highest RAM address (`_estack`)
- On each PUSH, SP decrements by 4 first, then writes
- On each POP, reads SP first, then increments by 4

```
RAM high address ← _estack (SP initial value)
              │ Stack │  ↓ grows toward low addresses
              │       │
              │ Heap  │  ↑ grows toward high addresses
RAM low address ← _sdata
```

### 6.2 Minimum Stack Checking

The `._user_heap_stack` section ensures at link time that there is enough space in RAM:

```ld
._user_heap_stack (NOLOAD) :
{
    . = ALIGN(8);
    PROVIDE ( end = . );
    PROVIDE ( _end = . );
    . = . + _Min_Heap_Size;    /* Reserve 2KB heap */
    . = . + _Min_Stack_Size;   /* Reserve 2KB stack */
    . = ALIGN(8);
} >RAM
```

If `.bss` + heap + stack exceeds RAM capacity, the linker will report an error:

```
region RAM overflowed by 1234 bytes
```

> [!TIP]
> `_Min_Stack_Size = 0x800` (2KB) is typically a **minimum value**; actual stack usage depends on call depth and local variable sizes. For scenarios using an RTOS or nested interrupts, 4KB or larger may be needed. Runtime stack overflow detection can be implemented using the `__stack_limit` symbol or the MPU.

---

## 7. SECTIONS: Section Allocation in Detail

### 7.1 Interrupt Vector Table (.isr_vector)

```ld
.isr_vector :
{
    . = ALIGN(4);
    KEEP(*(.isr_vector))    /* Force retain, not optimized away by --gc-sections */
    . = ALIGN(4);
} >FLASH
```

**Why must it be placed at the very beginning of Flash?**

After power-on, the STM32 CPU begins fetching instructions from `0x00000000`. Address `0x00000000` is mapped (aliased) to the Flash start address `0x08000000` via SYSBOOT configuration. The CPU first reads:

| Offset | Content | Meaning |
|--------|---------|---------|
| 0x00 | Initial SP | Main stack pointer initial value |
| 0x04 | ResetHandler | Reset vector, jumps to startup code |
| 0x08 | NMIHandler | Non-maskable interrupt |
| 0x0C | HardFaultHandler | Hard fault |
| ... | ... | Other interrupts/exceptions |

**Purpose of `KEEP`**: The `--gc-sections` optimization removes unreferenced sections, but entries in `.isr_vector` have no explicit references (the CPU hardware jumps to them automatically). `KEEP` prevents them from being optimized away.

### 7.2 Code Section (.text)

```ld
.text :
{
    . = ALIGN(4);
    *(.text)           /* All .text sections */
    *(.text*)          /* .text.* named sections */
    *(.glue_7)         /* ARM → Thumb stub code */
    *(.glue_7t)        /* Thumb → ARM stub code */
    *(.eh_frame)       /* Exception handling frames (C++) */

    KEEP (*(.init))    /* Constructor initialization */
    KEEP (*(.fini))    /* Destructor cleanup */

    . = ALIGN(4);
    _etext = .;        /* Code section end symbol */
} >FLASH
```

**`.glue_7` / `.glue_7t`**: Stub (veneer) code for switching between ARM and Thumb instruction sets. Cortex-M4 only supports Thumb-2, but these sections may still be generated by the generic ARM toolchain.

**`_etext`**: An exported symbol marking the end of the code section. The startup code uses it to determine the starting position of the `.data` section in Flash (`_sidata = LOADADDR(.data)` is more precise).

### 7.3 Read-Only Data Section (.rodata)

```ld
.rodata :
{
    . = ALIGN(4);
    *(.rodata)         /* const global variables, string literals */
    *(.rodata*)        /* .rodata.* named sections */
    . = ALIGN(4);
} >FLASH
```

`.rodata` stores `const` global variables, string literals, and similar data. In embedded systems, this data stays in Flash and does not consume RAM.

**A common pitfall**:

```c
const char *msg = "hello";   // "hello" is in .rodata, the msg pointer is in .data
const char msg[] = "hello";  // Everything is in .rodata, more optimal
```

### 7.4 C++ Related Sections

```ld
.ARM.extab (READONLY) : { ... } >FLASH
.ARM (READONLY) : { ... } >FLASH
.preinit_array (READONLY) : { ... } >FLASH
.init_array (READONLY) : { ... } >FLASH
.fini_array (READONLY) : { ... } >FLASH
```

| Section | Purpose |
|---------|---------|
| `.ARM.extab` / `.ARM` | C++ exception handling unwind tables |
| `.preinit_array` | Earliest initialization function pointer array |
| `.init_array` | C++ global constructor function pointer array |
| `.fini_array` | C++ global destructor function pointer array |

The `READONLY` keyword is a GCC 11+ feature that tells the linker these sections are not writable. It must be removed when using GCC 10 or earlier.

**Pure C projects**: If there is no C++ code, these sections are typically empty. However, `KEEP` ensures they are not mistakenly deleted even with `--gc-sections`.

### 7.5 CCMRAM Section

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

- `>CCMRAM`: VMA is in the CCMRAM region
- `AT> FLASH`: LMA is in Flash (initial values are stored in Flash)
- `_siccmram`: LMA start address; the startup code uses this to copy data from Flash to CCMRAM

**Usage**: In C code, use `__attribute__` to place variables into CCMRAM:

```c
__attribute__((section(".ccmram"))) int fast_buffer[256];
```

> [!WARNING]
> The startup code must explicitly handle the CCMRAM copy. The default `startup_*.s` generated by STM32CubeMX may not include CCMRAM copy logic; it needs to be added manually.

### 7.6 Initialized Data Section (.data)

```ld
_sidata = LOADADDR(.data);   /* Start address of .data in Flash */

.data :
{
    . = ALIGN(4);
    _sdata = .;               /* VMA start */
    *(.data)
    *(.data*)
    *(.RamFunc)               /* Functions executed in RAM */
    *(.RamFunc*)
    . = ALIGN(4);
} >RAM AT> FLASH
```

**Exported symbols**:

| Symbol | Meaning |
|--------|---------|
| `_sidata` | LMA start of `.data` (in Flash) |
| `_sdata` | VMA start of `.data` (in RAM) |
| `_edata` | VMA end of `.data` (defined at the end of the .tdata section) |

**Startup copy pseudocode**:

```c
// Logic in startup_stm32f407xx.s
uint32_t *src = &_sidata;        // Source address in Flash
uint32_t *dst = &_sdata;         // Destination address in RAM
while (dst < &_edata) {
    *dst++ = *src++;
}
```

**`.RamFunc` section**: In some scenarios, functions need to be copied to RAM for execution (e.g., when Flash erase/programming makes it impossible to fetch instructions from the same Flash bank). Mark them with `__attribute__((section(".RamFunc")))`.

### 7.7 TLS (Thread-Local Storage) Section

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

TLS is the underlying support for C11 `_Thread_local` / GCC `__thread`. It is rarely used in bare-metal embedded systems, but when using an RTOS, each thread may need its own copy of certain variables.

**Exported symbols**:

| Symbol | Meaning |
|--------|---------|
| `__tdata_start` / `__tdata_end` | VMA range of `.tdata` |
| `__tdata_source` / `__tdata_source_end` | LMA range of `.tdata` |
| `__tls_size` | Total TLS size |
| `__tls_align` | TLS alignment requirement |

### 7.8 BSS Section (.bss)

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

**`(NOLOAD)`**: This section generates no output content; it only allocates address space. The BSS section stores uninitialized global/static variables, which are zeroed by the startup code at runtime:

```c
// Startup code zeros BSS
uint32_t *dst = &_sbss;
while (dst < &_ebss) {
    *dst++ = 0;
}
```

**`(COMMON)`**: Uninitialized global variables in C (tentative definitions) are placed in the COMMON section before linking, and merged into `.bss` at link time.

### 7.9 DISCARD Section

```ld
/DISCARD/ :
{
    libc.a:* ( * )
    libm.a:* ( * )
    libgcc.a:* ( * )
}
```

Discards certain content from the standard libraries. In deep embedded contexts (newlib-nano, nosys), this avoids linking unneeded library code and reduces firmware size.

---

## 8. ALIGN and Address Alignment

```ld
. = ALIGN(4);
```

- `.` is the **location counter**, representing the current output offset
- `ALIGN(4)` aligns the location counter up to a 4-byte boundary
- ARM Cortex-M requires all accesses to be aligned to their natural boundary (32-bit to 4 bytes, 64-bit to 8 bytes)

**Why does .isr_vector need ALIGN(4)?**

Each entry in the interrupt vector table is 4 bytes (a 32-bit address) and must be 4-byte aligned; otherwise, the CPU will fault on instruction fetch.

---

## 9. PROVIDE and Symbol Visibility

```ld
PROVIDE( __bss_end = . );
```

`PROVIDE` defines a **weak symbol**: it takes effect only when no other definition of the same symbol exists in the program. If the C code defines `__bss_end`, the linker uses the C code's definition and ignores the `PROVIDE`.

**Difference from direct assignment**:

| Syntax | Behavior |
|--------|----------|
| `_sbss = .;` | Forced definition; if the program also has a definition, a multiple definition error is reported |
| `PROVIDE(_sbss = .);` | Only provided when undefined; allows the program to override |

---

## 10. KEEP and --gc-sections

`--gc-sections` (`-ffunction-sections -fdata-sections` + `-Wl,--gc-sections`) is a key optimization for slimming embedded firmware: the compiler generates independent sections for each function and data object, and the linker removes unreferenced sections.

**Problem**: The interrupt vector table, constructor/destructor arrays, etc. have no explicit references and would be mistakenly removed.

**Solution**: `KEEP()` tells the linker "retain this section, even if it appears unreferenced."

```ld
KEEP(*(.isr_vector))    /* Vector table: CPU hardware jumps, no software reference */
KEEP (*(.init))         /* C++ initialization */
KEEP (*(.fini))         /* C++ cleanup */
KEEP (*(SORT(.init_array.*)))  /* Constructor function pointer array */
```

---

## 11. Cooperation Between Startup Code and Linker Script

The linker script defines symbols; the startup code (`startup_stm32f407xx.s`) uses these symbols to perform initialization. The two must strictly cooperate:

```
┌──────────────────────┐        ┌──────────────────────┐
│     Linker Script (.ld)│        │  Startup Code (.s)   │
│                      │        │                      │
│  _estack             │───────→│  Initial SP value    │
│  _sidata             │───────→│  .data Flash source  │
│  _sdata, _edata     │───────→│  .data RAM range     │
│  _sbss, _ebss       │───────→│  .bss RAM range      │
│  _siccmram           │───────→│  .ccmram Flash src   │
│  _sccmram, _eccmram  │───────→│  .ccmram CCMRAM range│
└──────────────────────┘        └──────────────────────┘
```

### 11.1 Complete Startup Flow

```asm
Reset_Handler:
    /* 1. Set stack pointer (linker script _estack) */
    ldr  sp, =_estack

    /* 2. Copy .data from Flash to RAM */
    ldr  r0, =_sdata      /* dst start */
    ldr  r1, =_edata      /* dst end */
    ldr  r2, =_sidata     /* src start (LMA) */
CopyData:
    cmp  r0, r1
    ittt lt
    ldrlt r3, [r2], #4
    strlt r3, [r0], #4
    blt  CopyData

    /* 3. Zero .bss */
    ldr  r0, =_sbss
    ldr  r1, =_ebss
    movs r2, #0
ClearBSS:
    cmp  r0, r1
    itt  lt
    strlt r2, [r0], #4
    blt  ClearBSS

    /* 4. Call SystemInit() */
    bl   SystemInit

    /* 5. Call __libc_init_array() (C++ constructors) */
    bl   __libc_init_array

    /* 6. Call main() */
    bl   main

    /* 7. main should not return; if it does, infinite loop */
LoopForever:
    b    LoopForever
```

### 11.2 C++ Constructor Invocation

`__libc_init_array()` iterates over the function pointers in the `.init_array` section and calls each one. This section is guaranteed to exist and be in the correct order by `KEEP (*(SORT(.init_array.*)))` in the linker script (`SORT` ensures sorting by priority).

---

## 12. Practical Use of Custom Sections

### 12.1 Placing Variables in CCMRAM

```c
// The .ccmram section is already defined in the linker script
__attribute__((section(".ccmram"))) int ai_tensor_buf[1024];  // AI inference tensor buffer
```

Placing AI inference tensor buffers in CCMRAM provides zero-wait-state access, improving inference speed.

### 12.2 Placing Functions in RAM for Execution

```c
// The .RamFunc section is already defined in the linker script
__attribute__((section(".RamFunc"))) void flash_erase_and_write(void);
```

On STM32F407, Flash cannot be fetched from the same bank during erase/programming. Placing Flash operation functions in RAM avoids this issue.

### 12.3 NoInit Region

```c
// Variables not initialized by startup code (retain values after soft reset)
__attribute__((section(".bss.NoInit"))) volatile uint32_t update_flag;
```

When a Bootloader and App communicate via soft reset, `NoInit` variables are not zeroed by the startup code and can pass state across resets.

---

## 13. Common Issues and Debugging

### 13.1 region overflowed

```
region RAM overflowed by 1234 bytes
```

Cause: `.data` + `.bss` + heap + stack exceeds RAM capacity.

Investigation:
1. Check section sizes in the `.map` file
2. Check if large arrays are mistakenly placed in RAM (consider using Flash or external SRAM instead)
3. Increase `_Min_Stack_Size` or optimize code

### 13.2 Multiple Definition Error

```
multiple definition of `_sbss'
```

Using `_sbss = .;` in the linker script is a strong definition; if the C code also declares a symbol with the same name, there is a conflict. Use `PROVIDE(_sbss = .)` instead, or rename the C symbol.

### 13.3 Incorrect .data Copy

Symptom: Global variable initial values are not as expected.

Investigation:
1. Verify the values of `_sidata` and `_sdata`: `objdump -t firmware.elf | grep -E '_s(data|idata)'`
2. Confirm the startup code copy logic covers the full range from `.data` to `.tdata`
3. Check if CCMRAM copy was missed

### 13.4 Viewing Section Layout

```bash
# View VMA and LMA of each section
arm-none-eabi-objdump -h firmware.elf

# View complete memory map
arm-none-eabi-objdump -t firmware.elf | sort

# View section size summary
arm-none-eabi-size firmware.elf
   text    data     bss     dec     hex filename
  45678    1234    8900   55812    d9e4 firmware.elf
```

In the `size` output:
- `text`: .text + .rodata (Flash usage)
- `data`: .data (Flash stores initial values + RAM runtime usage)
- `bss`: .bss (RAM only)
- Total Flash usage = text + data
- Total RAM usage = data + bss

---

## 14. Summary

| Concept | Key Understanding |
|---------|-------------------|
| MEMORY | Declares the chip's physical memory regions and attributes |
| SECTIONS | Specifies the arrangement order and placement of each section |
| VMA / LMA | Runtime address vs load-time address; .data must be copied |
| >RAM AT> FLASH | VMA in RAM, LMA in Flash |
| ALIGN | Ensures hardware-required address alignment |
| KEEP | Prevents --gc-sections from mistakenly removing critical sections |
| PROVIDE | Weak symbol definition, allows program to override |
| _s*/_e* symbols | Boundary addresses exported for use by startup code |
| .bss (NOLOAD) | Generates no output, only allocates addresses; zeroed at startup |
| /DISCARD/ | Discards unneeded standard library content |

A linker script is not a "set it and forget it" black box; it is the bridge connecting hardware memory topology, compiler output, and startup code. Understanding it empowers you to navigate memory optimization, multi-partition bootloaders, custom sections, and other advanced scenarios with ease.

---

## References

- [GNU LD Manual - Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html)
- [STM32F4xx Reference Manual - Memory Map](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [GCC Attribute Documentation](https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html)
- [ARM Cortex-M4 Technical Reference Manual](https://developer.arm.com/documentation/ddi0343/latest/)
