---
title: "A Taste of Bootloader Five-Layer Architecture: OOP Object Design Model from Platform Abstraction to App Orchestration"
date: 2026-07-03
description: "Using the STM32F407 Secure Bootloader as a real-world example, this post explores the five-layer Impl/Platform/Service/Core/App + Drivers(Vendor) architecture's C object-oriented design model: ops virtual function tables, container_of downcasting, multiple inheritance with dual base classes, dependency injection containers, macro-wrapped safe calls, mapped one-to-one to oop-in-c-embedded concepts"
image: C语言结构体指针博客封面.png
categories:
  - "Embedded"
tags:
  - "STM32"
  - "Bootloader"
  - "OOP-in-C"
  - "Layered-Architecture"
  - "Object-Oriented"
  - "Platform-Abstraction"
math: true
---

## Introduction

In the previous post [The Essence of C: Structs and Pointers](../oop-in-c-embedded/), we started from `struct + me` and walked all the way to Linux kernel-style `gpio_chip` + initcall auto-registration. The theory is there, but what about practice?

This post uses the real code from the [STM32F407 Secure Bootloader](../stm32f407-secure-bootloader/) project as an example, taking a **taste** of its five-layer architecture object design model—seeing how the essence of OOP in C takes root in an industrial-grade Bootloader.

> [!NOTE]
> "Taste" doesn't mean superficial—it means focusing on the design intent and pattern mapping of each layer, rather than line-by-line interpretation.

---

## 1. Five-Layer Architecture Overview

### 1.1 Layer Diagram

| Layer | Directory | Responsibility | OOP Concept Mapping |
|-------|-----------|---------------|-------------------|
| **App** | `Core/Src/main.c` | System orchestration, startup flow, conditional branching | Top-level composition root / main orchestration |
| **Core** | `Bootloader_Core/` | Firmware download engine, A/B partition, jump logic | Business strategy / state machine |
| **Service** | `Service/` | YMODEM, OTA, menu, diff, decrypt, signature | Service objects / DI consumers |
| **Platform** | `Platform/Inc/` | Abstract interface definitions (ops vtable + base struct + macros) | Pure virtual base class / interface |
| **Impl** | `Impl/` | STM32 / ESP8266 concrete implementations | Derived classes / vtable instantiation |
| **Drivers** | `Drivers/` | HAL + Vendor drivers (STM32 HAL, W25Q128, FatFs...) | Hardware abstraction layer |

Call direction: **App → Core → Service → Platform ← Impl → Drivers**

Key design: **Service does not depend on Impl, only on Platform's abstract interfaces**. Impl "injects" concrete implementations into Platform's base instances at startup via register functions—this is [dependency injection](../oop-in-c-embedded/) realized in C.

### 1.2 Why Five Layers Instead of Three?

The classic three-layer architecture (App → HAL → Hardware) has a problem: business logic is coupled with hardware implementation. Adding a Service layer isolates business protocols, a Platform layer provides polymorphic abstraction, and an Impl layer carries concrete implementations—each layer depends only on the **abstraction** of the next layer, not its **implementation**.

---

## 2. Platform Layer: The Pure Virtual Interface Definer

The Platform layer is the "interface contract" of the entire architecture, defining only ops virtual function tables and base structs with no implementation code. Compare with the [ops operation table concept from oop-in-c-embedded](../oop-in-c-embedded/).

### 2.1 LED Module: The Simplest ops Table

```c
// Platform/Inc/platform_led.h

typedef struct {
    void (*on)(void* ctx);
    void (*off)(void* ctx);
    void (*toggle)(void* ctx);
    int16_t (*set_brightness)(void* ctx, uint8_t percent);
    int16_t (*set_rgb)(void* ctx, uint8_t r, uint8_t g, uint8_t b);
    int16_t (*get_state)(void* ctx, platform_led_state_t* state);
    int16_t (*get_brightness)(void* ctx, uint8_t* percent);
    int16_t (*get_rgb)(void* ctx, uint8_t* r, uint8_t* g, uint8_t* b);
} platform_led_ops_t;

typedef struct {
    const platform_led_ops_t* ops;  // Virtual pointer (vptr)
    const char* name;
    platform_led_type_t type;
    platform_led_state_t state;
    void* user_data;
} platform_led_base_t;
```

**Pattern Mapping**:

| C Construct | C++ Equivalent | oop-in-c-embedded Concept |
|-------------|---------------|-------------------------|
| `platform_led_ops_t` | Pure virtual class (abstract class) | ops operation table |
| `base.ops` | vptr | Virtual pointer |
| `ops->on(ctx)` | `virtual on()` | Virtual function call |
| `const ops` | Non-overridable vtable | Static vtable |

### 2.2 Six-Module ops Table Comparison

| Module | ops Function Count | Special Design |
|--------|-------------------|---------------|
| `platform_led_ops_t` | 8 | Brightness/RGB are optional (NULL means unsupported) |
| `platform_internal_flash_ops_t` | 10 | Includes protection status read/write |
| `platform_uart_ops_t` | 13 | Includes interrupt TX/RX + byte-level operations |
| `platform_fs_ops_t` + `platform_fs_dir_ops_t` | 11 + 3 | **Dual ops tables**: file and directory ops separated |
| `platform_mqtt_ops_t` | 12 | Pub/sub/will |
| `platform_tick_ops_t` | 2 | No ctx parameter (global singleton) |

### 2.3 Macro Wrapping: Safe Calls + Null Pointer Defense

```c
#define LED_ON(led) \
    do { \
        LED_ASSERT((led) && (led)->ops && (led)->ops->on); \
        if ((led)->ops->on) { \
            (led)->ops->on((led)); \
            (led)->state = LED_STATE_ON; \
        } \
    } while(0)
```

Each macro does three things:

1. **Assert check** (Debug mode)—`LED_ASSERT` becomes `((void)0)` in Release
2. **Null pointer defense**—`if (ops->on)` ensures optional functions are safely skipped
3. **State sync**—after calling, `base.state` is updated; callers don't need to maintain it manually

> [!TIP]
> Compare with oop-in-c-embedded's "pure virtual and optional strategy": setting PWM's `set_brightness` and RGB's `set_rgb` to `NULL` means the GPIO LED vtable has these pointers null—the macro's `if` check automatically skips them, no extra "unsupported" error code needed.

---

## 3. Impl Layer: The Derived Class Workshop

Each Impl module does three things: **define derived struct** → **fill static vtable** → **provide register function**.

### 3.1 GPIO LED: Simplest Derivation

```c
// Impl/Inc/platform_gpio_led_stm32_impl.h
typedef struct {
    platform_led_base_t base;   // Embedded base class (inheritance)
    GPIO_TypeDef *port;         // STM32-specific data
    uint16_t pin;
} gpio_led_stm32_t;
```

**Inheritance relationship**: `gpio_led_stm32_t` embeds `platform_led_base_t`—this is [struct nesting inheritance from oop-in-c-embedded](../oop-in-c-embedded/).

```c
// Impl/Src/platform_gpio_led_stm32_impl.c

static void gpio_led_on(void *ctx)
{
    gpio_led_stm32_t *self = container_of(ctx, gpio_led_stm32_t, base);
    HAL_GPIO_WritePin(self->port, self->pin, GPIO_PIN_SET);
}

static const platform_led_ops_t gpio_led_ops = {
    .on           = gpio_led_on,
    .off          = gpio_led_off,
    .toggle       = gpio_led_toggle,
    .set_brightness = NULL,     // GPIO doesn't support
    .set_rgb      = NULL,       // GPIO doesn't support
    .get_state    = gpio_led_get_state,
    .get_brightness = NULL,
    .get_rgb      = NULL,
};

void platform_gpio_led_stm32_register(gpio_led_stm32_t *led,
                                       GPIO_TypeDef *port, uint16_t pin,
                                       const char *name)
{
    led->port = port;
    led->pin = pin;
    LED_INIT_BASE(&led->base, &gpio_led_ops, name, LED_TYPE_GPIO);
}
```

**Three-Step Pattern**:

| Step | Code | OOP Concept |
|------|------|------------|
| 1. container_of downcast | `container_of(ctx, gpio_led_stm32_t, base)` | `dynamic_cast<Derived*>(base)` |
| 2. Static vtable | `static const platform_led_ops_t gpio_led_ops` | Vtable instantiation |
| 3. register constructor | `platform_gpio_led_stm32_register(...)` | Constructor + dependency injection |

### 3.2 Internal Flash: Multiple Inheritance with Dual Base Classes

This is the architecture's most elegant design. `internal_flash_stm32_t` embeds both `flash_base` and `transport_base`—one object plays two roles simultaneously.

```c
// Impl/Inc/platform_internal_flash_stm32_impl.h
typedef struct {
    platform_internal_flash_base_t flash_base;      // Base class 1: Flash operations
    platform_transport_base_t      transport_base;   // Base class 2: Transport target
    uint32_t written_size;
    uint32_t relocate_offset;      // A/B partition address relocation offset
    uint8_t  pending_buf[4];
    uint8_t  pending_len;
    uint8_t  is_open;
    uint8_t  is_erased;
} internal_flash_stm32_t;
```

**Two Vtables**:

| Vtable | Purpose | container_of Base |
|--------|---------|------------------|
| `flash_base.ops` (`internal_flash_ops`) | Direct Flash read/write/erase | `container_of(ctx, ..., flash_base)` |
| `transport_base.target_ops` (`internal_flash_target_ops`) | Transport protocol write (with relocation) | `container_of(ctx, ..., transport_base)` |

The same physical Flash presents different interfaces through **different base class pointers**—known in C++ as **interface separation via multiple inheritance**.

```c
// Different container_of bases, different "this" recovery paths
static int16_t internal_flash_read(void *ctx, ...) {
    internal_flash_stm32_t *self = container_of(ctx, internal_flash_stm32_t, flash_base);
    // ...
}

static int16_t internal_flash_tgt_write(const void *ctx, ...) {
    internal_flash_stm32_t *self = container_of(ctx, internal_flash_stm32_t, transport_base);
    // Contains address relocation logic
}
```

> [!IMPORTANT]
> The `relocate_offset` field is key to implementing A/B dual partitions—Slot B firmware is linked at Slot A addresses at compile time, and each 4-byte word needs an offset added at runtime. This field exists only in the Impl layer's derived struct; Platform's abstract interface is completely unaware of relocation—**implementation details are perfectly encapsulated**.

---

## 4. Service Layer: Dependency Injection Consumers

The Service layer doesn't call Impl directly; it works through Platform's abstract interfaces. Dependencies are injected at construction time and polymorphically called through base pointers at runtime.

### 4.1 YMODEM Service: Dual Dependency Injection

```c
// Service/Inc/service_ymodem.h
typedef struct {
    platform_uart_base_t      *uart;       // Dependency 1: UART
    platform_transport_base_t *transport;  // Dependency 2: Transport target
    uint32_t max_size;
    uint8_t *file_name;
    uint32_t file_name_len;
} ymodem_config_t;
```

YMODEM only knows "I need a UART and a transport target"—it doesn't care whether the UART is UART4 or USART1, or whether the target is internal Flash or SPI Flash. In `platform_config_init()`:

```c
// YMODEM actual injection (implicitly in main.c)
ymodem_config_t config = {
    .uart      = &g_uart4_console.base,     // Platform abstraction
    .transport = &g_slot_a_flash.transport_base,  // Platform abstraction
};
```

### 4.2 OneNet OTA Service: Triple Dependency Injection + Strategy Callback

```c
// Service/Inc/service_onenet_ota.h
typedef struct {
    platform_wifi_base_t *wifi;       // Dependency 1: WiFi
    platform_rtc_base_t  *rtc;       // Dependency 2: RTC
    platform_mqtt_base_t *mqtt;      // Dependency 3: MQTT
    onenet_ota_progress_cb_t progress_cb;  // Strategy callback
    uint8_t target_type;
    char firmware_version[32];
} onenet_ota_ctx_t;
```

| Dependency | Injected Instance | Replaceable With |
|------------|------------------|-----------------|
| `wifi` | `g_esp8266_wifi` | Any `platform_wifi_base_t` implementation |
| `rtc` | `g_rtc` | DS3231 external RTC |
| `mqtt` | `g_esp8266_mqtt` | Mosquitto embedded MQTT |

### 4.3 Menu Service: Command Handler Function Pointer Tree

```c
// Service/Inc/service_menu.h
typedef void (*menu_handler_t)(menu_ctx_t *ctx, int argc, char *argv[]);

typedef struct menu_item_s {
    const char *key;
    const char *name;
    const char *description;
    menu_item_type_t type;
    menu_handler_t handler;       // Command handler function pointer
    const menu_item_t *submenu;   // Submenu (tree structure)
    // ...
} menu_item_t;
```

The menu system uses function pointers + nested arrays to implement the **Command Pattern + Composite Pattern**—each menu item is either a leaf command (`handler`) or a submenu container (`submenu`).

---

## 5. Core Layer: Business Strategy and State Machine

The Bootloader_Core layer doesn't care about "what transport to use" or "what medium to write to"; it only defines download strategies and jump logic.

### 5.1 Dual Download Strategy

```c
// Bootloader_Core/bootloader_core.h

// Plain download: direct copy
bootloader_err_t bootloader_download(
    const platform_transport_base_t *src_transport,
    const platform_transport_base_t *tgt_transport,
    const char *path);

// Secure download: HMAC → HKDF → AES decrypt → Ed25519 verify → write
bootloader_err_t bootloader_secure_download(
    const platform_transport_base_t *src_transport,
    const platform_transport_base_t *tgt_transport,
    const char *path,
    const fw_pkg_verify_config_t *config,
    bootloader_secure_download_result_t *result);
```

The two function signatures are nearly identical, differing only in that `secure_download` has additional security verification config. Neither cares whether `src_transport` is SPI Flash or SD card—this is a classic application of the **Strategy Pattern**.

### 5.2 NoInit Cross-Reset State Transfer

```c
// Bootloader_Core/bootloader_core.c
volatile uint32_t update_flag __attribute__((section(".bss.NoInit"), used));
```

`update_flag` is placed in the `.bss.NoInit` section, which startup code does not zero. The App writes `UPDATE_FLAG_MAGIC` then triggers a soft reset; the Bootloader detects this and enters the upgrade menu—a **cross-reset state machine** without needing Flash persistence.

---

## 6. App Layer: Composition Root and Startup Orchestration

`main.c` is the architecture's **Composition Root**—all dependencies are assembled here.

### 6.1 Startup Flow

```
main()
  ├── Check NoInit update_flag → soft-reset jump
  ├── HAL_Init() + SystemClock_Config()
  ├── Peripheral init (MX_xxx_Init)
  ├── platform_config_init()        ← DI container initialization
  ├── ab_partition_init()           ← A/B partition state machine
  ├── A/B validity check and rollback
  ├── SD card detection and FS mount
  ├── Wait for manual command / check update request
  └── bootloader_request_jump()     ← Auto-jump to App
```

### 6.2 platform_config_init: Dependency Injection Container

```c
// Platform/Src/platform_config.c
void platform_config_init(void)
{
    platform_gpio_led_stm32_register(&g_status_led, LED_GPIO_Port, LED_Pin, "status_led");

    platform_w25q128_stm32_register(&g_w25q128_flash, &hspi1, GPIOA, GPIO_PIN_4, "w25q128");

    platform_internal_flash_stm32_register(&g_internal_flash,
                                            APPLICATION_ADDRESS,
                                            INTERNAL_FLASH_END_ADDR,
                                            "internal_flash");

    platform_internal_flash_stm32_register(&g_slot_a_flash,
                                            SLOT_A_START_ADDR, SLOT_A_END_ADDR, "slot_a");

    platform_internal_flash_stm32_register(&g_slot_b_flash,
                                            SLOT_B_START_ADDR, SLOT_B_END_ADDR, "slot_b");
    g_slot_b_flash.relocate_offset = SLOT_B_START_ADDR - SLOT_A_START_ADDR;

    platform_uart_stm32_register(&g_uart4_console, &huart4, "uart4_console");
    platform_uart_stm32_register(&g_usart1_esp8266, &huart1, "usart1_esp8266");

    platform_fatfs_stm32_register(&g_fatfs_transport, "fatfs");
    platform_lfs_stm32_register(&g_lfs_transport, "lfs");

    platform_rtc_stm32_register(&g_rtc, &hrtc, "rtc");

    platform_wifi_esp8266_register(&g_esp8266_wifi, &g_usart1_esp8266.base, "esp8266_wifi");
    platform_mqtt_esp8266_register(&g_esp8266_mqtt, &g_esp8266_wifi, "esp8266_mqtt");

    g_tick = platform_tick_stm32_get_instance();
}
```

This is the [oop-in-c-embedded "dependency injection container"](../oop-in-c-embedded/) realized in C:

- 13 global instances declared in `platform_config.h`
- Each instance initialized via its corresponding `register` function
- WiFi depends on UART, MQTT depends on WiFi—**dependency chains** are assembled in order within the container
- Slot B's `relocate_offset` is set separately after registration—**post-construction configuration**

---

## 7. Pattern Mapping: A Conversation with oop-in-c-embedded

Every design decision in this project can find a theoretical correspondence in [oop-in-c-embedded](../oop-in-c-embedded/):

| oop-in-c-embedded Concept | Project Instance | Code Location |
|--------------------------|-----------------|--------------|
| struct + me pointer | `void *ctx` in each ops function | Platform/Inc/*.h |
| ops operation table | `platform_led_ops_t` etc., 6 vtables | Platform/Inc/*.h |
| struct nesting inheritance | `gpio_led_stm32_t` embeds `base` | Impl/Inc/*_impl.h |
| container_of downcast | `container_of(ctx, gpio_led_stm32_t, base)` | Impl/Src/*_impl.c |
| Virtual pointer (vptr) | `base.ops` pointing to static vtable | Platform/Inc/*.h |
| Pure virtual & optional strategy | GPIO LED's `set_brightness = NULL` | Impl/Src/platform_gpio_led_stm32_impl.c |
| Multiple inheritance | `internal_flash_stm32_t` dual base classes | Impl/Inc/platform_internal_flash_stm32_impl.h |
| register constructor | `platform_gpio_led_stm32_register(...)` | Impl/Src/*_impl.c |
| Dependency injection container | `platform_config_init()` + 13 global instances | Platform/Src/platform_config.c |
| Polymorphic call macros | `LED_ON(led)` / `UART_TRANSMIT(uart, ...)` | Platform/Inc/*.h |
| Data three-level placement | HAL handles in Drivers, business data in Impl, abstract data in Platform | Across three layers |

---

## 8. container_of Deep Dive: The Architecture's Soul

`container_of` is the architecture's "downcast engine"—recovering a derived class instance from a base class pointer.

### 8.1 Macro Definition

```c
#ifndef container_of
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
#endif
```

### 8.2 Memory Layout Derivation

**gpio_led_stm32_t instance memory layout**:

| Field | Type | Description |
|--------|------|------|
| `base.ops` | `const platform_led_ops_t*` | Virtual pointer; `&instance.base` is the ctx passed to ops functions |
| `base.name` | `const char*` | LED name |
| `base.type` | `platform_led_type_t` | LED type enum |
| `base.state` | `platform_led_state_t` | Current state |
| `base.user_data` | `void*` | User data pointer |
| `port` | `GPIO_TypeDef*` | STM32 GPIO port (derived class specific) |
| `pin` | `uint16_t` | GPIO pin number (derived class specific) |

> `base` is the first member with offset 0, so `container_of(ctx, gpio_led_stm32_t, base)` is equivalent to `(gpio_led_stm32_t*)ctx`.

When `base` is the first member, `container_of` is equivalent to a cast. But with **multiple inheritance**, the offset is non-zero:

**internal_flash_stm32_t instance memory layout (dual base classes)**:

| Base Class | Offset | Fields | container_of Base |
|------------|--------|--------|------------------|
| `flash_base` | 0 | `ops`, `name`, `start_addr`... | `container_of(ctx, ..., flash_base)` offset 0 |
| `transport_base` | sizeof(flash_base) | `source_ops`, `target_ops` | `container_of(ctx, ..., transport_base)` offset > 0 |
| Derived specific | — | `written_size`, `relocate_offset`... | — |

> Two different `container_of` bases recover from **different base class pointers of the same instance** to **the same derived class instance**—this is exactly how `dynamic_cast` works with C++ multiple inheritance.

---

## 9. Chain Assembly of Dependency Injection

### 9.1 Dependency Graph

```
g_usart1_esp8266 (UART)
    ↓ .base
g_esp8266_wifi (WiFi)     ← depends on UART
    ↓
g_esp8266_mqtt (MQTT)     ← depends on WiFi

g_uart4_console (UART)    ← independent

g_internal_flash          ← independent
g_slot_a_flash            ← independent (offset = 0)
g_slot_b_flash            ← independent (offset ≠ 0, A/B relocation)
g_download_cache_flash    ← independent

g_w25q128_flash (SPI Flash) ← depends on SPI1
g_fatfs_transport          ← logical filesystem
g_lfs_transport            ← logical filesystem
g_rtc                      ← independent
g_status_led               ← independent
g_tick                     ← singleton
```

### 9.2 Why Global Instances Instead of Dynamic Allocation?

Embedded Bootloader constraints:

1. **No heap**—`malloc` is forbidden in a bare-metal Bootloader; heap fragmentation could cause update failure and brick the device
2. **Compile-time determined**—the count and size of 13 instances are known at compile time; BSS segment zeroing suffices
3. **Link-time optimization**—`--gc-sections` can trim unused modules; global instances don't affect final size

---

## 10. Dual ops Tables and Interface Segregation

`platform_filesystem.h` employs a **dual ops table** design:

```c
typedef struct {
    int16_t (*open)(...);
    int16_t (*close)(...);
    int32_t (*read)(...);
    int32_t (*write)(...);
    // ... 11 file operations
} platform_fs_ops_t;

typedef struct {
    int16_t (*open)(...);
    int16_t (*close)(...);
    int16_t (*read)(...);
} platform_fs_dir_ops_t;    // 3 directory operations

struct platform_fs_base_s {
    const platform_fs_ops_t     *ops;       // File ops
    const platform_fs_dir_ops_t *dir_ops;   // Directory ops
    const char *name;
    void *user_data;
};
```

Why not two separate classes? Because filesystem and directory are **two views of the same object**—FatFs and LittleFS share the same underlying state for directory traversal and file operations. Dual ops tables = C's version of the **Interface Segregation Principle (ISP)**.

---

## 11. From Theory to Practice: A Summary Table

| Design Dimension | oop-in-c-embedded Theory | Bootloader Practice |
|-----------------|------------------------|-------------------|
| Encapsulation | `struct + me` | `base + ops + ctx` |
| Inheritance | struct nesting | `gpio_led_stm32_t` embeds `base` |
| Polymorphism | ops vtable + vptr | 6 ops tables + macro dispatch |
| Downcast | `container_of` | Dual base class dual offset recovery |
| Multiple inheritance | Dual base class nesting | `flash_base + transport_base` |
| Construction | register function | 13 register calls |
| Dependency injection | platform_config container | `platform_config_init()` |
| Safe calls | NULL check + macro | `LED_ON` / `UART_TRANSMIT` |
| Interface segregation | Multiple ops tables | `fs_ops + dir_ops` |
| Optional strategy | NULL function pointers | GPIO LED skips brightness/RGB |

---

## 12. Thoughts After Tasting

### 12.1 The Cost of This Architecture

- **Code volume**: Each Platform module needs a `.h` (ops + base + macros) + an Impl `.h` + an Impl `.c` + register function—about 200-400 lines/module
- **Indirect call overhead**: Each virtual call adds one pointer dereference—approximately 1-2 extra clock cycles on Cortex-M4
- **Debug transparency**: GDB cannot directly show which function `ops->on` points to; manual dereferencing is needed

### 12.2 The Benefits of This Architecture

- **Zero-coupling replacement**: Changing UART from UART4 to USART2 requires modifying only one line in `platform_config_init()`
- **Unit test friendly**: Mock a `platform_uart_base_t` with test ops, and the Service layer can be tested completely without hardware
- **Cross-platform porting**: Adding ESP32 support only requires adding ESP32 implementations under `Impl/`; Platform and Service layers need zero changes
- **Compile-time trimming**: Don't need WiFi/MQTT? Remove the `platform_wifi_esp8266_register` and `platform_mqtt_esp8266_register` lines; the linker automatically trims related code

### 12.3 Applicability Boundaries

| Scenario | Applicable? |
|----------|------------|
| Extremely resource-constrained MCU (< 32KB Flash) | No—vtable and macro code bloat may exceed budget |
| Multi-platform/multi-hardware-variant product line | Highly applicable—one Service + N Impls |
| Firmware requiring unit tests | Highly applicable—Mock injection enables hardware-free testing |
| One-off prototype/Demo | Over-engineering—calling HAL directly is faster |

---

## 13. Further Reading

- [The Essence of C: Structs and Pointers](../oop-in-c-embedded/) — The theoretical foundation of this post
- [STM32F407 Secure Bootloader Design](../stm32f407-secure-bootloader/) — The code source for this post
- [GNU LD Linker Script in Depth](../gnu-ld-linker-script/) — NoInit section and linker script cooperation
- [PyIAPToolKit](../pyiap-toolkit/) — PC-side companion tool suite
