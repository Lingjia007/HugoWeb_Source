---
title: "STM32F407 安全 Bootloader 设计：A/B 双分区、加密签名与多渠道升级"
date: 2026-06-20
description: "基于 STM32F407VGT6 的工业级安全 Bootloader 完整设计，涵盖 A/B 双分区无缝升级、AES-256 加密 + Ed25519 签名安全固件包、地址重定位、差分升级、多渠道下载（UART/SD卡/SPI Flash/WiFi OTA）等核心技术详解"
image: STM32F407安全Bootloader设计.png
categories:
  - "嵌入式"
tags:
  - "STM32"
  - "Bootloader"
  - "IAP"
  - "A/B分区"
  - "固件加密"
  - "Ed25519"
  - "OTA"
math: true
---

## 前言

在物联网和嵌入式设备领域，固件升级（OTA）是产品生命周期中不可或缺的一环。然而，升级过程中的**断电崩溃、固件篡改、版本回滚**等问题，往往让开发者头疼不已。本文将详细介绍一个基于 STM32F407VGT6 的安全 Bootloader 项目，它通过 **A/B 双分区**、**安全固件包**、**地址重定位** 等机制，系统性地解决了这些痛点。

---

## 一、解决了什么问题？

| 问题                       | 传统方案                 | 本项目方案                                    |
| -------------------------- | ------------------------ | --------------------------------------------- |
| 升级中断电导致设备变砖     | 单分区直接覆盖，断电即死 | A/B 双分区，升级失败自动回滚                  |
| 固件被篡改或伪造           | 无校验或仅 CRC           | Ed25519 签名 + HMAC-SHA256 认证               |
| 固件被窃取逆向             | 明文传输                 | AES-256-CBC/CTR 加密 + 设备绑定密钥           |
| 旧版本固件降级攻击         | 无防护                   | 安全计数器防回滚                              |
| 为两个分区维护两套编译配置 | 维护两份工程             | 地址自动重定位 + 双 Target 编译，源码只需一份 |
| 升级包过大占用带宽         | 全量升级                 | HPatch + tuz 差分升级                         |

---

## 二、Flash 内存布局

STM32F407VGT6 拥有 1MB 内部 Flash，分区如下：

| 起始地址      | 区域           | 大小   | Flash 扇区       |
| ------------- | -------------- | ------ | ---------------- |
| `0x0800_0000` | Bootloader     | 128 KB | Sector 0-4       |
| `0x0802_0000` | Slot A         | 384 KB | Sector 5-7       |
| `0x0808_0000` | Slot B         | 384 KB | Sector 8-10      |
| `0x080E_0000` | Download Cache | 64 KB  | Sector 11 (部分) |
| `0x080F_0000` | Metadata       | 64 KB  | Sector 11 (部分) |

> [!TIP]
> 为什么 Bootloader 占用 128KB？因为本项目目前为测试 Demo，未做精简优化——实际上一个精简的 Bootloader 通常只需 32-48KB。当前 128KB 中包含了完整的菜单系统、加密库（AES/Ed25519/HKDF）、HPatch 差分、ESP8266 WiFi/MQTT 等所有功能模块。在量产化时，可通过裁剪不需要的功能，将 Bootloader 缩小至合理大小，从而为 App 释放更多 Flash 空间。

### 链接脚本（Scatter File）

```
LR_IROM1 0x08000000 0x00020000  {    ; Bootloader 128KB
  ER_IROM1 0x08000000 0x00020000  {
   *.o (RESET, +First)
   *(InRoot$$Sections)
   .ANY (+RO)
   .ANY (+XO)
  }
  RW_IRAM1 0x20000000 0x0001BFFC  {  ; 主 RAM
   .ANY (+RW +ZI)
  }
  RW_IRAM_NOINIT 0x2001BFFC UNINIT 0x00000004  {
   *(.bss.NoInit)                    ; Bootloader/App 通信标志
  }
}
```

**关键设计**：`NoInit` 区域位于 `0x2001BFFC`，存放 `update_flag` 变量。软复位后该变量不被清零，用于 Bootloader 和 App 之间的状态传递。

> [!NOTE]
> `update_flag` 的魔术值定义：
>
> - `0x5A5A5A5A` (`UPDATE_FLAG_MAGIC`) — App 请求固件更新
> - `0xA5A5A5A5` (`JUMP_FLAG_MAGIC`) — 请求跳转到 App（软复位后快速跳转）
> - `0x424F4F54` (`BOOT_CONFIRM_MAGIC`) — 启动确认标记

---

## 三、启动流程与跳转机制

### 3.1 完整启动流程

1. **上电/复位**
2. 检查 `update_flag == JUMP_FLAG_MAGIC`？→ 是：清除 flag → 直接跳转 App（快速启动）
3. `HAL_Init()` → `SystemClock_Config()` → 外设初始化
4. A/B 分区状态检查：
   - **TESTING** 状态 → 验证固件有效性
     - 无效 → 回滚到另一分区
     - 启动尝试超过3次 → 回滚
     - 有效 → 增加启动尝试计数
   - **CONFIRMED** 状态 → 正常启动
5. 等待 UART 命令（1.5秒超时）
   - 收到 'M' → 进入 Bootloader 菜单
   - 超时 → 继续
6. 检查 `update_flag == UPDATE_FLAG_MAGIC`？→ 是：进入 Bootloader 菜单（App 请求更新）
7. 自动跳转到活动分区的 App

### 3.2 跳转核心代码

跳转分两步实现，通过软复位确保硬件状态干净：

**第一步：请求跳转（设置标志 + 软复位）**

```c
void bootloader_request_jump(ab_slot_t slot) {
    uint32_t app_addr = bootloader_get_slot_addr(slot);
    if (app_addr == 0 || !bootloader_validate_sp(app_addr)) return;

    ab_partition_set_active_slot(slot);  // 设置活动分区
    update_flag = JUMP_FLAG_MAGIC;       // 设置跳转标志
    __DSB(); __ISB();
    NVIC_SystemReset();                  // 软复位
}
```

**第二步：实际执行跳转（复位后执行）**

```c
void bootloader_execute_jump(void) {
    uint32_t app_addr = bootloader_get_slot_addr(AB_SLOT_AUTO);
    uint32_t app_stack_ptr = *(__IO uint32_t *)app_addr;
    uint32_t app_reset_handler = *(__IO uint32_t *)(app_addr + 4);

    SysTick->CTRL = 0;           // 关闭 SysTick
    __set_PRIMASK(0);            // 使能中断
    SCB->VTOR = app_addr;        // 重定向中断向量表
    __DSB(); __ISB();
    __set_MSP(app_stack_ptr);    // 设置主栈指针
    ((pFunction)app_reset_handler)();  // 跳转到 App ResetHandler
}
```

**SP 验证**确保栈指针在合法 SRAM 范围内：

```c
bool bootloader_validate_sp(uint32_t app_addr) {
    uint32_t sp = *(__IO uint32_t *)app_addr;
    return (sp >= 0x20000000 && sp <= 0x2FFE0000);
}
```

> [!TIP]
> 为什么跳转分两步？直接跳转虽然更快，但软复位能确保所有外设和中断状态被干净地重置。本项目在正常启动流程中使用 `request_jump`（软复位），而在 `update_flag == JUMP_FLAG_MAGIC` 时直接调用 `execute_jump` 实现快速启动——因为此时是刚从 App 软复位回来，硬件状态已经是干净的。

---

## 四、A/B 双分区系统

### 4.1 元数据结构

元数据存储在 Flash 末尾的 Metadata 区域，采用**追加写入**策略避免频繁擦除：

```c
typedef struct __attribute__((packed)) {
    uint32_t fw_version;         // 固件版本
    uint32_t security_counter;   // 安全计数器（防回滚）
    uint32_t fw_size;            // 固件大小
    ab_slot_state_t state;       // 分区状态
    uint8_t boot_attempts;       // 启动尝试次数
    uint8_t sha256[32];          // 固件 SHA256
    uint8_t reserved[3];
} ab_slot_meta_t;

typedef struct __attribute__((packed)) {
    uint32_t magic;              // 0x41425441 ("ABTA")
    uint32_t version;            // 版本号 = 2
    ab_slot_t active_slot;       // 活动分区 (A/B)
    ab_slot_meta_t slots[2];     // 两个分区的元数据
    uint32_t crc32;              // CRC32 校验
} ab_metadata_t;
```

### 4.2 分区状态机

- **IDLE** → 写入新固件 → **TESTING**
- **TESTING** → App 确认 → **CONFIRMED**
- **TESTING** → 启动失败/超限3次 → **ROLLBACK** → 切换到另一分区

各状态说明：

- **IDLE**：初始状态，分区无有效固件
- **TESTING**：新固件写入后标记为测试状态，启动时增加 `boot_attempts`
- **CONFIRMED**：App 运行正常后主动确认，标记为稳定版本
- **INVALID**：固件校验失败

### 4.3 追加写入策略

64KB 的 Metadata 区域被划分为 128 字节对齐的实例（最多 512 个）。每次更新元数据时，追加写入下一个实例位置，而非擦除重写。当实例用完或位与不匹配时，执行**压缩**：擦除整个扇区后重新写入最新数据。

```c
// 追加写入核心逻辑
int ab_partition_write_metadata(const ab_metadata_t *meta) {
    // 找到下一个空闲实例位置
    uint32_t addr = find_next_free_instance();
    if (addr == 0) {
        // 实例用完，执行压缩
        return compact_metadata(meta);
    }
    // 追加写入
    return flash_write(addr, meta, sizeof(ab_metadata_t));
}
```

---

## 五、安全固件包系统

### 5.1 固件包格式

自定义 `.iap.bin` 格式，将加密、签名、校验融为一体：

| 区域        | 大小     | 说明                          |
| ----------- | -------- | ----------------------------- |
| Header      | 64 bytes | 魔术字、版本、加密/签名算法等 |
| DynamicSalt | 16 bytes | 随机盐值，每次打包不同        |
| IV          | 16 bytes | AES 初始化向量                |
| Ciphertext  | N bytes  | AES-256-CBC/CTR 加密的固件    |
| Signature   | 64 bytes | Ed25519 数字签名              |

### 5.2 Header 结构

```c
typedef struct __attribute__((packed)) {
    uint32_t magic;                  // 0x01504149
    uint8_t  header_version;         // 1
    uint8_t  firmware_major;         // 主版本号
    uint8_t  firmware_minor;         // 次版本号
    uint8_t  firmware_patch;         // 补丁版本号
    uint32_t total_payload_size;     // Salt + IV + Ciphertext 总大小
    uint8_t  image_type;             // APP=0x01, BOOTLOADER=0x02
    uint8_t  encryption_algo;        // NONE=0, AES256-CBC=1, ECB=2, CTR=3
    uint8_t  signature_algo;         // NONE=0, ED25519=1
    uint32_t hardware_compat;        // 硬件兼容性标识
    uint32_t security_counter;       // 安全计数器（防回滚）
    uint32_t build_timestamp;        // 构建时间戳
    uint8_t  reserved[5];
    uint8_t  header_checksum[32];    // HMAC-SHA256 校验
} fw_pkg_header_t;
```

### 5.3 安全下载全流程

| 步骤 | 操作                                              |
| ---- | ------------------------------------------------- |
| 1    | 读取 Header → 验证 Magic + Header Version         |
| 2    | HMAC-SHA256 验证 Header（使用设备 DevKey）        |
| 3    | 硬件兼容性检查                                    |
| 4    | 安全计数器防回滚检查                              |
| 5    | 读取 DynamicSalt + IV                             |
| 6    | HKDF 密钥派生：AES_Key = HKDF(Salt, DevKey, UID)  |
| 7    | 初始化 AES-256 解密上下文                         |
| 8    | 循环：读密文(4KB) → AES解密 → 写入Flash           |
| 9    | 读取 Ed25519 签名                                 |
| 10   | SHA-512 流式哈希验证（Header+Salt+IV+Ciphertext） |
| 11   | Ed25519 签名验证                                  |
| 12   | 回读 Flash → SHA256 完整性校验                    |
| 13   | 更新 A/B 元数据 → 设置活动分区(TESTING)           |

### 5.4 密钥派生：设备绑定加密

这是安全体系的核心设计。每台 STM32 芯片都有唯一的 UID（12 字节，地址 `0x1FFF7A10`），通过 HKDF 将 DevKey 与 UID 绑定：

```text
AES_Key = HKDF(
    salt     = DynamicSalt,   // 16 bytes, 每个固件包随机生成
    IKM      = DevKey,        // 16 bytes, 存储在 OTP 中
    info     = UID,           // 12 bytes, STM32 唯一ID
    key_len  = 32 bytes
)
```

**效果**：即使 DevKey 相同，不同设备派生出的 AES 密钥也不同。固件包只能由目标设备解密，**实现设备绑定加密**。

### 5.5 密钥存储方案

| 密钥               | 存储位置            | 特性                 |
| ------------------ | ------------------- | -------------------- |
| DevKey (16B)       | STM32 OTP 区域      | 一次性写入，不可修改 |
| Ed25519 公钥 (32B) | 固件中硬编码        | 可公开               |
| UID (12B)          | STM32 唯一ID 寄存器 | 出厂固化             |
| AES 密钥 (32B)     | 运行时 HKDF 派生    | 不存储，用完即弃     |

---

## 六、地址重定位：一份固件适配两个分区

### 6.1 问题

Slot A 起始地址 `0x08020000`，Slot B 起始地址 `0x08080000`。为 Slot A 编译的固件，其中断向量表和绝对地址跳转都指向 Slot A 的地址范围，直接写入 Slot B 无法运行。

### 6.2 解决方案

写入 Slot B 时，对每个 32 位字进行扫描，如果值落在 Slot A 地址范围内，自动加上偏移量：

```c
#define RELOCATE_OFFSET  0x00060000  // Slot B - Slot A

static uint32_t internal_flash_relocate_word(uint32_t word) {
    if (word >= SLOT_A_START_ADDR && word < SLOT_A_END_ADDR) {
        return word + RELOCATE_OFFSET;
    }
    return word;
}
```

**注意**：地址重定位能处理大部分绝对地址的修正（如中断向量表、函数指针等），但并非万能。实际工程中仍需在 Keil 中配置两个 Target，通过各自的链接脚本（Scatter File）控制编译地址，确保绝对地址和相对地址的寻址正确。重定位机制的意义在于：**App 侧只需维护一份源码**，通过切换 Target 编译出 Slot A / Slot B 两个版本的固件，而无需在代码层面做地址适配。

> [!WARNING]
> 地址重定位的局限性：它只能识别和修正**落在 Slot A 地址范围内的绝对地址字面量**。对于通过 PC 相对偏移寻址的指令（Thumb-2 的 BL/BLX 等），由于偏移量与基地址无关，重定位不会也不需要修改。但如果代码中有通过运行时计算得出的地址（如函数指针数组、虚表等），重定位可能无法覆盖所有情况，因此仍需双 Target 编译保证正确性。

---

## 七、平台抽象层：Transport 架构

### 7.1 统一接口设计

通过 `platform_transport_base_t` 抽象出 Source（读取）和 Target（写入）两种角色：

```c
// Source 接口 - 从某处读取数据
typedef struct {
    int16_t (*open)(const void* ctx, const char* path, uint32_t* total_size);
    int16_t (*read)(const void* ctx, uint8_t* buf, uint32_t size, uint32_t* bytes_read);
    int16_t (*close)(const void* ctx);
} platform_transport_source_ops_t;

// Target 接口 - 向某处写入数据
typedef struct {
    int16_t (*open)(const void* ctx, const char* path, uint32_t total_size);
    int16_t (*write)(const void* ctx, uint32_t offset, const uint8_t* data, uint32_t len);
    int16_t (*read)(const void* ctx, uint32_t offset, uint8_t* buf, uint32_t size, uint32_t* bytes_read);
    int16_t (*close)(const void* ctx);
} platform_transport_target_ops_t;
```

### 7.2 已实现的 Transport

| Transport                | Source | Target | 说明                          |
| ------------------------ | :----: | :----: | ----------------------------- |
| `g_slot_a_flash`         |   -    |  Yes   | Slot A 内部 Flash             |
| `g_slot_b_flash`         |   -    |  Yes   | Slot B 内部 Flash（带重定位） |
| `g_download_cache_flash` |   -    |  Yes   | 下载缓存区                    |
| `g_fatfs_transport`      |  Yes   |  Yes   | SD 卡 FatFS                   |
| `g_lfs_transport`        |  Yes   |  Yes   | SPI Flash LittleFS            |

### 7.3 灵活组合

Transport 架构使得下载逻辑与存储介质完全解耦：

```c
// 从 SD 卡下载到 Slot A
bootloader_download(g_fatfs_transport, "firmware.bin",
                    g_slot_a_flash, NULL);

// 从 SPI Flash 下载到 Slot B（自动重定位）
bootloader_download(g_lfs_transport, "firmware.bin",
                    g_slot_b_flash, NULL);
```

---

## 八、多渠道固件升级

### 8.1 UART YMODEM

通过串口进行固件下载，支持 128 字节和 1K 字节包大小。`Ymodem_Receive_To_Slot()` 支持直接下载到指定的 A/B 分区。

### 8.2 SD 卡升级

从 SD 卡读取 `.bin` / `.iap.bin` / `.hdiff` 文件，通过 FatFS Transport 读取后写入内部 Flash。

### 8.3 SPI Flash 升级

从 W25Q128 SPI Flash 读取固件文件（通过 LittleFS 文件系统管理），适用于无 SD 卡卡槽的产品。

### 8.4 WiFi OTA 升级

通过 ESP8266 WiFi 模块连接 OneNET 物联网平台，支持 MQTT 协议接收升级通知、HTTP 下载固件包、进度上报和结果通知。

### 8.5 差分升级 (HPatch + tuz)

使用 HPatch Lite + tuz 压缩算法实现差分升级，大幅减少传输数据量：

```c
// 差分升级核心流程
int service_hpatch_apply(const char *patch_path,
                         platform_transport_base_t *patch_source,
                         ab_slot_t target_slot) {
    // 1. 读取旧固件（当前活动分区）
    // 2. 读取差分文件
    // 3. HPatch 合成新固件 → 写入目标分区
    // 缓冲区：16KB cache + 4KB dict + 8KB stream
}
```

---

## 九、交互式菜单系统

Bootloader 提供了功能丰富的交互式菜单，通过 UART4 访问。所有功能项通过宏定义控制编译，可在 `menu.h` 中或编译选项中配置。

### 9.1 宏控制开关

| 宏名                           | 默认值   | 说明               |
| ------------------------------ | -------- | ------------------ |
| `MENU_ENABLE_DOWNLOAD`         | 1 (启用) | 固件下载           |
| `MENU_ENABLE_UPLOAD`           | 0 (禁用) | 固件上传           |
| `MENU_ENABLE_SPI_FLASH_STORE`  | 1 (启用) | SPI Flash 存储     |
| `MENU_ENABLE_EXECUTE_APP`      | 1 (启用) | 执行应用程序       |
| `MENU_ENABLE_FLASH_PROTECTION` | 0 (禁用) | Flash 写保护切换   |
| `MENU_ENABLE_AES_DECRYPT`      | 0 (禁用) | AES 解密           |
| `MENU_ENABLE_HPATCH`           | 1 (启用) | HPatch 差分升级    |
| `MENU_ENABLE_UART_PASSTHROUGH` | 0 (禁用) | UART 串口透传      |
| `MENU_ENABLE_ESP8266_WIFI`     | 0 (禁用) | ESP8266 WiFi & OTA |
| `MENU_ENABLE_ED25519_VERIFY`   | 0 (禁用) | Ed25519 签名验证   |
| `MENU_ENABLE_RNG_DEVKEY`       | 0 (禁用) | RNG 设备密钥生成   |
| `MENU_ENABLE_FIRMWARE_PACKAGE` | 1 (启用) | 固件包解析         |

> [!NOTE]
> 所有宏均采用 `#ifndef` 守卫模式，可通过编译器选项 `-DMENU_ENABLE_XXX=1` 在外部覆盖默认值。只有 `[F] A/B Partition Management` 无条件编译，始终存在。

### 9.2 完整菜单树

| 快捷键  | 菜单                                    | 子菜单                                           | 控制宏                         | 说明               |
| ------- | --------------------------------------- | ------------------------------------------------ | ------------------------------ | ------------------ |
| **[1]** | Download image                          |                                                  | `MENU_ENABLE_DOWNLOAD`         | 固件下载           |
|         |                                         | [1] Download via Serial (Ymodem)                 |                                | 串口下载           |
|         |                                         | [2] Download from SD card (FATFS)                |                                | SD卡下载           |
|         |                                         | [3] Download from SPI Flash (LittleFS)           |                                | SPI Flash下载      |
| **[2]** | Upload image from internal Flash        | -                                                | `MENU_ENABLE_UPLOAD`           | Ymodem上传         |
| **[3]** | Store image to SPI-Flash LFS            |                                                  | `MENU_ENABLE_SPI_FLASH_STORE`  | SPI Flash存储      |
|         |                                         | [1] Store image from TF card                     |                                | SD卡转存           |
|         |                                         | [2] Store image from Flash                       |                                | 内部Flash转存      |
|         |                                         | [3] Show stored images                           |                                | 查看已存储文件     |
|         |                                         | [4] Delete stored image                          |                                | 删除文件           |
|         |                                         | [5] Delete entire file system                    |                                | 删除整个文件系统   |
| **[4]** | Execute the loaded application          | -                                                | `MENU_ENABLE_EXECUTE_APP`      | 跳转到App          |
| **[5]** | Toggle Flash write protection           | -                                                | `MENU_ENABLE_FLASH_PROTECTION` | Flash写保护切换    |
| **[6]** | Decrypt and download encrypted firmware |                                                  | `MENU_ENABLE_AES_DECRYPT`      | AES解密下载        |
|         |                                         | [1] Decrypt from SD card and download to Flash   |                                | SD卡解密下载       |
|         |                                         | [2] Decrypt from SPI Flash and download to Flash |                                | SPI Flash解密下载  |
| **[7]** | Decrypt .bin.aes file on SD card        | -                                                | `MENU_ENABLE_AES_DECRYPT`      | SD卡AES解密        |
| **[8]** | HPatch differential upgrade             |                                                  | `MENU_ENABLE_HPATCH`           | 差分升级           |
|         |                                         | [1] HPatch upgrade from SD card                  |                                | SD卡差分升级       |
|         |                                         | [2] HPatch upgrade from SPI Flash                |                                | SPI Flash差分升级  |
| **[9]** | UART4 <-> USART1 Passthrough            | -                                                | `MENU_ENABLE_UART_PASSTHROUGH` | 串口透传           |
| **[A]** | ESP8266 WiFi & OTA Test                 |                                                  | `MENU_ENABLE_ESP8266_WIFI`     | WiFi与OTA          |
|         |                                         | [1] WiFi Init & Connect AP                       |                                | WiFi初始化连接     |
|         |                                         | [2] AT Command Test                              |                                | AT指令测试         |
|         |                                         | [3] TCP Connect Test                             |                                | TCP连接测试        |
|         |                                         | [4] Enter Transparent Mode                       |                                | 进入透传模式       |
|         |                                         | [5] Exit Transparent Mode                        |                                | 退出透传模式       |
|         |                                         | [6] Set OTA Target: Internal Flash               |                                | OTA目标：内部Flash |
|         |                                         | [7] Set OTA Target: SD Card (FATFS)              |                                | OTA目标：SD卡      |
|         |                                         | [8] Set OTA Target: SPI Flash (LFS)              |                                | OTA目标：SPI Flash |
|         |                                         | [9] OneNET OTA Download                          |                                | OneNET OTA下载     |
|         |                                         | [A] Show Current Time                            |                                | 显示当前时间       |
|         |                                         | [B] MQTT Test Menu →                             |                                | MQTT子菜单         |
|         |                                         |   [1] Check MQTT Connection Status               |                                | 检查MQTT连接       |
|         |                                         |   [2] Configure MQTT User                        |                                | 配置MQTT用户       |
|         |                                         |   [3] Connect to MQTT Server                     |                                | 连接MQTT服务器     |
|         |                                         |   [4] Subscribe Property Topics                  |                                | 订阅属性主题       |
|         |                                         |   [5] Publish Property                           |                                | 发布属性           |
|         |                                         |   [6] Listen & Auto Reply Property Set           |                                | 监听并自动回复     |
|         |                                         |   [7] Disconnect MQTT                            |                                | 断开MQTT           |
|         |                                         |   [8] Sync Time (SNTP)                           |                                | SNTP时间同步       |
|         |                                         |   [9] Publish RTC Time (1s interval)             |                                | 周期发布RTC时间    |
| **[B]** | Ed25519 Signature Verify                |                                                  | `MENU_ENABLE_ED25519_VERIFY`   | 签名验证           |
|         |                                         | [1] Verify firmware on SD card                   |                                | SD卡签名验证       |
|         |                                         | [2] Verify firmware on SPI Flash                 |                                | SPI Flash签名验证  |
|         |                                         | [3] Buffer verify test                           |                                | 缓冲区验证测试     |
| **[C]** | Generate Device Key (RNG)               | -                                                | `MENU_ENABLE_RNG_DEVKEY`       | RNG生成设备密钥    |
| **[D]** | Write DevKey to OTP                     | -                                                | `MENU_ENABLE_RNG_DEVKEY`       | 写入OTP设备密钥    |
| **[E]** | Firmware Package Parse                  |                                                  | `MENU_ENABLE_FIRMWARE_PACKAGE` | 固件包解析         |
|         |                                         | [1] Parse package from SD card (FATFS)           |                                | SD卡解析           |
|         |                                         | [2] Parse package from SPI Flash (LFS)           |                                | SPI Flash解析      |
|         |                                         | [3] Secure Download to A/B (SD Card)             |                                | 安全下载到A/B分区  |
|         |                                         | [4] Secure Download to A/B (SPI Flash)           |                                | 安全下载到A/B分区  |
| **[F]** | A/B Partition Management                |                                                  | 无（始终启用）                 | A/B分区管理        |
|         |                                         | [1] Show A/B Partition Status                    |                                | 查看分区状态       |
|         |                                         | [2] Set Active Slot                              |                                | 设置活动分区       |
|         |                                         | [3] Confirm Current Slot                         |                                | 确认当前分区       |
|         |                                         | [4] Rollback                                     |                                | 回滚               |
|         |                                         | [5] Jump to Slot                                 |                                | 跳转到指定分区     |

---

## 十、安全信任链：锚点、传播与保证

### 10.1 信任链全局架构

安全 Bootloader 的核心不是某个单一的加密算法，而是一条从**硬件信任根**出发、逐层传递的信任链。每一环的验证成功是下一环执行的前提，任何一环失败则整个流程中止，固件不会被写入 Flash。

```
信任根 (RoT)
  ├── UID (0x1FFF7A10, 出厂固化, 12B)
  └── DevKey (OTP, 一次性写入, 16B)
       │
       ▼
第一关: HMAC-SHA256 Header 认证
  "这个包是持有 DevKey 的人生成的吗？"
       │
       ▼
第二关: 硬件兼容性 + 安全计数器防回滚
  "这个包是为本硬件准备的吗？版本没有降级吗？"
       │
       ▼
第三关: HKDF 设备绑定密钥派生
  AES_Key = HKDF(Salt, DevKey, UID)
  "即使 DevKey 泄露，也只有目标设备能派生出正确的 AES 密钥"
       │
       ▼
第四关: AES-256 解密
  "只有目标设备能还原出明文固件"
       │
       ▼
第五关: Ed25519 数字签名验证
  SHA-512(Header‖Salt‖IV‖Ciphertext) → Ed25519 Verify
  "固件内容由私钥持有者签发，未被篡改"
       │
       ▼
第六关: SHA-256 Flash 回读校验
  "写入 Flash 的数据与解密输出完全一致"
       │
       ▼
信任终点: A/B 分区状态转换 (TESTING → CONFIRMED)
  "新固件经过运行验证后才成为正式版本"
```

### 10.2 信任根：硬件锚点

信任链必须有不可伪造的起点。本项目的信任根由两个硬件原语构成：

**STM32 唯一 ID (UID)**

- 地址：`0x1FFF7A10`，12 字节（96 位）
- 特性：每颗芯片出厂时由 ST 烧录，不可修改，不可擦除
- 作用：作为 HKDF 的 `info` 参数，将密钥派生绑定到特定芯片

```c
#define STM32F4_UID_ADDR 0x1FFF7A10
// 读取方式
const uint8_t *uid = (const uint8_t *)STM32F4_UID_ADDR;
```

**设备密钥 (DevKey)**

- 存储位置：STM32 OTP（One-Time Programmable）区域
- 大小：16 字节（128 位）
- 特性：OTP 区域只能从 0 写为 1，一旦写入不可修改不可擦除
- 作用：HMAC-SHA256 认证密钥 + HKDF 密钥派生的输入密钥材料 (IKM)

> [!IMPORTANT]
> DevKey 的 OTP 存储是信任根的关键安全属性。OTP 的"一次性写入"语义意味着：生产线上烧录 DevKey 后，即使攻击者获得了 JTAG 访问权限，也无法覆盖或修改 DevKey。这与 Flash 存储有本质区别——Flash 可以擦除重写，而 OTP 不行。

**信任根的形式化保证**：

$$\text{RoT} = \{\text{UID}_{\text{chip}}, \text{DevKey}_{\text{OTP}}\}$$

其中 $\text{UID}_{\text{chip}}$ 满足不可伪造性（出厂固化），$\text{DevKey}_{\text{OTP}}$ 满足不可修改性（OTP 语义）。

### 10.3 第一关：HMAC-SHA256 Header 认证

**目的**：在处理固件包的任何内容之前，首先验证 Header 的来源可信。

**机制**：

```c
fw_pkg_err_t fw_pkg_verify_header_hmac(const fw_pkg_ctx_t *ctx,
                                       const fw_pkg_verify_config_t *config)
{
    uint8_t computed_hmac[32];
    const uint8_t *header_prefix = (const uint8_t *)&ctx->header;

    // HMAC-SHA256(DevKey, Header[0..31])  ← Header 前 32 字节
    hmac_sha256(config->devkey, config->devkey_len,
                header_prefix, FW_PKG_HEADER_SIZE - FW_PKG_HMAC_SIZE,
                computed_hmac);

    // 与 Header 中嵌入的 HMAC 比对
    if (memcmp(computed_hmac, ctx->header.header_checksum, 32) != 0)
        return FW_PKG_ERR_HMAC;

    return FW_PKG_OK;
}
```

**安全属性**：

- **认证性**：只有持有 DevKey 的打包工具才能生成正确的 HMAC
- **选择性**：HMAC 仅覆盖 Header，不覆盖载荷——这是刻意设计，因为载荷的完整性由 Ed25519 签名保证
- **早拒绝**：Header 认证在读取 Salt/IV/Ciphertext 之前执行，无效包在最小 I/O 后即被拒绝

**为什么用 HMAC 而非直接用 Ed25519 签名 Header？**

HMAC-SHA256 计算速度远快于 Ed25519 签名验证（微秒级 vs 毫秒级），作为第一道防线可以在大量无效包的 DoS 场景下快速拒绝。Ed25519 签名验证放在最后，对整个包（Header + Salt + IV + Ciphertext）做完整性校验。

### 10.4 第二关：防回滚与硬件兼容性

**安全计数器 (security_counter)**

每个固件包的 Header 中携带一个单调递增的 `security_counter`，设备端存储当前已确认的最大计数器值：

```c
fw_pkg_err_t fw_pkg_check_rollback(const fw_pkg_ctx_t *ctx,
                                   const fw_pkg_verify_config_t *config)
{
    if (ctx->header.security_counter < config->stored_security_counter)
    {
        // 检测到回滚！包中的计数器 < 设备存储的计数器
        return FW_PKG_ERR_ROLLBACK;
    }
    return FW_PKG_OK;
}
```

**防回滚的形式化保证**：

设设备存储的计数器为 $c_{\text{stored}}$，固件包中的计数器为 $c_{\text{pkg}}$，则：

$$c_{\text{pkg}} \geq c_{\text{stored}} \implies \text{允许升级}$$
$$c_{\text{pkg}} < c_{\text{stored}} \implies \text{拒绝（回滚攻击）}$$

**硬件兼容性**：`hardware_compat` 字段确保固件包与目标硬件匹配，防止将不兼容的固件写入设备导致功能异常。

### 10.5 第三关：HKDF 设备绑定密钥派生

这是整个安全体系最核心的设计——**设备绑定加密**。即使攻击者截获了固件包和 DevKey，没有目标设备的 UID 也无法解密。

**HKDF (HMAC-based Extract-and-Expand Key Derivation Function, RFC 5869)**

```
AES_Key = HKDF(
    salt     = DynamicSalt,    // 16 字节，每个固件包随机生成
    IKM      = DevKey,         // 16 字节，存储在 OTP 中
    info     = UID,            // 12 字节，STM32 唯一 ID
    key_len  = 32 字节         // AES-256 密钥长度
)
```

HKDF 分两阶段工作：

1. **Extract（提取）**：`PRK = HMAC-Hash(salt, IKM)` — 将 DevKey 的熵提取到伪随机密钥 PRK 中
2. **Expand（扩展）**：`OKM = HMAC-Hash(PRK, info ∥ 0x01)` — 将 PRK 扩展为指定长度的输出密钥，info（即 UID）确保不同设备得到不同的密钥

**设备绑定的安全性分析**：

| 攻击场景 | 攻击者拥有 | 能否解密？ | 原因 |
|----------|-----------|-----------|------|
| 截获固件包 | Salt, IV, Ciphertext | 否 | 没有 DevKey 和 UID，无法派生 AES_Key |
| 截获固件包 + DevKey | Salt, IV, Ciphertext, DevKey | 否 | 没有 UID，HKDF(info) 输出不同 |
| 同 DevKey 不同设备 | Salt, IV, Ciphertext, DevKey, UID_B | 否 | UID_B ≠ UID_A，派生的 AES_Key 不同 |
| 目标设备本身 | Salt, IV, Ciphertext, DevKey, UID | **是** | 三要素齐全，正确派生 AES_Key |

**DynamicSalt 的作用**：每包随机生成的 16 字节盐值，使得即使对同一设备推送相同固件，两次派生的 AES 密钥也不同，消除密钥复用风险。

### 10.6 第四关：AES-256 解密与流式处理

支持三种加密模式，根据 Header 中的 `encryption_algo` 字段选择：

| 模式 | 常量 | 特点 |
|------|------|------|
| AES-256-CBC | `FW_PKG_ENC_AES256_CBC` | 最常用，PKCS7 填充，需 IV |
| AES-256-CTR | `FW_PKG_ENC_AES256_CTR` | 流式友好，无需填充 |
| AES-256-ECB | `FW_PKG_ENC_AES256_ECB` | 不推荐，仅用于兼容 |

**流式解密写入**：固件包可能远大于可用 RAM，因此采用 4KB 缓冲区循环处理：

```c
static uint8_t process_buf[4096] __attribute__((aligned(4)));

while (ciphertext_remaining > 0) {
    uint32_t to_read = MIN(ciphertext_remaining, sizeof(process_buf));
    SOURCE_READ(source, process_buf, to_read, &bytes_read);   // 读密文
    fw_pkg_decrypt_payload(&ctx, process_buf, bytes_read,      // AES 解密
                           is_final, &actual_len);
    TARGET_WRITE(target, flash_offset, process_buf, bytes_read); // 写 Flash
    flash_offset += bytes_read;
    ciphertext_remaining -= bytes_read;
}
```

**关键安全属性**：AES 密钥（`aes_key[32]`）是运行时通过 HKDF 派生的局部变量，函数返回后即从栈上消失，**不持久化存储**。即使攻击者随后获得了设备访问权限，也无法从内存中找到 AES 密钥。

### 10.7 第五关：Ed25519 数字签名验证

Ed25519 是 Bernstein 等人设计的 Curve25519 上的数字签名方案，提供 128 位安全强度。

**签名范围**：Ed25519 签名覆盖整个固件包的明文部分：

$$\text{hash} = \text{SHA-512}(\text{Header} \| \text{Salt} \| \text{IV} \| \text{Ciphertext})$$
$$\text{Verify}_{\text{PK}}(\text{signature}, \text{hash}) \stackrel{?}{=} \text{true}$$

**流式 SHA-512 计算**：由于固件包可能很大，SHA-512 采用流式更新：

```c
sha512_init(&sig_state);  // 在读取 Header 之前初始化

// 每读入一块数据，喂入 SHA-512
fw_pkg_sha512_feed(&sig_state, data, len, pending, &pending_len);

// 所有数据读完后，计算最终哈希
fw_pkg_sha512_finish(&sig_state, pending, pending_len, total_len, hash);

// Ed25519 验证
edsign_verify(signature, ed25519_pubkey, hash, 64);
```

**签名 vs HMAC 的互补关系**：

| 属性 | HMAC-SHA256 | Ed25519 |
|------|-------------|---------|
| 密钥类型 | 对称密钥 (DevKey) | 非对称密钥对 |
| 验证范围 | 仅 Header | Header + Salt + IV + Ciphertext |
| 验证速度 | 快（微秒级） | 慢（毫秒级） |
| 安全保证 | 来源认证 | 来源认证 + 完整性 |
| 密钥分发 | DevKey 预共享 | 公钥硬编码，私钥离线保存 |

HMAC 使用对称密钥，意味着持有 DevKey 的人（打包工具和设备）都能生成有效 HMAC。Ed25519 使用非对称密钥，**只有持有私钥的人才能签名**，设备端仅持有公钥用于验证。这确保了即使 DevKey 泄露，攻击者也无法伪造有效的 Ed25519 签名。

**公钥存储**：

```c
static const uint8_t FW_PKG_ED25519_PUBLIC_KEY[32] = {
    0xc2, 0xe9, 0xbf, 0x62, 0x02, 0x92, ...
};
```

公钥硬编码在 Bootloader 固件中，作为 Ed25519 验证的信任锚。对应的私钥只存在于离线的固件签名服务器上，永不部署到设备端。

### 10.8 第六关：SHA-256 Flash 回读校验

前五关保证了"收到的包是可信的"和"解密出的固件是正确的"，但 Flash 写入可能因硬件故障导致位翻转。第六关通过回读验证写入的正确性：

```c
// 解密完成后，回读整个 Flash 区域并计算 SHA-256
sha256_init(&sha256_ctx);
for (offset = 0; offset < firmware_size; offset += chunk) {
    TARGET_READ(target, offset, process_buf, chunk, &bytes_read);
    sha256_update(&sha256_ctx, process_buf, bytes_read);
}
sha256_final(&sha256_ctx, computed_sha256);

// 与固件末尾内嵌的 SHA-256 比对
if (memcmp(computed_sha256, stored_sha256, 32) == 0)
    printf("  Result: MATCH\r\n");
else
    printf("  Result: MISMATCH!\r\n");
```

这一关提供 **端到端完整性保证**：从打包工具的 SHA-256 嵌入，到写入 Flash，再到回读验证，构成了完整的数据完整性闭环。

### 10.9 A/B 分区作为安全网

信任链的终点不是"固件写入成功"，而是"固件运行正常"。A/B 分区提供了运行时验证的安全网：

```
新固件写入非活动分区
  ├── 状态设为 TESTING
  ├── boot_attempts = 0
  │
  ▼
启动新固件
  ├── 运行正常 → App 调用 ab_partition_mark_slot_confirmed()
  │                 状态转为 CONFIRMED
  │
  └── 运行异常
      ├── boot_attempts 递增
      ├── boot_attempts >= 3 → 自动回滚到旧分区
      └── SP 验证失败 → 立即回滚
```

`ab_partition_validate_slot()` 的双重检查：

```c
ab_err_t ab_partition_validate_slot(ab_slot_t slot) {
    uint32_t sp = *(__IO uint32_t *)addr;
    if (!ab_is_valid_sp(sp))           // 1. 栈指针必须在 SRAM 范围内
        return AB_ERR_SLOT_INVALID_FW;

    uint32_t reset_handler = *(__IO uint32_t *)(addr + 4);
    if (reset_handler < addr ||        // 2. ResetHandler 必须落在本分区地址范围内
        reset_handler > end_addr)
        return AB_ERR_SLOT_INVALID_FW;

    return AB_OK;
}
```

### 10.10 信任链的形式化总结

整个信任链可以表示为一个验证函数的链式组合，每个函数的成功是下一个函数执行的前提：

$$V_{\text{total}} = V_{\text{HMAC}} \circ V_{\text{compat}} \circ V_{\text{rollback}} \circ V_{\text{HKDF}} \circ V_{\text{AES}} \circ V_{\text{Ed25519}} \circ V_{\text{SHA256}} \circ V_{\text{A/B}}$$

其中每个 $V_i$ 的输出为 $\{\text{PASS}, \text{FAIL}\}$，且：

$$V_{\text{total}} = \text{PASS} \iff \forall i, V_i = \text{PASS}$$

| 验证环节 | 安全属性 | 攻击防护 | 信任根依赖 |
|----------|----------|----------|------------|
| HMAC-SHA256 | 来源认证 | 伪造 Header | DevKey |
| 硬件兼容 | 正确性 | 错误硬件刷写 | 无（明文比对） |
| 安全计数器 | 版本单调性 | 回滚攻击 | stored_counter |
| HKDF | 设备绑定 | 跨设备解密 | DevKey + UID |
| AES-256 | 机密性 | 固件窃取 | HKDF 派生密钥 |
| Ed25519 | 完整性 + 认证 | 内容篡改 | Ed25519 公钥 |
| SHA-256 回读 | 写入完整性 | Flash 位翻转 | 无（回读比对） |
| A/B 分区 | 可用性 | 变砖攻击 | SP + ResetHandler |

---

## 十一、源码深入剖析

### 11.1 main.c：启动决策主入口

`main()` 是整个 Bootloader 的决策中心，核心逻辑如下：

```c
int main(void) {
    // ===== 软复位快速跳转（HAL_Init 之前） =====
    if (update_flag == JUMP_FLAG_MAGIC) {
        update_flag = 0;
        __DSB(); __ISB();
        ab_slot_t slot = ab_partition_get_active_slot_from_flash();
        bootloader_execute_jump();  // 直接跳转，最快启动
    }

    // ===== 标准初始化 =====
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_UART4_Init();
    MX_SPI1_Init();
    // ... 其他外设初始化
    platform_config_init();       // 注册所有 Transport 驱动
    ab_partition_init();          // 读取 A/B 元数据

    // ===== A/B 分区启动检查 =====
    ab_slot_t active = ab_partition_get_active_slot();
    ab_slot_state_t state = g_ab_metadata.slots[active].state;

    if (state == AB_SLOT_TESTING) {
        if (!ab_partition_validate_slot(active)) {
            ab_partition_rollback();        // 固件无效，回滚
        } else if (g_ab_metadata.slots[active].boot_attempts >= AB_MAX_BOOT_RETRIES) {
            ab_partition_rollback();        // 超过3次，回滚
        } else {
            ab_partition_increment_boot_attempts(active);  // 递增尝试计数
        }
    }

    // ===== SD卡挂载 =====
    if (SD_Detect() == SD_PRESENT) {
        f_mount(&SDFatFS, SDPath, 1);
    }

    // ===== 菜单入口判断 =====
    Serial_PutString("\r\n按 'M' 进入 Bootloader 菜单 (1.5s 超时)...\r\n");
    if (WaitForSerialCommand(1500) == 'M') {
        Main_Menu();             // 进入交互式菜单
    } else if (update_flag == UPDATE_FLAG_MAGIC) {
        Main_Menu();             // App 请求更新
    } else {
        bootloader_request_jump(active);  // 自动跳转 App
    }
}
```

> [!NOTE]
> 软复位快速跳转放在 `HAL_Init()` **之前**执行，这意味着此时所有外设都未初始化，但 `bootloader_execute_jump()` 只操作内核寄存器（MSP、VTOR），不依赖任何外设，因此是安全的。这种设计将 App 启动延迟降到了最低。

### 11.2 bootloader_core.c：下载引擎

`bootloader_download()` 是普通下载的核心，通过 Transport 抽象层实现源到目标的数据搬运：

```c
int bootloader_download(platform_transport_base_t *source,
                        platform_transport_base_t *target,
                        const char *path) {
    uint8_t buf[DOWNLOAD_BUF_SIZE];  // 4096 字节缓冲区
    uint32_t total_size = 0;
    uint32_t bytes_read = 0;
    uint32_t offset = 0;

    // 1. 打开源传输
    int ret = TRANSPORT_SOURCE_OPEN(source, path, &total_size);
    if (ret != TRANSPORT_STATUS_OK) return BL_ERR_OPEN_SRC;

    // 2. 打开目标传输
    ret = TRANSPORT_TARGET_OPEN(target, NULL, total_size);
    if (ret != TRANSPORT_STATUS_OK) {
        TRANSPORT_SOURCE_CLOSE(source);
        return BL_ERR_OPEN_DST;
    }

    // 3. 循环读取 → 写入
    do {
        ret = TRANSPORT_SOURCE_READ(source, buf, sizeof(buf), &bytes_read);
        if (ret != TRANSPORT_STATUS_OK) break;

        if (bytes_read > 0) {
            ret = TRANSPORT_TARGET_WRITE(target, offset, buf, bytes_read);
            if (ret != TRANSPORT_STATUS_OK) break;
            offset += bytes_read;
        }
    } while (bytes_read > 0);

    // 4. 关闭传输
    TRANSPORT_TARGET_CLOSE(target);
    TRANSPORT_SOURCE_CLOSE(source);

    return (ret == TRANSPORT_STATUS_OK) ? BL_OK : transport_status_to_bootloader(ret);
}
```

`bootloader_secure_download()` 则在普通下载的基础上，增加了完整的安全验证链路：

```c
int bootloader_secure_download(platform_transport_base_t *source,
                                platform_transport_base_t *target,
                                const char *path,
                                const fw_pkg_verify_config_t *config,
                                fw_pkg_ctx_t *result) {
    // 调用固件包处理引擎，完成：
    // HMAC验证 → 防回滚 → HKDF密钥派生 → AES解密 → Ed25519签名验证
    int ret = fw_pkg_process_ex(source, target, path, config, result);
    if (ret != FW_PKG_OK) return fw_pkg_err_to_bootloader(ret);
    return BL_OK;
}
```

### 11.3 firmware_package.c：安全固件包处理引擎

这是安全体系最核心的文件（1107 行），`fw_pkg_process_ex()` 的完整流程：

```c
int fw_pkg_process_ex(platform_transport_base_t *source,
                       platform_transport_base_t *target,
                       const char *path,
                       const fw_pkg_verify_config_t *config,
                       fw_pkg_ctx_t *ctx) {
    fw_pkg_header_t header;
    uint8_t salt[SALT_SIZE], iv[IV_SIZE];
    uint8_t signature[SIGNATURE_SIZE];
    aes_ctx_t aes_ctx;
    sha512_ctx_t sha512_ctx;
    sha256_ctx_t sha256_ctx;

    // ===== 1. 读取并解析 Header =====
    SOURCE_READ(source, &header, sizeof(header));
    fw_pkg_parse_header(&header);  // 验证 Magic + Version

    // ===== 2. HMAC-SHA256 验证 Header =====
    fw_pkg_verify_header_hmac(&header, config->devkey);

    // ===== 3. 硬件兼容性检查 =====
    if (header.hardware_compat != config->hardware_id)
        return FW_PKG_ERR_HW_INCOMPAT;

    // ===== 4. 防回滚检查 =====
    fw_pkg_check_rollback(header.security_counter, config->min_security_counter);

    // ===== 5. 读取 DynamicSalt + IV =====
    SOURCE_READ(source, salt, SALT_SIZE);
    SOURCE_READ(source, iv, IV_SIZE);

    // ===== 6. HKDF 密钥派生 =====
    uint8_t aes_key[32];
    fw_pkg_derive_aes_key(salt, config->devkey, config->uid, aes_key);

    // ===== 7. 初始化 AES 解密 =====
    fw_pkg_decrypt_init(&aes_ctx, header.encryption_algo, aes_key, iv);

    // ===== 8. 打开目标，循环解密写入 =====
    TARGET_OPEN(target, NULL, ciphertext_size);
    uint32_t offset = 0;
    while (remaining > 0) {
        uint32_t chunk = MIN(remaining, sizeof(buf));
        SOURCE_READ(source, buf, chunk, &bytes_read);
        fw_pkg_decrypt_payload(&aes_ctx, buf, bytes_read, is_last);
        TARGET_WRITE(target, offset, buf, bytes_read);
        offset += bytes_read;
        remaining -= bytes_read;
    }

    // ===== 9. 读取并验证 Ed25519 签名 =====
    SOURCE_READ(source, signature, SIGNATURE_SIZE);
    fw_pkg_verify_signature(&header, salt, iv, ciphertext,
                            ciphertext_size, signature, config->ed25519_pubkey);

    // ===== 10. 回读 Flash SHA-256 校验 =====
    sha256_init(&sha256_ctx);
    for (offset = 0; offset < firmware_size; offset += sizeof(buf)) {
        TARGET_READ(target, offset, buf, chunk);
        sha256_update(&sha256_ctx, buf, chunk);
    }
    sha256_final(&sha256_ctx, computed_hash);
    // 与固件末尾内嵌的 SHA-256 比对

    // ===== 11. 填充结果上下文 =====
    ctx->fw_version = header.firmware_version;
    ctx->security_counter = header.security_counter;
    memcpy(ctx->sha256, computed_hash, 32);

    return FW_PKG_OK;
}
```

> [!IMPORTANT]
> HKDF 密钥派生是设备绑定加密的关键。`fw_pkg_derive_aes_key()` 的三个输入中，`DynamicSalt` 每包随机生成，`DevKey` 存储在 OTP 中每台设备不同，`UID` 是芯片出厂唯一ID。三者组合确保：即使固件包被截获，也无法在其他设备上解密。

### 11.4 ab_partition.c：A/B 分区管理

元数据的追加写入策略是 A/B 分区管理的精髓：

```c
// 元数据实例对齐：128 字节对齐，最多 512 个实例
#define METADATA_INSTANCE_ALIGN  128
#define METADATA_MAX_INSTANCES   (METADATA_SIZE / METADATA_INSTANCE_ALIGN)

static int ab_write_metadata_raw(const ab_metadata_t *meta) {
    uint32_t addr = METADATA_START_ADDR + g_ab_current_instance * METADATA_INSTANCE_ALIGN;

    // 检查能否直接写入（Flash 只允许 1→0）
    if (can_write_at(addr, meta, sizeof(*meta))) {
        ab_flash_write_words(addr, (const uint32_t *)meta,
                             sizeof(*meta) / 4);
        g_ab_current_instance++;
    } else {
        // 无法追加写入，执行压缩
        ab_compact_and_write(meta);
    }
}
```

回滚逻辑确保设备始终可启动：

```c
int ab_partition_rollback(void) {
    ab_slot_t current = g_ab_metadata.active_slot;
    ab_slot_t fallback = (current == AB_SLOT_A) ? AB_SLOT_B : AB_SLOT_A;

    // 标记当前槽位为无效
    g_ab_metadata.slots[current].state = AB_SLOT_INVALID;

    // 验证回退槽位有有效固件
    if (!ab_partition_validate_slot(fallback)) {
        return AB_ERR_NO_VALID_SLOT;  // 两个槽位都无效！
    }

    // 切换到回退槽位
    g_ab_metadata.active_slot = fallback;
    g_ab_metadata.slots[fallback].state = AB_SLOT_CONFIRMED;
    g_ab_metadata.slots[fallback].boot_attempts = 0;
    ab_partition_metadata_flush();

    return AB_OK;
}
```

> [!WARNING]
> 如果两个槽位都无效（`ab_partition_rollback` 返回 `AB_ERR_NO_VALID_SLOT`），设备将无法自动启动，只能停留在 Bootloader 菜单等待手动刷入固件。这是 A/B 分区最后的防线——设备不会变砖，但需要人工干预。

### 11.5 service_hpatch.c：差分升级实现

差分升级使用 HPatch Lite + tuz 压缩，内存占用可控：

```c
// 静态分配的缓冲区（编译期确定）
static uint8_t tuz_dict_and_cache[HPATCH_DICT_SIZE + HPATCH_CACHE_SIZE];  // 4KB + 16KB
static uint8_t stream_diff_buf[HPATCH_STREAM_BUF_SIZE];                   // 8KB
static uint8_t patch_temp_buf[HPATCH_CACHE_SIZE];                         // 16KB

int hpatch_upgrade(hpatch_config_t *config) {
    // 1. 打开差分文件、旧固件、创建输出文件
    fs_instance->open(&diff_file, config->diff_path, "rb");
    fs_instance->open(&old_file, config->old_path, "rb");
    fs_instance->open(&out_file, config->out_path, "wb");

    // 2. 解析差分文件头
    hpatch_lite_open(&listener, &new_size, &compress_type);

    // 3. 若使用 tuz 压缩，初始化解压流
    if (compress_type == hpi_compress_tuz) {
        tuz_TStream_open(&tuz_stream_obj, tuz_dict_and_cache,
                         dict_size, tuz_fs_read_code);
        // 替换读取回调为流式解压
        listener.read_diff = stream_read_diff;
    }

    // 4. 执行差分补丁应用
    hpatch_lite_patch(&listener, new_size, patch_temp_buf);

    // 5. 清理
    fs_instance->close(&diff_file);
    fs_instance->close(&old_file);
    fs_instance->close(&out_file);
}
```

> [!TIP]
> 差分升级的典型效果：假设旧固件 384KB，新固件 386KB，差分文件可能只有 5-20KB。在带宽受限的场景（如 NB-IoT、LoRa）下，差分升级能显著减少传输时间和流量费用。

### 11.6 esp8266_ota_api.c：WiFi OTA 封装

WiFi OTA 是对 OneNET 物联网平台的薄封装：

```c
static onenet_ota_ctx_t g_ota_ctx;

void esp8266_ota_init(void) {
    // 注入三个依赖：WiFi模块、RTC（用于MQTT认证时间戳）、MQTT客户端
    onenet_ota_ctx_init(&g_ota_ctx, &g_esp8266_wifi.base,
                        &g_rtc.base, &g_esp8266_mqtt.base);
}

int esp8266_ota_download(void) {
    onenet_ota_process_upgrade(&g_ota_ctx);  // MQTT接收 → HTTP下载 → 写入目标
    return 1;
}
```

支持三种下载目标：

```c
void esp8266_ota_set_target_internal_flash(void);  // 直接写入内部 Flash
void esp8266_ota_set_target_sd_card(void);          // 保存到 SD 卡
void esp8266_ota_set_target_spi_flash(void);         // 保存到 SPI Flash
```

### 11.7 整体分层架构

| 层级       | 模块                                           | 说明                                   |
| ---------- | ---------------------------------------------- | -------------------------------------- |
| **应用层** | Menu                                           | 交互式菜单，4600+行                    |
| **核心层** | Bootloader Core                                | 下载/跳转/安全下载                     |
|            | ├ firmware_package                             | 包解析/HMAC/AES/Ed25519                |
|            | └ ab_partition                                 | A/B分区/回滚/元数据管理                |
| **服务层** | Service Layer                                  | 功能服务                               |
|            | ├ service_hpatch                               | 差分升级                               |
|            | ├ esp8266_ota_api                              | WiFi OTA (OneNET)                      |
|            | └ service_aes_decrypt / service_ed25519_verify | 加解密/签名验证                        |
| **抽象层** | Platform Transport                             | 传输抽象                               |
|            | ├ Source                                       | FatFS / LittleFS / Ymodem / HTTP       |
|            | └ Target                                       | Internal Flash / FatFS / LittleFS      |
| **硬件层** | HAL                                            | STM32F407: Flash/RNG/SPI/UART/SDIO/RTC |

---

## 十二、总结

本项目是一个功能完备的工业级安全 Bootloader，核心亮点：

1. **A/B 双分区**：无缝升级 + 自动回滚，设备永不变砖
2. **完整安全体系**：HMAC 认证 + AES-256 加密 + Ed25519 签名 + 防回滚 + 设备绑定密钥
3. **地址重定位 + 双 Target 编译**：源码只需一份，简化构建流程
4. **Transport 抽象**：下载逻辑与存储介质完全解耦，易于扩展
5. **多渠道升级**：UART / SD卡 / SPI Flash / WiFi OTA，覆盖各种场景
6. **差分升级**：HPatch + tuz 减少传输数据量
7. **交互式菜单**：功能丰富，便于调试和生产使用

---

## 参考资料

- [STM32F4xx Reference Manual](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [Ed25519: High-speed high-security signatures](https://ed25519.cr.yp.to/)
- [HKDF (RFC 5869)](https://tools.ietf.org/html/rfc5869)
- [HPatchLite - 差分补丁库](https://github.com/sisong/HPatchLite)
- [LittleFS - 嵌入式文件系统](https://github.com/littlefs-project/littlefs)
