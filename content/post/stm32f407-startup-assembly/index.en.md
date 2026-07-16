---
title: "Deep Dive into the STM32F407 Assembly Startup File: The Complete Journey from Reset Vector to C main()"
date: 2026-07-11
description: "Line-by-line analysis of the startup_stm32f407xx.s assembly startup file, covering processor instruction set declarations, vector table structure, Reset_Handler boot flow (stack pointer init → .data copy → .bss zero → C lib init → main call), weak symbol interrupt mechanism and .thumb_set alias technique"
image: STM32F407汇编启动文件解析.png
categories:
  - "Embedded"
  - "Startup"
tags:
  - "STM32"
  - "Assembly"
  - "Startup File"
  - "Cortex-M4"
  - "Vector Table"
  - "GCC"
---

## Preface

Every STM32 project contains a `startup_stm32f407xx.s` file, quietly sitting in the project root, rarely needing modification. But if you've ever wondered: **Where does the first instruction come from after power-on? Who initializes global variables? How are interrupt service functions "registered"?** — the answers are all in this file.

This article will use the GCC version of `startup_stm32f407xx.s` as the reference, breaking down its mechanisms section by section, revealing the complete journey from hardware reset to C language `main()` execution.

> [!NOTE]
> The startup file analyzed in this article comes from the STM32CubeF4 HAL library, for the GCC/ARM toolchain. The Keil version (ARMASM syntax) and IAR version have similar structure but different assembly syntax.

---

## 1. Overall File Structure

The startup file can be divided into four logical regions:

| Region | Function | Line Range |
|--------|----------|-----------|
| Header Declarations | Processor architecture, instruction set, FPU type | Lines 1-31 |
| Linker Symbols | .data / .bss section address references | Lines 35-46 |
| Reset_Handler | Core boot initialization flow | Lines 57-102 |
| Vector Table + Weak Symbols | Interrupt vector table and default handler | Lines 111-508 |

---

## 2. Header Declarations: Telling the Assembler Who We Are

```asm
.syntax unified
.cpu cortex-m4
.fpu softvfp
.thumb
```

Line-by-line analysis:

| Directive | Meaning | Why It's Needed |
|-----------|---------|-----------------|
| `.syntax unified` | Use unified assembly syntax (ARM + Thumb instructions share the same syntax) | GCC defaults to `divided` syntax; unified is more modern and concise |
| `.cpu cortex-m4` | Target processor is Cortex-M4 | Affects instruction encoding and available instruction set; Cortex-M4 supports DSP instructions and optional FPU |
| `.fpu softvfp` | Use software floating-point calling convention | Even with hardware FPU, software FP ABI ensures compatibility with soft-float libraries; switch to `fpv4-sp-d16` to enable hardware floating-point |
| `.thumb` | Use Thumb instruction set | Cortex-M series **only supports** Thumb instruction set (16/32-bit mixed encoding); this is a hardware mandate |

> [!TIP]
> If your project needs to use the hardware FPU (Floating Point Unit), you must modify both `.fpu fpv4-sp-d16` and the linker flag `-mfloat-abi=hard`. Modifying the startup file alone is not enough — the compiler, assembler, and linker must maintain ABI consistency.

Next, two global symbols are declared:

```asm
.global  g_pfnVectors
.global  Default_Handler
```

- `g_pfnVectors`: The starting label of the vector table; the linker needs to know its address to place interrupt vectors
- `Default_Handler`: The default interrupt handler; all unimplemented interrupts point to it

---

## 3. Linker Symbols: Address Anchors for .data and .bss

```asm
.word  _sidata    /* Start address of .data initial values in Flash */
.word  _sdata     /* Start address of .data section in SRAM */
.word  _edata     /* End address of .data section in SRAM */
.word  _sbss      /* Start address of .bss section in SRAM */
.word  _ebss      /* End address of .bss section in SRAM */
```

These five `.word` directives are not code, but **data reserved for Reset_Handler** — they reference symbols defined in the linker script (`.ld` file) via label names. Understanding these symbols is the key to understanding the boot flow.

### 3.1 Why Does .data Need to Be Copied?

Global variables with initial values in C (e.g., `int x = 42;`) are stored in the `.data` section. But `.data` has two "homes":

| Location | Content | Reason |
|----------|---------|--------|
| Flash (`_sidata`) | Initial values (42) | Flash retains data without power; stores initial values |
| SRAM (`_sdata` ~ `_edata`) | Runtime variables | SRAM is read-write; used during program execution |

At power-on, SRAM content is random; initial values must be **copied** from Flash to SRAM for `.data` variables to have correct initial values. This is what the "Copy Data" loop in Reset_Handler does.

### 3.2 Why Does .bss Only Need Zeroing?

Uninitialized global variables in C (e.g., `int y;`) are stored in the `.bss` section. The C standard specifies that uninitialized global variables have a value of 0. The `.bss` section doesn't occupy Flash space (no initial values to store); it only needs the corresponding SRAM region zeroed at startup.

> [!IMPORTANT]
> The names of these five symbols are **conventional**, defined by the linker script. If using a custom `.ld` file, you must ensure these symbol names match the references in the startup file. Common naming variants include `__data_start` / `__data_end`, etc.

---

## 4. Reset_Handler: The Core of the Boot Flow

This is the most important part of the entire file — the first meaningful code executed after the chip powers on.

### 4.1 Complete Code with Line-by-Line Annotations

```asm
    .section  .text.Reset_Handler
  .weak  Reset_Handler
  .type  Reset_Handler, %function
Reset_Handler:
  ldr   sp, =_estack       /* [1] Set stack pointer */

  bl  SystemInit            /* [2] Call system initialization */

  /* [3] Copy .data section from Flash to SRAM */
  ldr r0, =_sdata           /*   r0 = .data start address in SRAM */
  ldr r1, =_edata           /*   r1 = .data end address in SRAM */
  ldr r2, =_sidata          /*   r2 = .data initial values start address in Flash */
  movs r3, #0               /*   r3 = offset, initialized to 0 */
  b LoopCopyDataInit        /*   Jump to loop condition check */

CopyDataInit:
  ldr r4, [r2, r3]          /*   Read 4 bytes from Flash[r2+r3] to r4 */
  str r4, [r0, r3]          /*   Write r4 to SRAM[r0+r3] */
  adds r3, r3, #4           /*   offset += 4 (word-by-word copy) */

LoopCopyDataInit:
  adds r4, r0, r3           /*   r4 = current write address = _sdata + offset */
  cmp r4, r1                /*   current address < _edata ? */
  bcc CopyDataInit          /*   Yes: continue copying (bcc = branch if carry clear, i.e. unsigned less than) */

  /* [4] Zero fill the .bss section */
  ldr r2, =_sbss            /*   r2 = .bss start address */
  ldr r4, =_ebss            /*   r4 = .bss end address */
  movs r3, #0               /*   r3 = 0 */
  b LoopFillZerobss         /*   Jump to loop condition check */

FillZerobss:
  str  r3, [r2]             /*   Write 0 to [r2] */
  adds r2, r2, #4           /*   address += 4 */

LoopFillZerobss:
  cmp r2, r4                /*   current address < _ebss ? */
  bcc FillZerobss           /*   Yes: continue zeroing */

  /* [5] Call C library initialization */
  bl __libc_init_array

  /* [6] Call main() */
  bl  main
  bx  lr                    /*   If main() returns, jump back to caller */
.size  Reset_Handler, .-Reset_Handler
```

### 4.2 Six Steps Explained

#### Step [1]: Set Stack Pointer

```asm
ldr   sp, =_estack
```

`_estack` is the top-of-stack address, defined in the linker script (typically the end of SRAM, e.g., `0x20020000`). The Cortex-M4 stack is **Full Descending** — SP is decremented before writing on push. Therefore, the top-of-stack address points to the highest SRAM address + 1 (i.e., the byte after the SRAM end).

> [!NOTE]
> The Cortex-M4 hardware automatically loads SP from vector table offset 0 (i.e., `_estack`) on reset, so this line appears redundant. However, it's defensive programming: ensuring SP is correct even if the vector table is misconfigured.

#### Step [2]: Call SystemInit

```asm
bl  SystemInit
```

`SystemInit()` is typically defined in `system_stm32f4xx.c`, responsible for:

- Configuring Flash wait states (LATENCY)
- Enabling PLL and switching the system clock to maximum frequency (168MHz for STM32F407)
- Configuring AHB/APB1/APB2 prescalers

The `bl` (Branch with Link) instruction saves the return address to the `lr` register while jumping; the function returns via `bx lr` to the call site.

#### Step [3]: Copy .data Section

This is the most elegant code in the startup file, using **offset addressing** to copy data from Flash to SRAM:

| Register | Role | Value |
|----------|------|-------|
| r0 | SRAM target start address | `_sdata` |
| r1 | SRAM target end address | `_edata` |
| r2 | Flash source start address | `_sidata` |
| r3 | Offset (increments by 4 each iteration) | 0 → 4 → 8 → ... |
| r4 | Temporary register | Current address / data |

Core loop logic:

```asm
CopyDataInit:
  ldr r4, [r2, r3]    /* Source read: Flash[_sidata + offset] */
  str r4, [r0, r3]    /* Target write: SRAM[_sdata + offset] */
  adds r3, r3, #4     /* Increment offset */
LoopCopyDataInit:
  adds r4, r0, r3     /* r4 = _sdata + offset = current write address */
  cmp r4, r1          /* Compare with _edata */
  bcc CopyDataInit    /* Not at end address, continue */
```

> [!TIP]
> Why use offset `[r2, r3]` instead of directly incrementing pointers? Because the source (Flash) and target (SRAM) have different base addresses, but the offset is the same. Using a single offset `r3` to index both source and target is the most concise approach, requiring only one counter.

#### Step [4]: Zero .bss Section

Similar to .data copying, but simpler — no need to read from Flash, just write 0:

```asm
FillZerobss:
  str  r3, [r2]       /* r3 = 0, write to [r2] */
  adds r2, r2, #4     /* Increment address */
LoopFillZerobss:
  cmp r2, r4          /* current address < _ebss ? */
  bcc FillZerobss     /* Continue zeroing */
```

Here `r2` (the current address pointer) is directly incremented, unlike the offset approach used for .data. This is because .bss zeroing only needs one moving pointer — no need to maintain both source and target addresses.

#### Step [5]: Call C Library Initialization

```asm
bl __libc_init_array
```

This step is easily overlooked but crucial for C++ and certain C code. `__libc_init_array` performs two key operations:

1. **Calls function pointers in the `.init_array` section** — C++ global object constructors (`__libc_init_array` internally calls `.preinit_array` first, then `.init_array`)
2. **Calls functions decorated with `__attribute__((constructor))`**

> [!IMPORTANT]
> If your project is pure C with no `__attribute__((constructor))`, this step could theoretically be skipped. However, the standard practice is to always call it, because:
> - Some C library initialization logic depends on it
> - If C++ code is introduced later, omitting it won't cause bugs
> - The overhead is minimal (nearly a no-op when `.init_array` is empty)

#### Step [6]: Call main()

```asm
bl  main
bx  lr
```

After all the above preparation, the C runtime environment is finally ready: the stack is set, global variables are initialized, and the C library is ready. It's now safe to call `main()`.

If `main()` somehow returns (it shouldn't in embedded programs), `bx lr` would jump back to the caller — but the caller is Reset_Handler itself, and `lr`'s value is undefined at this point, so the behavior is unpredictable. In a robust system, `main()` should be an infinite loop that never returns.

### 4.3 Boot Flow Overview

```
Power-on / Reset
  │
  ▼
Hardware auto-loads SP = _estack from vector table  ← Offset 0x00
Hardware auto-loads PC = Reset_Handler from vector table  ← Offset 0x04
  │
  ▼
Reset_Handler executes
  ├─ [1] ldr sp, =_estack          Re-set stack pointer (defensive)
  ├─ [2] bl SystemInit             Configure system clock
  ├─ [3] Copy .data: Flash → SRAM  Initialize global variables
  ├─ [4] Zero .bss: SRAM clear     Initialize uninitialized globals
  ├─ [5] bl __libc_init_array      C++ constructors / constructor attr
  └─ [6] bl main                   Enter user program
```

---

## 5. Default_Handler: Safety Net for Unexpected Interrupts

```asm
    .section  .text.Default_Handler,"ax",%progbits
Default_Handler:
Infinite_Loop:
  b  Infinite_Loop
  .size  Default_Handler, .-Default_Handler
```

If the CPU enters an unimplemented interrupt handler, it executes `Default_Handler` — an infinite loop. Its purpose is to **halt the program in place**, making it easy for a debugger to capture the scene:

- The interrupt return address (`lr`) is still preserved in registers
- The debugger can see the program stuck in `Default_Handler`, immediately knowing an unhandled interrupt occurred
- Register and stack contents are undisturbed, allowing analysis of the interrupt source

`.section` attributes explained:

| Attribute | Meaning |
|-----------|---------|
| `"ax"` | allocatable + executable |
| `%progbits` | Section contains actual data (not BSS-style zero-fill) |

> [!WARNING]
> In production environments, the infinite loop in `Default_Handler` causes the device to "freeze" — appearing to run but no longer responding. A better approach is to execute a soft reset (`NVIC_SystemReset()`) in `Default_Handler`, or at least log the exception information.

---

## 6. Vector Table: The Interrupt Routing Table

### 6.1 Vector Table Structure

The vector table is the core of the Cortex-M4 boot mechanism; it must be placed at the beginning of Flash (`0x08000000`).

```asm
   .section  .isr_vector,"a",%progbits
  .type  g_pfnVectors, %object

g_pfnVectors:
  .word  _estack               /* Offset 0x00: Stack top address */
  .word  Reset_Handler         /* Offset 0x04: Reset vector */
  .word  NMI_Handler           /* Offset 0x08: Non-maskable interrupt */
  .word  HardFault_Handler     /* Offset 0x0C: Hard fault */
  .word  MemManage_Handler     /* Offset 0x10: Memory management fault */
  .word  BusFault_Handler      /* Offset 0x14: Bus fault */
  .word  UsageFault_Handler    /* Offset 0x18: Usage fault */
  .word  0                     /* Offset 0x1C: Reserved */
  .word  0                     /* Offset 0x20: Reserved */
  .word  0                     /* Offset 0x24: Reserved */
  .word  0                     /* Offset 0x28: Reserved */
  .word  SVC_Handler           /* Offset 0x2C: Supervisor call */
  .word  DebugMon_Handler      /* Offset 0x30: Debug monitor */
  .word  0                     /* Offset 0x34: Reserved */
  .word  PendSV_Handler        /* Offset 0x38: Pendable service */
  .word  SysTick_Handler       /* Offset 0x3C: SysTick */
  /* External interrupts start from offset 0x40... */
```

### 6.2 Cortex-M4 Exception Vector Details

The first 16 vectors are **system exceptions** defined by the ARM Cortex-M4 architecture, identical across all M4 chips:

| Offset | Exception # | Name | Priority | Typical Trigger |
|--------|------------|------|----------|-----------------|
| 0x00 | - | Initial SP | - | Hardware auto-loads |
| 0x04 | 1 | Reset | -3 (highest) | Power-on / reset |
| 0x08 | 2 | NMI | -2 | External NMI pin / Watchdog |
| 0x0C | 3 | HardFault | -1 | Fallback when other exceptions can't be handled |
| 0x10 | 4 | MemManage | Configurable | Accessing MPU-protected region |
| 0x14 | 5 | BusFault | Configurable | Accessing invalid address (e.g., unmapped peripheral) |
| 0x18 | 6 | UsageFault | Configurable | Executing undefined instruction / divide by zero |
| 0x2C | 11 | SVCall | Configurable | Executing `SVC` instruction (RTOS system call) |
| 0x30 | 12 | Debug Monitor | Configurable | Debug event |
| 0x38 | 14 | PendSV | Configurable | RTOS context switch |
| 0x3C | 15 | SysTick | Configurable | SysTick timer overflow |

> [!NOTE]
> Lower priority numbers indicate higher priority. Reset (-3), NMI (-2), and HardFault (-1) are fixed and non-configurable. Other exception priorities can be programmed via NVIC registers.

### 6.3 STM32F407 External Interrupt Vectors

Starting from offset 0x40 are chip-specific external interrupts; the STM32F407 has 82:

| Exception # | Interrupt Name | Peripheral | Description |
|-------------|---------------|------------|-------------|
| 16 | WWDG_IRQHandler | WWDG | Window Watchdog |
| 17 | PVD_IRQHandler | PVD | Power Voltage Detector |
| 18 | TAMP_STAMP_IRQHandler | RTC | Tamper and Timestamp |
| ... | ... | ... | ... |
| 25 | EXTI0_IRQHandler | EXTI | External interrupt line 0 |
| ... | ... | ... | ... |
| 54 | USART1_IRQHandler | USART1 | Serial port 1 |
| 67 | ETH_IRQHandler | Ethernet | Ethernet |
| 97 | FPU_IRQHandler | FPU | Floating Point Unit |

Some positions are filled with 0 (reserved), such as offsets 0x1C-0x28 and the CRYP position. Writing 0 to reserved positions ensures the vector table size and offsets are correct.

### 6.4 Hardware Vector Table Loading Mechanism

Cortex-M4 hardware behavior on reset (purely hardware, no software involvement):

1. Read 4 bytes from address `0x00000000` (mapped to `0x08000000`) → load into **SP** (Main Stack Pointer)
2. Read 4 bytes from address `0x00000004` (mapped to `0x08000000+4`) → load into **PC** (Program Counter)
3. CPU begins executing from the address PC points to — i.e., `Reset_Handler`

> [!TIP]
> The STM32 Flash start address `0x08000000` is aliased to `0x00000000` (configured via BOOT pins). This is why the vector table is placed at the Flash beginning, but hardware reads from `0x00000000` — they point to the same physical storage.

---

## 7. Weak Symbol Mechanism: "Placeholders" for Interrupt Handlers

### 7.1 How .weak + .thumb_set Works

The final section of the startup file is a long series of repeated patterns:

```asm
   .weak      NMI_Handler
   .thumb_set NMI_Handler,Default_Handler

   .weak      HardFault_Handler
   .thumb_set HardFault_Handler,Default_Handler

   /* ... 80+ interrupts ... */

   .weak      FPU_IRQHandler
   .thumb_set FPU_IRQHandler,Default_Handler
```

These two directives form an elegant **default value mechanism**:

| Directive | Meaning |
|-----------|---------|
| `.weak NMI_Handler` | Declares `NMI_Handler` as a weak symbol — if another file defines a symbol with the same name, the weak definition is overridden |
| `.thumb_set NMI_Handler, Default_Handler` | Sets the value of `NMI_Handler` to the address of `Default_Handler` (Thumb function alias) |

### 7.2 Linker Behavior with Weak Symbols

```
Scenario 1: User has not implemented NMI_Handler
  → Linker uses weak definition → NMI_Handler = Default_Handler → infinite loop

Scenario 2: User implements NMI_Handler in stm32f4xx_it.c
  → Linker chooses strong definition → NMI_Handler = user function → normal execution
```

This is the most common interrupt registration method in embedded development — **no need to modify the startup file; just define a function with the same name in C code**:

```c
// stm32f4xx_it.c
void NMI_Handler(void) {
    // User-defined NMI handling logic
}
```

The linker sees the strong symbol `NMI_Handler`, automatically ignores the weak definition in the startup file, and the corresponding entry in the vector table automatically points to the user function.

### 7.3 .thumb_set vs .set

`.thumb_set` is the Thumb-specific variant of `.set`; it automatically sets bit[0] of the address to 1 (Thumb interwork flag):

| Directive | Address bit[0] | Use Case |
|-----------|----------------|----------|
| `.set` | Unmodified | ARM mode (not applicable for Cortex-M) |
| `.thumb_set` | Auto-set to 1 | Thumb mode (must be used for Cortex-M) |

Cortex-M4 only supports Thumb instructions; the address bit[0] in interrupt vectors must be 1, otherwise a HardFault is triggered. Using `.thumb_set` ensures this.

> [!IMPORTANT]
> If you write assembly interrupt handlers manually, you must ensure the function symbol's bit[0] is 1. GCC-compiled C functions handle this automatically, but hand-written assembly requires declaring with `.thumb_func` or setting with `.thumb_set`.

---

## 8. Collaboration Between Startup File and Linker Script

The startup file doesn't work alone; it depends on symbols defined in the linker script (`.ld`) and functions provided by the C runtime. Complete dependency relationships:

| Reference in Startup File | Defined In | Type |
|--------------------------|-----------|------|
| `_estack` | Linker script | Stack top address |
| `_sidata`, `_sdata`, `_edata` | Linker script | .data section addresses |
| `_sbss`, `_ebss` | Linker script | .bss section addresses |
| `SystemInit` | `system_stm32f4xx.c` | Clock initialization function |
| `__libc_init_array` | C runtime library (libc) | Constructor caller |
| `main` | User code | Program entry point |

### 8.1 Key Linker Script Snippet

```ld
/* Typical STM32F407 linker script */
MEMORY {
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 1024K
  RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS {
  .isr_vector : {
    . = ALIGN(4);
    KEEP(*(.isr_vector))    /* Vector table, cannot be optimized away */
    . = ALIGN(4);
  } > FLASH

  .text : {
    *(.text*)
    *(.rodata*)
  } > FLASH

  _sidata = LOADADDR(.data);  /* Load address of .data in Flash */

  .data : {
    _sdata = .;
    *(.data*)
    _edata = .;
  } > RAM AT > FLASH          /* Runs in RAM, loaded in Flash */

  .bss : {
    _sbss = .;
    *(.bss*)
    *(COMMON)
    _ebss = .;
  } > RAM

  _estack = ORIGIN(RAM) + LENGTH(RAM);  /* Stack top = end of RAM */
}
```

> [!NOTE]
> `.data : { ... } > RAM AT > FLASH` is the key syntax — it means the `.data` section's **runtime address** is in RAM (`> RAM`), but its **load address** is in Flash (`AT > FLASH`). `LOADADDR(.data)` returns the load address (i.e., the position in Flash), which is the value of `_sidata`.

---

## 9. Common Issues and Troubleshooting

### 9.1 HardFault Infinite Loop

**Symptom**: Program stuck in `Default_Handler` (actually `HardFault_Handler`).

**Troubleshooting steps**:

1. Pause in debugger, examine `PC` and `LR` registers
2. Check `SCB->HFSR`, `SCB->CFSR`, `SCB->MMFAR`/`SCB->BFAR` registers
3. Common causes:
   - Null pointer dereference (`SCB->MMFAR` points to illegal address)
   - Stack overflow (SP out of SRAM range)
   - Function address in vector table has bit[0] = 0 (non-Thumb address)

### 9.2 Incorrect Global Variable Initial Values

**Symptom**: Global variables with initial values are wrong, or change across reboots.

**Cause**: The .data copy in Reset_Handler was skipped or executed incorrectly.

**Troubleshooting**:

- Check the linker script correctly defines `_sidata`, `_sdata`, `_edata`
- Check if a custom Reset_Handler overrode the weak symbol without implementing the copy logic
- Confirm the vector table `.isr_vector` section uses `KEEP()` to prevent optimization

### 9.3 C++ Global Objects Not Constructed

**Symptom**: C++ global object constructors are not called.

**Cause**: `__libc_init_array` was omitted. Some minimal startup files (or custom Reset_Handler) may not call it.

### 9.4 Interrupt Handler Not Working

**Symptom**: Defined an interrupt handler but it's never called; always enters infinite loop.

**Troubleshooting**:

- Confirm the function name in the C file **exactly matches** the weak symbol name in the startup file (case-sensitive)
- Confirm the `stm32f4xx_it.c` file is included in compilation
- Check NVIC interrupt enable is configured

---

## 10. Startup File Modification Guide

### 10.1 When to Modify the Startup File

| Scenario | Modification |
|----------|-------------|
| Using external SDRAM for extended memory | Add SDRAM initialization code in Reset_Handler |
| Changing heap/stack size | Modify `_estack` in the linker script or increase `_Min_Heap_Size`/`_Min_Stack_Size` |
| Adding custom interrupts | Vector table usually doesn't need changes — weak symbol mechanism handles it automatically |
| Bootloader jump scenario | Adjust vector table offset (`SCB->VTOR`); startup file itself usually doesn't need changes |
| Using hardware FPU | Change `.fpu` to `fpv4-sp-d16` and enable FPU in SystemInit |

### 10.2 Adding Custom Reset Initialization Logic

If you need to execute additional initialization before entering `main()` (e.g., watchdog feeding, early peripheral configuration), there are two approaches:

**Approach 1: Modify SystemInit (Recommended)**

Add code to `SystemInit()` in `system_stm32f4xx.c`; no need to modify the assembly file.

**Approach 2: Insert Assembly Code in Reset_Handler**

Insert custom assembly calls between `bl SystemInit` and `bl main`. This approach requires understanding the ARM calling convention (AAPCS) to ensure register state is not corrupted.

---

## 11. Summary

Although `startup_stm32f407xx.s` is only about 500 lines, it carries the full responsibility of establishing the C runtime environment from hardware reset. Key takeaways:

1. **Header declarations** determine the target architecture and instruction set — Cortex-M4 only supports Thumb, a hardware mandate
2. **Linker symbols** are the bridge between the startup file and the linker script — .data copying and .bss zeroing depend on these symbols
3. **Reset_Handler** is the core of the boot flow — SP init → clock config → data copy → BSS zero → C lib init → main()
4. **Weak symbol mechanism** allows interrupt registration without modifying the startup file — just define a function with the same name in C code
5. **Vector table** is the contract between hardware and software — hardware auto-loads SP and PC on reset; subsequent interrupts are also routed through it

Understanding the startup file means understanding the entire process of an embedded system going "from zero to main()".

---

## References

- [ARM Cortex-M4 Technical Reference Manual](https://developer.arm.com/documentation/ddi0439/latest/)
- [STM32F4xx Reference Manual (RM0090)](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [ARM Architecture Procedure Call Standard (AAPCS)](https://github.com/ARM-software/abi-aa/blob/main/aapcs32/aapcs32.rst)
- [GNU Assembler Manual](https://sourceware.org/binutils/docs/as/)
- [Cortex-M4 Devices Generic User Guide](https://developer.arm.com/documentation/dui0553/latest/)
