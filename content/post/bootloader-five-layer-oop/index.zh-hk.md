---
title: "淺嘗 Bootloader 五層架構：從 Platform 抽象到 App 編排的 OOP 物件設計模型"
date: 2026-07-03
description: "以 STM32F407 安全 Bootloader 為實例，淺嘗 Impl/Platform/Service/Core/App + Drivers(Vendor) 五層架構的 C 物件導向設計模型：ops 虛函式表、container_of 向下轉型、多重繼承雙基類、依賴注入容器、宏封裝安全呼叫，與 oop-in-c-embedded 概念逐一映射"
categories:
  - "嵌入式"
tags:
  - "STM32"
  - "Bootloader"
  - "OOP-in-C"
  - "分層架構"
  - "物件導向"
  - "平台抽象"
math: true
---

## 前言

在上一篇 [C 語言的精髓：結構體與指標](../oop-in-c-embedded/) 中，我們從 `struct + me` 出發，一路走到了 Linux 核心風格的 `gpio_chip` + initcall 自動註冊。理論有了，實戰呢？

本文以 [STM32F407 安全 Bootloader](../stm32f407-secure-bootloader/) 專案的真實程式碼為實例，**淺嘗**其五層架構物件設計模型——看 OOP in C 的精髓如何在一個工業級 Bootloader 中落地生根。

> [!NOTE]
> 「淺嘗」不是淺嘗輒止——而是以品味為核心，聚焦每一層的設計意圖和模式映射，而非逐行解讀。

---

## 一、五層架構總覽

### 1.1 分層圖

| 層 | 目錄 | 職責 | OOP 概念映射 |
|----|------|------|--------------|
| **App** | `Core/Src/main.c` | 系統編排、啟動流程、條件分支 | 頂層組合根 / main 編排 |
| **Core** | `Bootloader_Core/` | 韌體下載引擎、A/B 分區、跳轉邏輯 | 業務策略 / 狀態機 |
| **Service** | `Service/` | YMODEM、OTA、選單、差分、解密、簽章 | 服務物件 / 依賴注入消費者 |
| **Platform** | `Platform/Inc/` | 抽象介面定義（ops 虛表 + base 結構體 + 宏） | 純虛基類 / 介面 |
| **Impl** | `Impl/` | STM32 / ESP8266 具體實作 | 衍生類 / 虛表實例化 |
| **Drivers** | `Drivers/` | HAL + Vendor 驅動（STM32 HAL、W25Q128、FatFs…） | 硬體抽象層 |

呼叫方向：**App → Core → Service → Platform ← Impl → Drivers**

關鍵設計：**Service 不依賴 Impl，只依賴 Platform 的抽象介面**。Impl 透過 register 函式在啟動時將具體實作「注入」Platform 的 base 實例——這就是 [依賴注入](../oop-in-c-embedded/) 在 C 中的落地。

### 1.2 為什麼是五層而非三層？

經典三層架構（App → HAL → Hardware）的問題：業務邏輯和硬體實作耦合。增加 Service 層隔離業務協定，增加 Platform 層實作多型抽象，增加 Impl 層承載具體實作——每一層只依賴下一層的**抽象**，不依賴**實作**。

---

## 二、Platform 層：純虛介面的定義者

Platform 層是整個架構的「介面契約」，只定義 ops 虛函式表和 base 結構體，不包含任何實作程式碼。對照 [oop-in-c-embedded 的 ops 操作表](../oop-in-c-embedded/) 概念。

### 2.1 LED 模組：最簡 ops 表

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
    const platform_led_ops_t* ops;  // 虛指標 (vptr)
    const char* name;
    platform_led_type_t type;
    platform_led_state_t state;
    void* user_data;
} platform_led_base_t;
```

**模式映射**：

| C 寫法 | C++ 等價 | oop-in-c-embedded 概念 |
|--------|----------|----------------------|
| `platform_led_ops_t` | 純虛類（abstract class） | ops 操作表 |
| `base.ops` | vptr | 虛指標 |
| `ops->on(ctx)` | `virtual on()` | 虛函式呼叫 |
| `const ops` | 不可覆寫的虛表 | 靜態虛表 |

### 2.2 六模組 ops 表對比

| 模組 | ops 函式數 | 特殊設計 |
|------|-----------|---------|
| `platform_led_ops_t` | 8 | 亮度/RGB 為選填（NULL 表示不支援） |
| `platform_internal_flash_ops_t` | 10 | 含保護狀態讀寫 |
| `platform_uart_ops_t` | 13 | 含中斷收發 + 位元組級操作 |
| `platform_fs_ops_t` + `platform_fs_dir_ops_t` | 11 + 3 | **雙 ops 表**：檔案操作與目錄操作分離 |
| `platform_mqtt_ops_t` | 12 | 發布/訂閱/遺囑 |
| `platform_tick_ops_t` | 2 | 無 ctx 參數（全域單例） |

### 2.3 宏封裝：安全呼叫 + 空指標防禦

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

每個宏做三件事：

1. **斷言檢查**（Debug 模式）——`LED_ASSERT` 在 Release 下為 `((void)0)`
2. **空指標防禦**——`if (ops->on)` 確保選填函式安全跳過
3. **狀態同步**——呼叫後更新 `base.state`，呼叫者無需手動維護

> [!TIP]
> 對照 oop-in-c-embedded 的「純虛函式與選填策略」：將 PWM 的 `set_brightness`、RGB 的 `set_rgb` 設為 `NULL`，GPIO LED 的虛表中這幾個指標為空——宏的 `if` 判斷自動跳過，無需額外的「不支援」錯誤碼。

---

## 三、Impl 層：衡生類的實作車間

Impl 層的每個模組做三件事：**定義衡生結構體** → **填充靜態虛表** → **提供 register 函式**。

### 3.1 GPIO LED：最簡衡生

```c
// Impl/Inc/platform_gpio_led_stm32_impl.h
typedef struct {
    platform_led_base_t base;   // 內嵌基類（繼承）
    GPIO_TypeDef *port;         // STM32 特有資料
    uint16_t pin;
} gpio_led_stm32_t;
```

**繼承關係**：`gpio_led_stm32_t` 內嵌 `platform_led_base_t`——這就是 [oop-in-c-embedded 的 struct 巢套繼承](../oop-in-c-embedded/)。

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
    .set_brightness = NULL,     // GPIO 不支援
    .set_rgb      = NULL,       // GPIO 不支援
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

**三步模式**：

| 步驟 | 程式碼 | OOP 概念 |
|------|--------|---------|
| 1. container_of 向下轉型 | `container_of(ctx, gpio_led_stm32_t, base)` | `dynamic_cast<Derived*>(base)` |
| 2. 靜態虛表 | `static const platform_led_ops_t gpio_led_ops` | 虛表（vtable）實例化 |
| 3. register 建構函式 | `platform_gpio_led_stm32_register(...)` | 建構函式 + 依賴注入 |

### 3.2 Internal Flash：多重繼承雙基類

這是本架構最精妙的設計。`internal_flash_stm32_t` 同時內嵌 `flash_base` 和 `transport_base`，一個物件同時扮演兩個角色。

```c
// Impl/Inc/platform_internal_flash_stm32_impl.h
typedef struct {
    platform_internal_flash_base_t flash_base;      // 基類1：Flash 操作
    platform_transport_base_t      transport_base;   // 基類2：傳輸目標
    uint32_t written_size;
    uint32_t relocate_offset;      // A/B 分區位址重定位偏移
    uint8_t  pending_buf[4];
    uint8_t  pending_len;
    uint8_t  is_open;
    uint8_t  is_erased;
} internal_flash_stm32_t;
```

**兩個虛表**：

| 虛表 | 用途 | container_of 基準 |
|------|------|------------------|
| `flash_base.ops`（`internal_flash_ops`） | Flash 直接讀寫/抹除 | `container_of(ctx, ..., flash_base)` |
| `transport_base.target_ops`（`internal_flash_target_ops`） | 傳輸協定寫入（含重定位） | `container_of(ctx, ..., transport_base)` |

同一個物理 Flash，透過**不同基類指標**呈現不同介面——C++ 中稱為**多重繼承的介面分離**。

```c
// 不同的 container_of 基準，不同的 "this" 恢復路徑
static int16_t internal_flash_read(void *ctx, ...) {
    internal_flash_stm32_t *self = container_of(ctx, internal_flash_stm32_t, flash_base);
    // ...
}

static int16_t internal_flash_tgt_write(const void *ctx, ...) {
    internal_flash_stm32_t *self = container_of(ctx, internal_flash_stm32_t, transport_base);
    // 含位址重定位邏輯
}
```

> [!IMPORTANT]
> `relocate_offset` 欄位是實作 A/B 雙分區的關鍵——Slot B 的韌體在編譯時按 Slot A 位址鏈結，執行時每個 4 位元組 word 需加上偏移量重定位。這個欄位只存在於 Impl 層的衡生結構體中，Platform 層的抽象介面完全不知道重定位的存在——**實作細節被完美封裝**。

---

## 四、Service 層：依賴注入的消費者

Service 層不直接呼叫 Impl，而是透過 Platform 抽象介面工作。依賴在建構時注入，執行時透過 base 指標多型呼叫。

### 4.1 YMODEM 服務：雙依賴注入

```c
// Service/Inc/service_ymodem.h
typedef struct {
    platform_uart_base_t      *uart;       // 依賴1：串列埠
    platform_transport_base_t *transport;  // 依賴2：傳輸目標
    uint32_t max_size;
    uint8_t *file_name;
    uint32_t file_name_len;
} ymodem_config_t;
```

YMODEM 只知道「我需要一個串列埠和一個傳輸目標」，不關心串列埠是 UART4 還是 USART1，不關心目標是內部 Flash 還是 SPI Flash。在 `platform_config_init()` 中：

```c
// YMODEM 實際注入方式（main.c 中隱式）
ymodem_config_t config = {
    .uart      = &g_uart4_console.base,     // Platform 抽象
    .transport = &g_slot_a_flash.transport_base,  // Platform 抽象
};
```

### 4.2 OneNet OTA 服務：三依賴注入 + 策略回呼

```c
// Service/Inc/service_onenet_ota.h
typedef struct {
    platform_wifi_base_t *wifi;       // 依賴1：WiFi
    platform_rtc_base_t  *rtc;       // 依賴2：RTC
    platform_mqtt_base_t *mqtt;      // 依賴3：MQTT
    onenet_ota_progress_cb_t progress_cb;  // 策略回呼
    uint8_t target_type;
    char firmware_version[32];
} onenet_ota_ctx_t;
```

| 依賴 | 注入實例 | 可替換為 |
|------|---------|---------|
| `wifi` | `g_esp8266_wifi` | 任何 `platform_wifi_base_t` 實作 |
| `rtc` | `g_rtc` | DS3231 外部 RTC |
| `mqtt` | `g_esp8266_mqtt` | Mosquitto 嵌入式 MQTT |

### 4.3 選單服務：命令處理函式指標樹

```c
// Service/Inc/service_menu.h
typedef void (*menu_handler_t)(menu_ctx_t *ctx, int argc, char *argv[]);

typedef struct menu_item_s {
    const char *key;
    const char *name;
    const char *description;
    menu_item_type_t type;
    menu_handler_t handler;       // 命令處理函式指標
    const menu_item_t *submenu;   // 子選單（樹形結構）
    // ...
} menu_item_t;
```

選單系統用函式指標 + 巢套陣列實作**命令模式 + 組合模式**——每個選單項要麼是葉子命令（`handler`），要麼是子選單容器（`submenu`）。

---

## 五、Core 層：業務策略與狀態機

Bootloader_Core 層不關心「用什麼傳輸」和「寫到什麼介質」，只定義下載策略和跳轉邏輯。

### 5.1 雙下載策略

```c
// Bootloader_Core/bootloader_core.h

// 普通下載：直接搬運
bootloader_err_t bootloader_download(
    const platform_transport_base_t *src_transport,
    const platform_transport_base_t *tgt_transport,
    const char *path);

// 安全下載：HMAC → HKDF → AES 解密 → Ed25519 驗簽 → 寫入
bootloader_err_t bootloader_secure_download(
    const platform_transport_base_t *src_transport,
    const platform_transport_base_t *tgt_transport,
    const char *path,
    const fw_pkg_verify_config_t *config,
    bootloader_secure_download_result_t *result);
```

兩個函式簽章幾乎相同，區別在於 `secure_download` 多了安全驗證設定。它們都不關心 `src_transport` 是 SPI Flash 還是 SD 卡——這是 **策略模式** 的典型應用。

### 5.2 NoInit 跨重啟狀態傳遞

```c
// Bootloader_Core/bootloader_core.c
volatile uint32_t update_flag __attribute__((section(".bss.NoInit"), used));
```

`update_flag` 放在 `.bss.NoInit` 段，啟動程式碼不會清零它。App 寫入 `UPDATE_FLAG_MAGIC` 後觸發軟復位，Bootloader 偵測到後進入升級選單——**跨復位的狀態機**，無需持久化到 Flash。

---

## 六、App 層：組合根與啟動編排

`main.c` 是整個架構的**組合根（Composition Root）**——所有依賴在這裡組裝。

### 6.1 啟動流程

```
main()
  ├── 檢查 NoInit update_flag → 軟復位跳轉
  ├── HAL_Init() + SystemClock_Config()
  ├── 外設初始化 (MX_xxx_Init)
  ├── platform_config_init()        ← 依賴注入容器初始化
  ├── ab_partition_init()           ← A/B 分區狀態機
  ├── A/B 有效性校驗與回滾
  ├── SD 卡偵測與檔案系統掛載
  ├── 等待手動命令 / 檢查升級請求
  └── bootloader_request_jump()     ← 自動跳轉 App
```

### 6.2 platform_config_init：依賴注入容器

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

這就是 [oop-in-c-embedded 的「依賴注入容器」](../oop-in-c-embedded/) 在 C 中的真實形態：

- 13 個全域實例宣告在 `platform_config.h`
- 每個實例透過對應的 `register` 函式初始化
- WiFi 依賴 UART，MQTT 依賴 WiFi——**依賴鏈**在容器中按序組裝
- Slot B 的 `relocate_offset` 在註冊後單獨設定——**建構後設定**

---

## 七、模式映射：與 oop-in-c-embedded 的對話

本專案的每個設計決策，都能在 [oop-in-c-embedded](../oop-in-c-embedded/) 中找到理論對應：

| oop-in-c-embedded 概念 | 本專案實例 | 程式碼位置 |
|------------------------|-----------|---------|
| struct + me 指標 | 每個 ops 函式的 `void *ctx` | Platform/Inc/*.h |
| ops 操作表 | `platform_led_ops_t` 等 6 個虛表 | Platform/Inc/*.h |
| struct 巢套繼承 | `gpio_led_stm32_t` 內嵌 `base` | Impl/Inc/*_impl.h |
| container_of 向下轉型 | `container_of(ctx, gpio_led_stm32_t, base)` | Impl/Src/*_impl.c |
| 虛指標 (vptr) | `base.ops` 指向靜態虛表 | Platform/Inc/*.h |
| 純虛與選填策略 | GPIO LED 的 `set_brightness = NULL` | Impl/Src/platform_gpio_led_stm32_impl.c |
| 多重繼承 | `internal_flash_stm32_t` 雙基類 | Impl/Inc/platform_internal_flash_stm32_impl.h |
| register 建構函式 | `platform_gpio_led_stm32_register(...)` | Impl/Src/*_impl.c |
| 依賴注入容器 | `platform_config_init()` + 13 個全域實例 | Platform/Src/platform_config.c |
| 多型呼叫宏 | `LED_ON(led)` / `UART_TRANSMIT(uart, ...)` | Platform/Inc/*.h |
| 資料三級歸位 | HAL 柄在 Drivers，業務資料在 Impl，抽象資料在 Platform | 跨三層分佈 |

---

## 八、container_of 深度剖析：本架構的靈魂

`container_of` 是整個架構的「向下轉型引擎」——從基類指標恢復衡生類實例。

### 8.1 宏定義

```c
#ifndef container_of
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
#endif
```

### 8.2 記憶體佈局推導

```
gpio_led_stm32_t 實例在記憶體中：
┌──────────────────────────┐ ← &instance (gpio_led_stm32_t*)
│  base (platform_led_base_t) │
│    ├── ops (const ptr)     │ ← &instance.base (platform_led_base_t*)
│    ├── name                │     傳入 ops 函式的 ctx 就是這個位址
│    ├── type                │
│    ├── state               │
│    └── user_data           │
│  port (GPIO_TypeDef*)      │
│  pin  (uint16_t)           │
└──────────────────────────┘

container_of(ctx, gpio_led_stm32_t, base) 的計算：
  = (gpio_led_stm32_t*)((char*)ctx - offsetof(gpio_led_stm32_t, base))
  = (gpio_led_stm32_t*)((char*)ctx - 0)    // base 是第一個成員，偏移為 0
  = (gpio_led_stm32_t*)ctx
```

當 `base` 是第一個成員時，`container_of` 等價於強轉。但 **多重繼承** 時偏移不為零：

```
internal_flash_stm32_t 實例：
┌────────────────────────────┐ ← &instance
│  flash_base (offset = 0)    │ ← container_of(ctx, ..., flash_base) 偏移 0
│    ├── ops                  │
│    ├── name, start_addr...  │
│  transport_base (offset≠0)  │ ← container_of(ctx, ..., transport_base) 偏移 > 0
│    ├── source_ops           │
│    ├── target_ops           │
│  written_size, relocate...  │
└────────────────────────────┘
```

兩個不同的 `container_of` 基準，從**同一個實例的不同基類指標**，恢復到**同一個衡生類實例**——這正是 C++ 多重繼承中 `dynamic_cast` 的運作原理。

---

## 九、依賴注入的鏈式組裝

### 9.1 依賴圖

```
g_usart1_esp8266 (UART)
    ↓ .base
g_esp8266_wifi (WiFi)     ← 依賴 UART
    ↓
g_esp8266_mqtt (MQTT)     ← 依賴 WiFi

g_uart4_console (UART)    ← 獨立

g_internal_flash          ← 獨立
g_slot_a_flash            ← 獨立（偏移 = 0）
g_slot_b_flash            ← 獨立（偏移 ≠ 0，A/B 重定位）
g_download_cache_flash    ← 獨立

g_w25q128_flash (SPI Flash) ← 依賴 SPI1
g_fatfs_transport          ← 邏輯檔案系統
g_lfs_transport            ← 邏輯檔案系統
g_rtc                      ← 獨立
g_status_led               ← 獨立
g_tick                     ← 單例
```

### 9.2 為什麼用全域實例而非動態分配？

嵌入式 Bootloader 的約束：

1. **無堆積**——`malloc` 在裸機 Bootloader 中是禁忌，堆積碎片化可能導致升級失敗變磚
2. **編譯期確定**——13 個實例的數量和大小在編譯時已知，`BSS` 段清零即可
3. **鏈結時最佳化**——`--gc-sections` 可以裁剪未使用的模組，全域實例不影響最終體積

---

## 十、雙 ops 表與介面分離

`platform_filesystem.h` 採用了**雙 ops 表**設計：

```c
typedef struct {
    int16_t (*open)(...);
    int16_t (*close)(...);
    int32_t (*read)(...);
    int32_t (*write)(...);
    // ... 11 個檔案操作
} platform_fs_ops_t;

typedef struct {
    int16_t (*open)(...);
    int16_t (*close)(...);
    int16_t (*read)(...);
} platform_fs_dir_ops_t;    // 3 個目錄操作

struct platform_fs_base_s {
    const platform_fs_ops_t     *ops;       // 檔案 ops
    const platform_fs_dir_ops_t *dir_ops;   // 目錄 ops
    const char *name;
    void *user_data;
};
```

為什麼不分兩個類？因為檔案系統和目錄是**同一個物件的兩個視圖**——FatFs 和 LittleFS 的目錄遍歷與檔案操作共用同一個底層狀態。雙 ops 表 = C 版的**介面分離原則（ISP）**。

---

## 十一、從理論到實戰：一張圖總結

| 設計維度 | oop-in-c-embedded 理論 | Bootloader 實戰 |
|---------|----------------------|----------------|
| 封裝 | `struct + me` | `base + ops + ctx` |
| 繼承 | struct 巢套 | `gpio_led_stm32_t` 內嵌 `base` |
| 多型 | ops 虛表 + vptr | 6 個 ops 表 + 宏分發 |
| 向下轉型 | `container_of` | 雙基類雙偏移恢復 |
| 多重繼承 | 雙基類巢套 | `flash_base + transport_base` |
| 建構 | register 函式 | 13 個 register 呼叫 |
| 依賴注入 | platform_config 容器 | `platform_config_init()` |
| 安全呼叫 | NULL 檢查 + 宏 | `LED_ON` / `UART_TRANSMIT` |
| 介面分離 | 多 ops 表 | `fs_ops + dir_ops` |
| 選填策略 | NULL 函式指標 | GPIO LED 跳過 brightness/RGB |

---

## 十二、淺嘗之後的思考

### 12.1 這套架構的代價

- **程式碼量**：每個 Platform 模組需要一個 `.h`（ops + base + 宏）+ 一個 Impl `.h` + 一個 Impl `.c` + register 函式——約 200-400 行/模組
- **間接呼叫開銷**：每次虛函式呼叫多一次指標解參考，在 Cortex-M4 上約 1-2 個額外時脈週期
- **除錯透明度**：GDB 無法直接顯示 `ops->on` 指向哪個函式，需手動解參考

### 12.2 這套架構的收益

- **零耦合替換**：將 UART 從 UART4 換到 USART2，只改 `platform_config_init()` 一行
- **單元測試友好**：Mock 一個 `platform_uart_base_t` 填入測試 ops，Service 層可完全脫離硬體測試
- **跨平台移植**：新增 ESP32 平台只需新增 `Impl/` 目錄下的 ESP32 實作，Platform 和 Service 層零改動
- **編譯裁剪**：不需要 WiFi/MQTT？刪掉 `platform_wifi_esp8266_register` 和 `platform_mqtt_esp8266_register` 兩行，鏈結器自動裁剪相關程式碼

### 12.3 適用邊界

| 場景 | 是否適用 |
|------|---------|
| MCU 資源極度緊張（< 32KB Flash） | 不適用——虛表和宏的程式碼膨脹可能超出預算 |
| 多平台/多硬體變體產品線 | 高度適用——一份 Service + N 份 Impl |
| 需要單元測試的韌體 | 高度適用——Mock 注入即可脫硬體測試 |
| 一次性原型/Demo | 過度設計——直接調 HAL 更快 |

---

## 十三、延伸閱讀

- [C 語言的精髓：結構體與指標](../oop-in-c-embedded/) —— 本文的理論基礎
- [STM32F407 安全 Bootloader 設計](../stm32f407-secure-bootloader/) —— 本文的程式碼來源
- [GNU LD 鏈結腳本詳解](../gnu-ld-linker-script/) —— NoInit 段與鏈結腳本的協作
- [PyIAPToolKit](../pyiap-toolkit/) —— PC 端配套工具套件
