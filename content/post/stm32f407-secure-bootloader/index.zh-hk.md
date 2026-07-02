---
title: "STM32F407 安全 Bootloader 設計：A/B 雙分區、加密簽名與多渠道升級"
date: 2026-06-20
description: "基於 STM32F407VGT6 的工業級安全 Bootloader 完整設計，涵蓋 A/B 雙分區無縫升級、AES-256 加密 + Ed25519 簽名安全固件包、地址重定位、差分升級、多渠道下載（UART/SD卡/SPI Flash/WiFi OTA）等核心技術詳解"
image: STM32F407安全Bootloader设计.png
categories:
  - "嵌入式"
tags:
  - "STM32"
  - "Bootloader"
  - "IAP"
  - "A/B分區"
  - "固件加密"
  - "Ed25519"
  - "OTA"
math: true
---

## 前言

在物聯網和嵌入式設備領域，固件升級（OTA）是產品生命週期中不可或缺的一環。然而，升級過程中的**斷電崩潰、固件篡改、版本回滾**等問題，往往讓開發者頭疼不已。本文將詳細介紹一個基於 STM32F407VGT6 的安全 Bootloader 項目，它通過 **A/B 雙分區**、**安全固件包**、**地址重定位** 等機制，系統性地解決了這些痛點。

---

## 一、解決了什麼問題？

| 問題                       | 傳統方案                 | 本項目方案                                    |
| -------------------------- | ------------------------ | --------------------------------------------- |
| 升級中斷電導致設備變磚     | 單分區直接覆蓋，斷電即死 | A/B 雙分區，升級失敗自動回滾                  |
| 固件被篡改或偽造           | 無校驗或僅 CRC           | Ed25519 簽名 + HMAC-SHA256 認證               |
| 固件被竊取逆向             | 明文傳輸                 | AES-256-CBC/CTR 加密 + 設備綁定密鑰           |
| 舊版本固件降級攻擊         | 無防護                   | 安全計數器防回滾                              |
| 為兩個分區維護兩套編譯配置 | 維護兩份工程             | 地址自動重定位 + 雙 Target 編譯，源碼只需一份 |
| 升級包過大佔用帶寬         | 全量升級                 | HPatch + tuz 差分升級                         |

---

## 二、Flash 內存佈局

STM32F407VGT6 擁有 1MB 內部 Flash，分區如下：

| 起始地址      | 區域           | 大小   | Flash 扇區       |
| ------------- | -------------- | ------ | ---------------- |
| `0x0800_0000` | Bootloader     | 128 KB | Sector 0-4       |
| `0x0802_0000` | Slot A         | 384 KB | Sector 5-7       |
| `0x0808_0000` | Slot B         | 384 KB | Sector 8-10      |
| `0x080E_0000` | Download Cache | 64 KB  | Sector 11 (部分) |
| `0x080F_0000` | Metadata       | 64 KB  | Sector 11 (部分) |

> [!TIP]
> 為什麼 Bootloader 佔用 128KB？因為本項目目前為測試 Demo，未做精簡優化——實際上一個精簡的 Bootloader 通常只需 32-48KB。當前 128KB 中包含了完整的菜單系統、加密庫（AES/Ed25519/HKDF）、HPatch 差分、ESP8266 WiFi/MQTT 等所有功能模塊。在量產化時，可通過裁剪不需要的功能，將 Bootloader 縮小至合理大小，從而為 App 釋放更多 Flash 空間。

### 鏈接腳本（Scatter File）

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
   *(.bss.NoInit)                    ; Bootloader/App 通信標誌
  }
}
```

**關鍵設計**：`NoInit` 區域位於 `0x2001BFFC`，存放 `update_flag` 變量。軟復位後該變量不被清零，用於 Bootloader 和 App 之間的狀態傳遞。

> [!NOTE]
> `update_flag` 的魔術值定義：
>
> - `0x5A5A5A5A` (`UPDATE_FLAG_MAGIC`) — App 請求固件更新
> - `0xA5A5A5A5` (`JUMP_FLAG_MAGIC`) — 請求跳轉到 App（軟復位後快速跳轉）
> - `0x424F4F54` (`BOOT_CONFIRM_MAGIC`) — 啟動確認標記

---

## 三、啟動流程與跳轉機制

### 3.1 完整啟動流程

1. **上電/復位**
2. 檢查 `update_flag == JUMP_FLAG_MAGIC`？→ 是：清除 flag → 直接跳轉 App（快速啟動）
3. `HAL_Init()` → `SystemClock_Config()` → 外設初始化
4. A/B 分區狀態檢查：
   - **TESTING** 狀態 → 驗證固件有效性
     - 無效 → 回滾到另一分區
     - 啟動嘗試超過3次 → 回滾
     - 有效 → 增加啟動嘗試計數
   - **CONFIRMED** 狀態 → 正常啟動
5. 等待 UART 命令（1.5秒超時）
   - 收到 'M' → 進入 Bootloader 菜單
   - 超時 → 繼續
6. 檢查 `update_flag == UPDATE_FLAG_MAGIC`？→ 是：進入 Bootloader 菜單（App 請求更新）
7. 自動跳轉到活動分區的 App

### 3.2 跳轉核心代碼

跳轉分兩步實現，通過軟復位確保硬件狀態乾淨：

**第一步：請求跳轉（設置標誌 + 軟復位）**

```c
void bootloader_request_jump(ab_slot_t slot) {
    uint32_t app_addr = bootloader_get_slot_addr(slot);
    if (app_addr == 0 || !bootloader_validate_sp(app_addr)) return;

    ab_partition_set_active_slot(slot);  // 設置活動分區
    update_flag = JUMP_FLAG_MAGIC;       // 設置跳轉標誌
    __DSB(); __ISB();
    NVIC_SystemReset();                  // 軟復位
}
```

**第二步：實際執行跳轉（復位後執行）**

```c
void bootloader_execute_jump(void) {
    uint32_t app_addr = bootloader_get_slot_addr(AB_SLOT_AUTO);
    uint32_t app_stack_ptr = *(__IO uint32_t *)app_addr;
    uint32_t app_reset_handler = *(__IO uint32_t *)(app_addr + 4);

    SysTick->CTRL = 0;           // 關閉 SysTick
    __set_PRIMASK(0);            // 使能中斷
    SCB->VTOR = app_addr;        // 重定向中斷向量表
    __DSB(); __ISB();
    __set_MSP(app_stack_ptr);    // 設置主棧指針
    ((pFunction)app_reset_handler)();  // 跳轉到 App ResetHandler
}
```

**SP 驗證**確保棧指針在合法 SRAM 範圍內：

```c
bool bootloader_validate_sp(uint32_t app_addr) {
    uint32_t sp = *(__IO uint32_t *)app_addr;
    return (sp >= 0x20000000 && sp <= 0x2FFE0000);
}
```

> [!TIP]
> 為什麼跳轉分兩步？直接跳轉雖然更快，但軟復位能確保所有外設和中斷狀態被乾淨地重置。本項目在正常啟動流程中使用 `request_jump`（軟復位），而在 `update_flag == JUMP_FLAG_MAGIC` 時直接調用 `execute_jump` 實現快速啟動——因為此時是剛從 App 軟復位回來，硬件狀態已經是乾淨的。

---

## 四、A/B 雙分區系統

### 4.1 元數據結構

元數據存儲在 Flash 末尾的 Metadata 區域，採用**追加寫入**策略避免頻繁擦除：

```c
typedef struct __attribute__((packed)) {
    uint32_t fw_version;         // 固件版本
    uint32_t security_counter;   // 安全計數器（防回滾）
    uint32_t fw_size;            // 固件大小
    ab_slot_state_t state;       // 分區狀態
    uint8_t boot_attempts;       // 啟動嘗試次數
    uint8_t sha256[32];          // 固件 SHA256
    uint8_t reserved[3];
} ab_slot_meta_t;

typedef struct __attribute__((packed)) {
    uint32_t magic;              // 0x41425441 ("ABTA")
    uint32_t version;            // 版本號 = 2
    ab_slot_t active_slot;       // 活動分區 (A/B)
    ab_slot_meta_t slots[2];     // 兩個分區的元數據
    uint32_t crc32;              // CRC32 校驗
} ab_metadata_t;
```

### 4.2 分區狀態機

- **IDLE** → 寫入新固件 → **TESTING**
- **TESTING** → App 確認 → **CONFIRMED**
- **TESTING** → 啟動失敗/超限3次 → **ROLLBACK** → 切換到另一分區

各狀態說明：

- **IDLE**：初始狀態，分區無有效固件
- **TESTING**：新固件寫入後標記為測試狀態，啟動時增加 `boot_attempts`
- **CONFIRMED**：App 運行正常後主動確認，標記為穩定版本
- **INVALID**：固件校驗失敗

### 4.3 追加寫入策略

64KB 的 Metadata 區域被劃分為 128 字節對齊的實例（最多 512 個）。每次更新元數據時，追加寫入下一個實例位置，而非擦除重寫。當實例用完或位與不匹配時，執行**壓縮**：擦除整個扇區後重新寫入最新數據。

```c
// 追加寫入核心邏輯
int ab_partition_write_metadata(const ab_metadata_t *meta) {
    // 找到下一個空閒實例位置
    uint32_t addr = find_next_free_instance();
    if (addr == 0) {
        // 實例用完，執行壓縮
        return compact_metadata(meta);
    }
    // 追加寫入
    return flash_write(addr, meta, sizeof(ab_metadata_t));
}
```

---

## 五、安全固件包系統

### 5.1 固件包格式

自定義 `.iap.bin` 格式，將加密、簽名、校驗融為一體：

| 區域        | 大小     | 說明                          |
| ----------- | -------- | ----------------------------- |
| Header      | 64 bytes | 魔術字、版本、加密/簽名算法等 |
| DynamicSalt | 16 bytes | 隨機鹽值，每次打包不同        |
| IV          | 16 bytes | AES 初始化向量                |
| Ciphertext  | N bytes  | AES-256-CBC/CTR 加密的固件    |
| Signature   | 64 bytes | Ed25519 數字簽名              |

### 5.2 Header 結構

```c
typedef struct __attribute__((packed)) {
    uint32_t magic;                  // 0x01504149
    uint8_t  header_version;         // 1
    uint8_t  firmware_major;         // 主版本號
    uint8_t  firmware_minor;         // 次版本號
    uint8_t  firmware_patch;         // 補丁版本號
    uint32_t total_payload_size;     // Salt + IV + Ciphertext 總大小
    uint8_t  image_type;             // APP=0x01, BOOTLOADER=0x02
    uint8_t  encryption_algo;        // NONE=0, AES256-CBC=1, ECB=2, CTR=3
    uint8_t  signature_algo;         // NONE=0, ED25519=1
    uint32_t hardware_compat;        // 硬件兼容性標識
    uint32_t security_counter;       // 安全計數器（防回滾）
    uint32_t build_timestamp;        // 構建時間戳
    uint8_t  reserved[5];
    uint8_t  header_checksum[32];    // HMAC-SHA256 校驗
} fw_pkg_header_t;
```

### 5.3 安全下載全流程

| 步驟 | 操作                                              |
| ---- | ------------------------------------------------- |
| 1    | 讀取 Header → 驗證 Magic + Header Version         |
| 2    | HMAC-SHA256 驗證 Header（使用設備 DevKey）        |
| 3    | 硬件兼容性檢查                                    |
| 4    | 安全計數器防回滾檢查                              |
| 5    | 讀取 DynamicSalt + IV                             |
| 6    | HKDF 密鑰派生：AES_Key = HKDF(Salt, DevKey, UID)  |
| 7    | 初始化 AES-256 解密上下文                         |
| 8    | 循環：讀密文(4KB) → AES解密 → 寫入Flash           |
| 9    | 讀取 Ed25519 簽名                                 |
| 10   | SHA-512 流式哈希驗證（Header+Salt+IV+Ciphertext） |
| 11   | Ed25519 簽名驗證                                  |
| 12   | 回讀 Flash → SHA256 完整性校驗                    |
| 13   | 更新 A/B 元數據 → 設置活動分區(TESTING)           |

### 5.4 密鑰派生：設備綁定加密

這是安全體系的核心設計。每台 STM32 芯片都有唯一的 UID（12 字節，地址 `0x1FFF7A10`），通過 HKDF 將 DevKey 與 UID 綁定：

```text
AES_Key = HKDF(
    salt     = DynamicSalt,   // 16 bytes, 每個固件包隨機生成
    IKM      = DevKey,        // 16 bytes, 存儲在 OTP 中
    info     = UID,           // 12 bytes, STM32 唯一ID
    key_len  = 32 bytes
)
```

**效果**：即使 DevKey 相同，不同設備派生出的 AES 密鑰也不同。固件包只能由目標設備解密，**實現設備綁定加密**。

### 5.5 密鑰存儲方案

| 密鑰               | 存儲位置            | 特性                 |
| ------------------ | ------------------- | -------------------- |
| DevKey (16B)       | STM32 OTP 區域      | 一次性寫入，不可修改 |
| Ed25519 公鑰 (32B) | 固件中硬編碼        | 可公開               |
| UID (12B)          | STM32 唯一ID 寄存器 | 出廠固化             |
| AES 密鑰 (32B)     | 運行時 HKDF 派生    | 不存儲，用完即棄     |

---

## 六、地址重定位：一份固件適配兩個分區

### 6.1 問題

Slot A 起始地址 `0x08020000`，Slot B 起始地址 `0x08080000`。為 Slot A 編譯的固件，其中斷向量表和絕對地址跳轉都指向 Slot A 的地址範圍，直接寫入 Slot B 無法運行。

### 6.2 解決方案

寫入 Slot B 時，對每個 32 位字進行掃描，如果值落在 Slot A 地址範圍內，自動加上偏移量：

```c
#define RELOCATE_OFFSET  0x00060000  // Slot B - Slot A

static uint32_t internal_flash_relocate_word(uint32_t word) {
    if (word >= SLOT_A_START_ADDR && word < SLOT_A_END_ADDR) {
        return word + RELOCATE_OFFSET;
    }
    return word;
}
```

**注意**：地址重定位能處理大部分絕對地址的修正（如中斷向量表、函數指針等），但並非萬能。實際工程中仍需在 Keil 中配置兩個 Target，通過各自的鏈接腳本（Scatter File）控制編譯地址，確保絕對地址和相對地址的尋址正確。重定位機制的意義在於：**App 側只需維護一份源碼**，通過切換 Target 編譯出 Slot A / Slot B 兩個版本的固件，而無需在代碼層面做地址適配。

> [!WARNING]
> 地址重定位的局限性：它只能識別和修正**落在 Slot A 地址範圍內的絕對地址字面量**。對於通過 PC 相對偏移尋址的指令（Thumb-2 的 BL/BLX 等），由於偏移量與基地址無關，重定位不會也不需要修改。但如果代碼中有通過運行時計算得出的地址（如函數指針數組、虛表等），重定位可能無法覆蓋所有情況，因此仍需雙 Target 編譯保證正確性。

---

## 七、平台抽象層：Transport 架構

### 7.1 統一接口設計

通過 `platform_transport_base_t` 抽象出 Source（讀取）和 Target（寫入）兩種角色：

```c
// Source 接口 - 從某處讀取數據
typedef struct {
    int16_t (*open)(const void* ctx, const char* path, uint32_t* total_size);
    int16_t (*read)(const void* ctx, uint8_t* buf, uint32_t size, uint32_t* bytes_read);
    int16_t (*close)(const void* ctx);
} platform_transport_source_ops_t;

// Target 接口 - 向某處寫入數據
typedef struct {
    int16_t (*open)(const void* ctx, const char* path, uint32_t total_size);
    int16_t (*write)(const void* ctx, uint32_t offset, const uint8_t* data, uint32_t len);
    int16_t (*read)(const void* ctx, uint32_t offset, uint8_t* buf, uint32_t size, uint32_t* bytes_read);
    int16_t (*close)(const void* ctx);
} platform_transport_target_ops_t;
```

### 7.2 已實現的 Transport

| Transport                | Source | Target | 說明                          |
| ------------------------ | :----: | :----: | ----------------------------- |
| `g_slot_a_flash`         |   -    |  Yes   | Slot A 內部 Flash             |
| `g_slot_b_flash`         |   -    |  Yes   | Slot B 內部 Flash（帶重定位） |
| `g_download_cache_flash` |   -    |  Yes   | 下載緩存區                    |
| `g_fatfs_transport`      |  Yes   |  Yes   | SD 卡 FatFS                   |
| `g_lfs_transport`        |  Yes   |  Yes   | SPI Flash LittleFS            |

### 7.3 靈活組合

Transport 架構使得下載邏輯與存儲介質完全解耦：

```c
// 從 SD 卡下載到 Slot A
bootloader_download(g_fatfs_transport, "firmware.bin",
                    g_slot_a_flash, NULL);

// 從 SPI Flash 下載到 Slot B（自動重定位）
bootloader_download(g_lfs_transport, "firmware.bin",
                    g_slot_b_flash, NULL);
```

---

## 八、多渠道固件升級

### 8.1 UART YMODEM

通過串口進行固件下載，支持 128 字節和 1K 字節包大小。`Ymodem_Receive_To_Slot()` 支持直接下載到指定的 A/B 分區。

### 8.2 SD 卡升級

從 SD 卡讀取 `.bin` / `.iap.bin` / `.hdiff` 文件，通過 FatFS Transport 讀取後寫入內部 Flash。

### 8.3 SPI Flash 升級

從 W25Q128 SPI Flash 讀取固件文件（通過 LittleFS 文件系統管理），適用於無 SD 卡卡槽的產品。

### 8.4 WiFi OTA 升級

通過 ESP8266 WiFi 模塊連接 OneNET 物聯網平台，支持 MQTT 協議接收升級通知、HTTP 下載固件包、進度上報和結果通知。

### 8.5 差分升級 (HPatch + tuz)

使用 HPatch Lite + tuz 壓縮算法實現差分升級，大幅減少傳輸數據量：

```c
// 差分升級核心流程
int service_hpatch_apply(const char *patch_path,
                         platform_transport_base_t *patch_source,
                         ab_slot_t target_slot) {
    // 1. 讀取舊固件（當前活動分區）
    // 2. 讀取差分文件
    // 3. HPatch 合成新固件 → 寫入目標分區
    // 緩衝區：16KB cache + 4KB dict + 8KB stream
}
```

---

## 九、交互式菜單系統

Bootloader 提供了功能豐富的交互式菜單，通過 UART4 訪問。所有功能項通過宏定義控制編譯，可在 `menu.h` 中或編譯選項中配置。

### 9.1 宏控制開關

| 宏名                           | 默認值   | 說明               |
| ------------------------------ | -------- | ------------------ |
| `MENU_ENABLE_DOWNLOAD`         | 1 (啟用) | 固件下載           |
| `MENU_ENABLE_UPLOAD`           | 0 (禁用) | 固件上傳           |
| `MENU_ENABLE_SPI_FLASH_STORE`  | 1 (啟用) | SPI Flash 存儲     |
| `MENU_ENABLE_EXECUTE_APP`      | 1 (啟用) | 執行應用程序       |
| `MENU_ENABLE_FLASH_PROTECTION` | 0 (禁用) | Flash 寫保護切換   |
| `MENU_ENABLE_AES_DECRYPT`      | 0 (禁用) | AES 解密           |
| `MENU_ENABLE_HPATCH`           | 1 (啟用) | HPatch 差分升級    |
| `MENU_ENABLE_UART_PASSTHROUGH` | 0 (禁用) | UART 串口透傳      |
| `MENU_ENABLE_ESP8266_WIFI`     | 0 (禁用) | ESP8266 WiFi & OTA |
| `MENU_ENABLE_ED25519_VERIFY`   | 0 (禁用) | Ed25519 簽名驗證   |
| `MENU_ENABLE_RNG_DEVKEY`       | 0 (禁用) | RNG 設備密鑰生成   |
| `MENU_ENABLE_FIRMWARE_PACKAGE` | 1 (啟用) | 固件包解析         |

> [!NOTE]
> 所有宏均採用 `#ifndef` 守衛模式，可通過編譯器選項 `-DMENU_ENABLE_XXX=1` 在外部覆蓋默認值。只有 `[F] A/B Partition Management` 無條件編譯，始終存在。

### 9.2 完整菜單樹

| 快捷鍵  | 菜單                                    | 子菜單                                           | 控制宏                         | 說明               |
| ------- | --------------------------------------- | ------------------------------------------------ | ------------------------------ | ------------------ |
| **[1]** | Download image                          |                                                  | `MENU_ENABLE_DOWNLOAD`         | 固件下載           |
|         |                                         | [1] Download via Serial (Ymodem)                 |                                | 串口下載           |
|         |                                         | [2] Download from SD card (FATFS)                |                                | SD卡下載           |
|         |                                         | [3] Download from SPI Flash (LittleFS)           |                                | SPI Flash下載      |
| **[2]** | Upload image from internal Flash        | -                                                | `MENU_ENABLE_UPLOAD`           | Ymodem上傳         |
| **[3]** | Store image to SPI-Flash LFS            |                                                  | `MENU_ENABLE_SPI_FLASH_STORE`  | SPI Flash存儲      |
|         |                                         | [1] Store image from TF card                     |                                | SD卡轉存           |
|         |                                         | [2] Store image from Flash                       |                                | 內部Flash轉存      |
|         |                                         | [3] Show stored images                           |                                | 查看已存儲文件     |
|         |                                         | [4] Delete stored image                          |                                | 刪除文件           |
|         |                                         | [5] Delete entire file system                    |                                | 刪除整個文件系統   |
| **[4]** | Execute the loaded application          | -                                                | `MENU_ENABLE_EXECUTE_APP`      | 跳轉到App          |
| **[5]** | Toggle Flash write protection           | -                                                | `MENU_ENABLE_FLASH_PROTECTION` | Flash寫保護切換    |
| **[6]** | Decrypt and download encrypted firmware |                                                  | `MENU_ENABLE_AES_DECRYPT`      | AES解密下載        |
|         |                                         | [1] Decrypt from SD card and download to Flash   |                                | SD卡解密下載       |
|         |                                         | [2] Decrypt from SPI Flash and download to Flash |                                | SPI Flash解密下載  |
| **[7]** | Decrypt .bin.aes file on SD card        | -                                                | `MENU_ENABLE_AES_DECRYPT`      | SD卡AES解密        |
| **[8]** | HPatch differential upgrade             |                                                  | `MENU_ENABLE_HPATCH`           | 差分升級           |
|         |                                         | [1] HPatch upgrade from SD card                  |                                | SD卡差分升級       |
|         |                                         | [2] HPatch upgrade from SPI Flash                |                                | SPI Flash差分升級  |
| **[9]** | UART4 <-> USART1 Passthrough            | -                                                | `MENU_ENABLE_UART_PASSTHROUGH` | 串口透傳           |
| **[A]** | ESP8266 WiFi & OTA Test                 |                                                  | `MENU_ENABLE_ESP8266_WIFI`     | WiFi與OTA          |
|         |                                         | [1] WiFi Init & Connect AP                       |                                | WiFi初始化連接     |
|         |                                         | [2] AT Command Test                              |                                | AT指令測試         |
|         |                                         | [3] TCP Connect Test                             |                                | TCP連接測試        |
|         |                                         | [4] Enter Transparent Mode                       |                                | 進入透傳模式       |
|         |                                         | [5] Exit Transparent Mode                        |                                | 退出透傳模式       |
|         |                                         | [6] Set OTA Target: Internal Flash               |                                | OTA目標：內部Flash |
|         |                                         | [7] Set OTA Target: SD Card (FATFS)              |                                | OTA目標：SD卡      |
|         |                                         | [8] Set OTA Target: SPI Flash (LFS)              |                                | OTA目標：SPI Flash |
|         |                                         | [9] OneNET OTA Download                          |                                | OneNET OTA下載     |
|         |                                         | [A] Show Current Time                            |                                | 顯示當前時間       |
|         |                                         | [B] MQTT Test Menu →                             |                                | MQTT子菜單         |
|         |                                         | [1] Check MQTT Connection Status                 |                                | 檢查MQTT連接       |
|         |                                         | [2] Configure MQTT User                          |                                | 配置MQTT用戶       |
|         |                                         | [3] Connect to MQTT Server                       |                                | 連接MQTT伺服器     |
|         |                                         | [4] Subscribe Property Topics                    |                                | 訂閱屬性主題       |
|         |                                         | [5] Publish Property                             |                                | 發佈屬性           |
|         |                                         | [6] Listen & Auto Reply Property Set             |                                | 監聽並自動回覆     |
|         |                                         | [7] Disconnect MQTT                              |                                | 斷開MQTT           |
|         |                                         | [8] Sync Time (SNTP)                             |                                | SNTP時間同步       |
|         |                                         | [9] Publish RTC Time (1s interval)               |                                | 週期發佈RTC時間    |
| **[B]** | Ed25519 Signature Verify                |                                                  | `MENU_ENABLE_ED25519_VERIFY`   | 簽名驗證           |
|         |                                         | [1] Verify firmware on SD card                   |                                | SD卡簽名驗證       |
|         |                                         | [2] Verify firmware on SPI Flash                 |                                | SPI Flash簽名驗證  |
|         |                                         | [3] Buffer verify test                           |                                | 緩衝區驗證測試     |
| **[C]** | Generate Device Key (RNG)               | -                                                | `MENU_ENABLE_RNG_DEVKEY`       | RNG生成設備密鑰    |
| **[D]** | Write DevKey to OTP                     | -                                                | `MENU_ENABLE_RNG_DEVKEY`       | 寫入OTP設備密鑰    |
| **[E]** | Firmware Package Parse                  |                                                  | `MENU_ENABLE_FIRMWARE_PACKAGE` | 固件包解析         |
|         |                                         | [1] Parse package from SD card (FATFS)           |                                | SD卡解析           |
|         |                                         | [2] Parse package from SPI Flash (LFS)           |                                | SPI Flash解析      |
|         |                                         | [3] Secure Download to A/B (SD Card)             |                                | 安全下載到A/B分區  |
|         |                                         | [4] Secure Download to A/B (SPI Flash)           |                                | 安全下載到A/B分區  |
| **[F]** | A/B Partition Management                |                                                  | 無（始終啟用）                 | A/B分區管理        |
|         |                                         | [1] Show A/B Partition Status                    |                                | 查看分區狀態       |
|         |                                         | [2] Set Active Slot                              |                                | 設置活動分區       |
|         |                                         | [3] Confirm Current Slot                         |                                | 確認當前分區       |
|         |                                         | [4] Rollback                                     |                                | 回滾               |
|         |                                         | [5] Jump to Slot                                 |                                | 跳轉到指定分區     |

---

## 十、安全信任鏈：錨點、傳播與保證

### 10.1 信任鏈全局架構

安全 Bootloader 的核心不是某個單一的加密算法，而是一條從**硬體信任根**出發、逐層傳遞的信任鏈。每一環的驗證成功是下一環執行的前提，任何一環失敗則整個流程中止，固件不會被寫入 Flash。

```
信任根 (RoT)
  ├── UID (0x1FFF7A10, 出廠固化, 12B)
  └── DevKey (OTP, 一次性寫入, 16B)
       │
       ▼
第一關: HMAC-SHA256 Header 認證
  "這個包是持有 DevKey 的人生成的嗎？"
       │
       ▼
第二關: 硬體相容性 + 安全計數器防回滾
  "這個包是為本硬體準備的嗎？版本沒有降級嗎？"
       │
       ▼
第三關: HKDF 設備綁定密鑰派生
  AES_Key = HKDF(Salt, DevKey, UID)
  "即使 DevKey 洩露，也只有目標設備能派生出正確的 AES 密鑰"
       │
       ▼
第四關: AES-256 解密
  "只有目標設備能還原出明文固件"
       │
       ▼
第五關: Ed25519 數位簽章驗證
  SHA-512(Header‖Salt‖IV‖Ciphertext) → Ed25519 Verify
  "固件內容由私鑰持有者簽發，未被篡改"
       │
       ▼
第六關: SHA-256 Flash 回讀校驗
  "寫入 Flash 的數據與解密輸出完全一致"
       │
       ▼
信任終點: A/B 分區狀態轉換 (TESTING → CONFIRMED)
  "新固件經過運行驗證後才成為正式版本"
```

### 10.2 信任根：硬體錨點

信任鏈必須有不可偽造的起點。本項目的信任根由兩個硬體原語構成：

**STM32 唯一 ID (UID)**

- 地址：`0x1FFF7A10`，12 字節（96 位）
- 特性：每顆芯片出廠時由 ST 燒錄，不可修改，不可擦除
- 作用：作為 HKDF 的 `info` 參數，將密鑰派生綁定到特定芯片

```c
#define STM32F4_UID_ADDR 0x1FFF7A10
// 讀取方式
const uint8_t *uid = (const uint8_t *)STM32F4_UID_ADDR;
```

**設備密鑰 (DevKey)**

- 存儲位置：STM32 OTP（One-Time Programmable）區域
- 大小：16 字節（128 位）
- 特性：OTP 區域只能從 0 寫為 1，一旦寫入不可修改不可擦除
- 作用：HMAC-SHA256 認證密鑰 + HKDF 密鑰派生的輸入密鑰材料 (IKM)

> [!IMPORTANT]
> DevKey 的 OTP 存儲是信任根的關鍵安全屬性。OTP 的「一次性寫入」語義意味著：生產線上燒錄 DevKey 後，即使攻擊者取得了 JTAG 存取權限，也無法覆蓋或修改 DevKey。這與 Flash 存儲有本質區別——Flash 可以擦除重寫，而 OTP 不行。

**信任根的形式化保證**：

$$\text{RoT} = \{\text{UID}_{\text{chip}}, \text{DevKey}_{\text{OTP}}\}$$

其中 $\text{UID}_{\text{chip}}$ 滿足不可偽造性（出廠固化），$\text{DevKey}_{\text{OTP}}$ 滿足不可修改性（OTP 語義）。

### 10.3 第一關：HMAC-SHA256 Header 認證

**目的**：在處理固件包的任何內容之前，首先驗證 Header 的來源可信。

**機制**：

```c
fw_pkg_err_t fw_pkg_verify_header_hmac(const fw_pkg_ctx_t *ctx,
                                       const fw_pkg_verify_config_t *config)
{
    uint8_t computed_hmac[32];
    const uint8_t *header_prefix = (const uint8_t *)&ctx->header;

    // HMAC-SHA256(DevKey, Header[0..31])  ← Header 前 32 字節
    hmac_sha256(config->devkey, config->devkey_len,
                header_prefix, FW_PKG_HEADER_SIZE - FW_PKG_HMAC_SIZE,
                computed_hmac);

    // 與 Header 中嵌入的 HMAC 比對
    if (memcmp(computed_hmac, ctx->header.header_checksum, 32) != 0)
        return FW_PKG_ERR_HMAC;

    return FW_PKG_OK;
}
```

**安全屬性**：

- **認證性**：只有持有 DevKey 的打包工具才能生成正確的 HMAC
- **選擇性**：HMAC 僅覆蓋 Header，不覆蓋載荷——這是刻意設計，因為載荷的完整性由 Ed25519 簽章保證
- **早拒絕**：Header 認證在讀取 Salt/IV/Ciphertext 之前執行，無效包在最小 I/O 後即被拒絕

**為什麼用 HMAC 而非直接用 Ed25519 簽章 Header？**

HMAC-SHA256 計算速度遠快於 Ed25519 簽章驗證（微秒級 vs 毫秒級），作為第一道防線可以在大量無效包的 DoS 場景下快速拒絕。Ed25519 簽章驗證放在最後，對整個包（Header + Salt + IV + Ciphertext）做完整性校驗。

### 10.4 第二關：防回滾與硬體相容性

**安全計數器 (security_counter)**

每個固件包的 Header 中攜帶一個單調遞增的 `security_counter`，設備端存儲當前已確認的最大計數器值：

```c
fw_pkg_err_t fw_pkg_check_rollback(const fw_pkg_ctx_t *ctx,
                                   const fw_pkg_verify_config_t *config)
{
    if (ctx->header.security_counter < config->stored_security_counter)
    {
        // 檢測到回滾！包中的計數器 < 設備存儲的計數器
        return FW_PKG_ERR_ROLLBACK;
    }
    return FW_PKG_OK;
}
```

**防回滾的形式化保證**：

設設備存儲的計數器為 $c_{\text{stored}}$，固件包中的計數器為 $c_{\text{pkg}}$，則：

$$c_{\text{pkg}} \geq c_{\text{stored}} \implies \text{允許升級}$$
$$c_{\text{pkg}} < c_{\text{stored}} \implies \text{拒絕（回滾攻擊）}$$

**硬體相容性**：`hardware_compat` 欄位確保固件包與目標硬體匹配，防止將不相容的固件寫入設備導致功能異常。

### 10.5 第三關：HKDF 設備綁定密鑰派生

這是整個安全體系最核心的設計——**設備綁定加密**。即使攻擊者截獲了固件包和 DevKey，沒有目標設備的 UID 也無法解密。

**HKDF (HMAC-based Extract-and-Expand Key Derivation Function, RFC 5869)**

```
AES_Key = HKDF(
    salt     = DynamicSalt,    // 16 字節，每個固件包隨機生成
    IKM      = DevKey,         // 16 字節，存儲在 OTP 中
    info     = UID,            // 12 字節，STM32 唯一 ID
    key_len  = 32 字節         // AES-256 密鑰長度
)
```

HKDF 分兩階段工作：

1. **Extract（提取）**：`PRK = HMAC-Hash(salt, IKM)` — 將 DevKey 的熵提取到偽隨機密鑰 PRK 中
2. **Expand（擴展）**：`OKM = HMAC-Hash(PRK, info ∥ 0x01)` — 將 PRK 擴展為指定長度的輸出密鑰，info（即 UID）確保不同設備得到不同的密鑰

**設備綁定的安全性分析**：

| 攻擊場景 | 攻擊者擁有 | 能否解密？ | 原因 |
|----------|-----------|-----------|------|
| 截獲固件包 | Salt, IV, Ciphertext | 否 | 沒有 DevKey 和 UID，無法派生 AES_Key |
| 截獲固件包 + DevKey | Salt, IV, Ciphertext, DevKey | 否 | 沒有 UID，HKDF(info) 輸出不同 |
| 同 DevKey 不同設備 | Salt, IV, Ciphertext, DevKey, UID_B | 否 | UID_B ≠ UID_A，派生的 AES_Key 不同 |
| 目標設備本身 | Salt, IV, Ciphertext, DevKey, UID | **是** | 三要素齊全，正確派生 AES_Key |

**DynamicSalt 的作用**：每包隨機生成的 16 字節鹽值，使得即使對同一設備推送相同固件，兩次派生的 AES 密鑰也不同，消除密鑰復用風險。

### 10.6 第四關：AES-256 解密與串流處理

支援三種加密模式，根據 Header 中的 `encryption_algo` 欄位選擇：

| 模式 | 常量 | 特點 |
|------|------|------|
| AES-256-CBC | `FW_PKG_ENC_AES256_CBC` | 最常用，PKCS7 填充，需 IV |
| AES-256-CTR | `FW_PKG_ENC_AES256_CTR` | 串流友好，無需填充 |
| AES-256-ECB | `FW_PKG_ENC_AES256_ECB` | 不推薦，僅用於相容 |

**串流解密寫入**：固件包可能遠大於可用 RAM，因此採用 4KB 緩衝區循環處理：

```c
static uint8_t process_buf[4096] __attribute__((aligned(4)));

while (ciphertext_remaining > 0) {
    uint32_t to_read = MIN(ciphertext_remaining, sizeof(process_buf));
    SOURCE_READ(source, process_buf, to_read, &bytes_read);   // 讀密文
    fw_pkg_decrypt_payload(&ctx, process_buf, bytes_read,      // AES 解密
                           is_final, &actual_len);
    TARGET_WRITE(target, flash_offset, process_buf, bytes_read); // 寫 Flash
    flash_offset += bytes_read;
    ciphertext_remaining -= bytes_read;
}
```

**關鍵安全屬性**：AES 密鑰（`aes_key[32]`）是運行時通過 HKDF 派生的局部變量，函式返回後即從棧上消失，**不持久化存儲**。即使攻擊者隨後取得了設備存取權限，也無法從記憶體中找到 AES 密鑰。

### 10.7 第五關：Ed25519 數位簽章驗證

Ed25519 是 Bernstein 等人設計的 Curve25519 上的數位簽章方案，提供 128 位安全強度。

**簽章範圍**：Ed25519 簽章覆蓋整個固件包的明文部分：

$$\text{hash} = \text{SHA-512}(\text{Header} \| \text{Salt} \| \text{IV} \| \text{Ciphertext})$$
$$\text{Verify}_{\text{PK}}(\text{signature}, \text{hash}) \stackrel{?}{=} \text{true}$$

**串流 SHA-512 計算**：由於固件包可能很大，SHA-512 採用串流更新：

```c
sha512_init(&sig_state);  // 在讀取 Header 之前初始化

// 每讀入一塊數據，餵入 SHA-512
fw_pkg_sha512_feed(&sig_state, data, len, pending, &pending_len);

// 所有數據讀完後，計算最終哈希
fw_pkg_sha512_finish(&sig_state, pending, pending_len, total_len, hash);

// Ed25519 驗證
edsign_verify(signature, ed25519_pubkey, hash, 64);
```

**簽章 vs HMAC 的互補關係**：

| 屬性 | HMAC-SHA256 | Ed25519 |
|------|-------------|---------|
| 密鑰類型 | 對稱密鑰 (DevKey) | 非對稱密鑰對 |
| 驗證範圍 | 僅 Header | Header + Salt + IV + Ciphertext |
| 驗證速度 | 快（微秒級） | 慢（毫秒級） |
| 安全保證 | 來源認證 | 來源認證 + 完整性 |
| 密鑰分發 | DevKey 預共享 | 公鑰硬編碼，私鑰離線保存 |

HMAC 使用對稱密鑰，意味著持有 DevKey 的人（打包工具和設備）都能生成有效 HMAC。Ed25519 使用非對稱密鑰，**只有持有私鑰的人才能簽章**，設備端僅持有公鑰用於驗證。這確保了即使 DevKey 洩露，攻擊者也無法偽造有效的 Ed25519 簽章。

**公鑰存儲**：

```c
static const uint8_t FW_PKG_ED25519_PUBLIC_KEY[32] = {
    0xc2, 0xe9, 0xbf, 0x62, 0x02, 0x92, ...
};
```

公鑰硬編碼在 Bootloader 固件中，作為 Ed25519 驗證的信任錨。對應的私鑰只存在於離線的固件簽章伺服器上，永不部署到設備端。

### 10.8 第六關：SHA-256 Flash 回讀校驗

前五關保證了「收到的包是可信的」和「解密出的固件是正確的」，但 Flash 寫入可能因硬體故障導致位翻轉。第六關通過回讀驗證寫入的正確性：

```c
// 解密完成後，回讀整個 Flash 區域並計算 SHA-256
sha256_init(&sha256_ctx);
for (offset = 0; offset < firmware_size; offset += chunk) {
    TARGET_READ(target, offset, process_buf, chunk, &bytes_read);
    sha256_update(&sha256_ctx, process_buf, bytes_read);
}
sha256_final(&sha256_ctx, computed_sha256);

// 與固件末尾內嵌的 SHA-256 比對
if (memcmp(computed_sha256, stored_sha256, 32) == 0)
    printf("  Result: MATCH\r\n");
else
    printf("  Result: MISMATCH!\r\n");
```

這一關提供 **端到端完整性保證**：從打包工具的 SHA-256 嵌入，到寫入 Flash，再到回讀驗證，構成了完整的數據完整性閉環。

### 10.9 A/B 分區作為安全網

信任鏈的終點不是「固件寫入成功」，而是「固件運行正常」。A/B 分區提供了運行時驗證的安全網：

```
新固件寫入非活動分區
  ├── 狀態設為 TESTING
  ├── boot_attempts = 0
  │
  ▼
啟動新固件
  ├── 運行正常 → App 調用 ab_partition_mark_slot_confirmed()
  │                 狀態轉為 CONFIRMED
  │
  └── 運行異常
      ├── boot_attempts 遞增
      ├── boot_attempts >= 3 → 自動回滾到舊分區
      └── SP 驗證失敗 → 立即回滾
```

`ab_partition_validate_slot()` 的雙重檢查：

```c
ab_err_t ab_partition_validate_slot(ab_slot_t slot) {
    uint32_t sp = *(__IO uint32_t *)addr;
    if (!ab_is_valid_sp(sp))           // 1. 棧指針必須在 SRAM 範圍內
        return AB_ERR_SLOT_INVALID_FW;

    uint32_t reset_handler = *(__IO uint32_t *)(addr + 4);
    if (reset_handler < addr ||        // 2. ResetHandler 必須落在本分區地址範圍內
        reset_handler > end_addr)
        return AB_ERR_SLOT_INVALID_FW;

    return AB_OK;
}
```

### 10.10 信任鏈的形式化總結

整個信任鏈可以表示為一個驗證函式的鏈式組合，每個函式的成功是下一個函式執行的前提：

$$V_{\text{total}} = V_{\text{HMAC}} \circ V_{\text{compat}} \circ V_{\text{rollback}} \circ V_{\text{HKDF}} \circ V_{\text{AES}} \circ V_{\text{Ed25519}} \circ V_{\text{SHA256}} \circ V_{\text{A/B}}$$

其中每個 $V_i$ 的輸出為 $\{\text{PASS}, \text{FAIL}\}$，且：

$$V_{\text{total}} = \text{PASS} \iff \forall i, V_i = \text{PASS}$$

| 驗證環節 | 安全屬性 | 攻擊防護 | 信任根依賴 |
|----------|----------|----------|------------|
| HMAC-SHA256 | 來源認證 | 偽造 Header | DevKey |
| 硬體相容 | 正確性 | 錯誤硬體刷寫 | 無（明文比對） |
| 安全計數器 | 版本單調性 | 回滾攻擊 | stored_counter |
| HKDF | 設備綁定 | 跨設備解密 | DevKey + UID |
| AES-256 | 機密性 | 固件竊取 | HKDF 派生密鑰 |
| Ed25519 | 完整性 + 認證 | 內容篡改 | Ed25519 公鑰 |
| SHA-256 回讀 | 寫入完整性 | Flash 位翻轉 | 無（回讀比對） |
| A/B 分區 | 可用性 | 變磚攻擊 | SP + ResetHandler |

---

## 十一、源碼深入剖析

### 11.1 main.c：啟動決策主入口

`main()` 是整個 Bootloader 的決策中心，核心邏輯如下：

```c
int main(void) {
    // ===== 軟復位快速跳轉（HAL_Init 之前） =====
    if (update_flag == JUMP_FLAG_MAGIC) {
        update_flag = 0;
        __DSB(); __ISB();
        ab_slot_t slot = ab_partition_get_active_slot_from_flash();
        bootloader_execute_jump();  // 直接跳轉，最快啟動
    }

    // ===== 標準初始化 =====
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_UART4_Init();
    MX_SPI1_Init();
    // ... 其他外設初始化
    platform_config_init();       // 註冊所有 Transport 驅動
    ab_partition_init();          // 讀取 A/B 元數據

    // ===== A/B 分區啟動檢查 =====
    ab_slot_t active = ab_partition_get_active_slot();
    ab_slot_state_t state = g_ab_metadata.slots[active].state;

    if (state == AB_SLOT_TESTING) {
        if (!ab_partition_validate_slot(active)) {
            ab_partition_rollback();        // 固件無效，回滾
        } else if (g_ab_metadata.slots[active].boot_attempts >= AB_MAX_BOOT_RETRIES) {
            ab_partition_rollback();        // 超過3次，回滾
        } else {
            ab_partition_increment_boot_attempts(active);  // 遞增嘗試計數
        }
    }

    // ===== SD卡掛載 =====
    if (SD_Detect() == SD_PRESENT) {
        f_mount(&SDFatFS, SDPath, 1);
    }

    // ===== 菜單入口判斷 =====
    Serial_PutString("\r\n按 'M' 進入 Bootloader 菜單 (1.5s 超時)...\r\n");
    if (WaitForSerialCommand(1500) == 'M') {
        Main_Menu();             // 進入交互式菜單
    } else if (update_flag == UPDATE_FLAG_MAGIC) {
        Main_Menu();             // App 請求更新
    } else {
        bootloader_request_jump(active);  // 自動跳轉 App
    }
}
```

> [!NOTE]
> 軟復位快速跳轉放在 `HAL_Init()` **之前**執行，這意味著此時所有外設都未初始化，但 `bootloader_execute_jump()` 只操作內核寄存器（MSP、VTOR），不依賴任何外設，因此是安全的。這種設計將 App 啟動延遲降到了最低。

### 11.2 bootloader_core.c：下載引擎

`bootloader_download()` 是普通下載的核心，通過 Transport 抽象層實現源到目標的數據搬運：

```c
int bootloader_download(platform_transport_base_t *source,
                        platform_transport_base_t *target,
                        const char *path) {
    uint8_t buf[DOWNLOAD_BUF_SIZE];  // 4096 字節緩衝區
    uint32_t total_size = 0;
    uint32_t bytes_read = 0;
    uint32_t offset = 0;

    // 1. 打開源傳輸
    int ret = TRANSPORT_SOURCE_OPEN(source, path, &total_size);
    if (ret != TRANSPORT_STATUS_OK) return BL_ERR_OPEN_SRC;

    // 2. 打開目標傳輸
    ret = TRANSPORT_TARGET_OPEN(target, NULL, total_size);
    if (ret != TRANSPORT_STATUS_OK) {
        TRANSPORT_SOURCE_CLOSE(source);
        return BL_ERR_OPEN_DST;
    }

    // 3. 循環讀取 → 寫入
    do {
        ret = TRANSPORT_SOURCE_READ(source, buf, sizeof(buf), &bytes_read);
        if (ret != TRANSPORT_STATUS_OK) break;

        if (bytes_read > 0) {
            ret = TRANSPORT_TARGET_WRITE(target, offset, buf, bytes_read);
            if (ret != TRANSPORT_STATUS_OK) break;
            offset += bytes_read;
        }
    } while (bytes_read > 0);

    // 4. 關閉傳輸
    TRANSPORT_TARGET_CLOSE(target);
    TRANSPORT_SOURCE_CLOSE(source);

    return (ret == TRANSPORT_STATUS_OK) ? BL_OK : transport_status_to_bootloader(ret);
}
```

`bootloader_secure_download()` 則在普通下載的基礎上，增加了完整的安全驗證鏈路：

```c
int bootloader_secure_download(platform_transport_base_t *source,
                                platform_transport_base_t *target,
                                const char *path,
                                const fw_pkg_verify_config_t *config,
                                fw_pkg_ctx_t *result) {
    // 調用固件包處理引擎，完成：
    // HMAC驗證 → 防回滾 → HKDF密鑰派生 → AES解密 → Ed25519簽名驗證
    int ret = fw_pkg_process_ex(source, target, path, config, result);
    if (ret != FW_PKG_OK) return fw_pkg_err_to_bootloader(ret);
    return BL_OK;
}
```

### 11.3 firmware_package.c：安全固件包處理引擎

這是安全體系最核心的文件（1107 行），`fw_pkg_process_ex()` 的完整流程：

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

    // ===== 1. 讀取並解析 Header =====
    SOURCE_READ(source, &header, sizeof(header));
    fw_pkg_parse_header(&header);  // 驗證 Magic + Version

    // ===== 2. HMAC-SHA256 驗證 Header =====
    fw_pkg_verify_header_hmac(&header, config->devkey);

    // ===== 3. 硬件兼容性檢查 =====
    if (header.hardware_compat != config->hardware_id)
        return FW_PKG_ERR_HW_INCOMPAT;

    // ===== 4. 防回滾檢查 =====
    fw_pkg_check_rollback(header.security_counter, config->min_security_counter);

    // ===== 5. 讀取 DynamicSalt + IV =====
    SOURCE_READ(source, salt, SALT_SIZE);
    SOURCE_READ(source, iv, IV_SIZE);

    // ===== 6. HKDF 密鑰派生 =====
    uint8_t aes_key[32];
    fw_pkg_derive_aes_key(salt, config->devkey, config->uid, aes_key);

    // ===== 7. 初始化 AES 解密 =====
    fw_pkg_decrypt_init(&aes_ctx, header.encryption_algo, aes_key, iv);

    // ===== 8. 打開目標，循環解密寫入 =====
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

    // ===== 9. 讀取並驗證 Ed25519 簽名 =====
    SOURCE_READ(source, signature, SIGNATURE_SIZE);
    fw_pkg_verify_signature(&header, salt, iv, ciphertext,
                            ciphertext_size, signature, config->ed25519_pubkey);

    // ===== 10. 回讀 Flash SHA-256 校驗 =====
    sha256_init(&sha256_ctx);
    for (offset = 0; offset < firmware_size; offset += sizeof(buf)) {
        TARGET_READ(target, offset, buf, chunk);
        sha256_update(&sha256_ctx, buf, chunk);
    }
    sha256_final(&sha256_ctx, computed_hash);
    // 與固件末尾內嵌的 SHA-256 比對

    // ===== 11. 填充結果上下文 =====
    ctx->fw_version = header.firmware_version;
    ctx->security_counter = header.security_counter;
    memcpy(ctx->sha256, computed_hash, 32);

    return FW_PKG_OK;
}
```

> [!IMPORTANT]
> HKDF 密鑰派生是設備綁定加密的關鍵。`fw_pkg_derive_aes_key()` 的三個輸入中，`DynamicSalt` 每包隨機生成，`DevKey` 存儲在 OTP 中每台設備不同，`UID` 是芯片出廠唯一ID。三者組合確保：即使固件包被截獲，也無法在其他設備上解密。

### 11.4 ab_partition.c：A/B 分區管理

元數據的追加寫入策略是 A/B 分區管理的精髓：

```c
// 元數據實例對齊：128 字節對齊，最多 512 個實例
#define METADATA_INSTANCE_ALIGN  128
#define METADATA_MAX_INSTANCES   (METADATA_SIZE / METADATA_INSTANCE_ALIGN)

static int ab_write_metadata_raw(const ab_metadata_t *meta) {
    uint32_t addr = METADATA_START_ADDR + g_ab_current_instance * METADATA_INSTANCE_ALIGN;

    // 檢查能否直接寫入（Flash 只允許 1→0）
    if (can_write_at(addr, meta, sizeof(*meta))) {
        ab_flash_write_words(addr, (const uint32_t *)meta,
                             sizeof(*meta) / 4);
        g_ab_current_instance++;
    } else {
        // 無法追加寫入，執行壓縮
        ab_compact_and_write(meta);
    }
}
```

回滾邏輯確保設備始終可啟動：

```c
int ab_partition_rollback(void) {
    ab_slot_t current = g_ab_metadata.active_slot;
    ab_slot_t fallback = (current == AB_SLOT_A) ? AB_SLOT_B : AB_SLOT_A;

    // 標記當前槽位為無效
    g_ab_metadata.slots[current].state = AB_SLOT_INVALID;

    // 驗證回退槽位有有效固件
    if (!ab_partition_validate_slot(fallback)) {
        return AB_ERR_NO_VALID_SLOT;  // 兩個槽位都無效！
    }

    // 切換到回退槽位
    g_ab_metadata.active_slot = fallback;
    g_ab_metadata.slots[fallback].state = AB_SLOT_CONFIRMED;
    g_ab_metadata.slots[fallback].boot_attempts = 0;
    ab_partition_metadata_flush();

    return AB_OK;
}
```

> [!WARNING]
> 如果兩個槽位都無效（`ab_partition_rollback` 返回 `AB_ERR_NO_VALID_SLOT`），設備將無法自動啟動，只能停留在 Bootloader 菜單等待手動刷入固件。這是 A/B 分區最後的防線——設備不會變磚，但需要人工干預。

### 11.5 service_hpatch.c：差分升級實現

差分升級使用 HPatch Lite + tuz 壓縮，內存佔用可控：

```c
// 靜態分配的緩衝區（編譯期確定）
static uint8_t tuz_dict_and_cache[HPATCH_DICT_SIZE + HPATCH_CACHE_SIZE];  // 4KB + 16KB
static uint8_t stream_diff_buf[HPATCH_STREAM_BUF_SIZE];                   // 8KB
static uint8_t patch_temp_buf[HPATCH_CACHE_SIZE];                         // 16KB

int hpatch_upgrade(hpatch_config_t *config) {
    // 1. 打開差分文件、舊固件、創建輸出文件
    fs_instance->open(&diff_file, config->diff_path, "rb");
    fs_instance->open(&old_file, config->old_path, "rb");
    fs_instance->open(&out_file, config->out_path, "wb");

    // 2. 解析差分文件頭
    hpatch_lite_open(&listener, &new_size, &compress_type);

    // 3. 若使用 tuz 壓縮，初始化解壓流
    if (compress_type == hpi_compress_tuz) {
        tuz_TStream_open(&tuz_stream_obj, tuz_dict_and_cache,
                         dict_size, tuz_fs_read_code);
        // 替換讀取回調為流式解壓
        listener.read_diff = stream_read_diff;
    }

    // 4. 執行差分補丁應用
    hpatch_lite_patch(&listener, new_size, patch_temp_buf);

    // 5. 清理
    fs_instance->close(&diff_file);
    fs_instance->close(&old_file);
    fs_instance->close(&out_file);
}
```

> [!TIP]
> 差分升級的典型效果：假設舊固件 384KB，新固件 386KB，差分文件可能只有 5-20KB。在帶寬受限的場景（如 NB-IoT、LoRa）下，差分升級能顯著減少傳輸時間和流量費用。

### 11.6 esp8266_ota_api.c：WiFi OTA 封裝

WiFi OTA 是對 OneNET 物聯網平台的薄封裝：

```c
static onenet_ota_ctx_t g_ota_ctx;

void esp8266_ota_init(void) {
    // 注入三個依賴：WiFi模塊、RTC（用於MQTT認證時間戳）、MQTT客戶端
    onenet_ota_ctx_init(&g_ota_ctx, &g_esp8266_wifi.base,
                        &g_rtc.base, &g_esp8266_mqtt.base);
}

int esp8266_ota_download(void) {
    onenet_ota_process_upgrade(&g_ota_ctx);  // MQTT接收 → HTTP下載 → 寫入目標
    return 1;
}
```

支持三種下載目標：

```c
void esp8266_ota_set_target_internal_flash(void);  // 直接寫入內部 Flash
void esp8266_ota_set_target_sd_card(void);          // 保存到 SD 卡
void esp8266_ota_set_target_spi_flash(void);         // 保存到 SPI Flash
```

### 11.7 整體分層架構

| 層級       | 模塊                                           | 說明                                   |
| ---------- | ---------------------------------------------- | -------------------------------------- |
| **應用層** | Menu                                           | 交互式菜單，4600+行                    |
| **核心層** | Bootloader Core                                | 下載/跳轉/安全下載                     |
|            | ├ firmware_package                             | 包解析/HMAC/AES/Ed25519                |
|            | └ ab_partition                                 | A/B分區/回滾/元數據管理                |
| **服務層** | Service Layer                                  | 功能服務                               |
|            | ├ service_hpatch                               | 差分升級                               |
|            | ├ esp8266_ota_api                              | WiFi OTA (OneNET)                      |
|            | └ service_aes_decrypt / service_ed25519_verify | 加解密/簽名驗證                        |
| **抽象層** | Platform Transport                             | 傳輸抽象                               |
|            | ├ Source                                       | FatFS / LittleFS / Ymodem / HTTP       |
|            | └ Target                                       | Internal Flash / FatFS / LittleFS      |
| **硬件層** | HAL                                            | STM32F407: Flash/RNG/SPI/UART/SDIO/RTC |

---

## 十二、總結

本項目是一個功能完備的工業級安全 Bootloader，核心亮點：

1. **A/B 雙分區**：無縫升級 + 自動回滾，設備永不變磚
2. **完整安全體系**：HMAC 認證 + AES-256 加密 + Ed25519 簽名 + 防回滾 + 設備綁定密鑰
3. **地址重定位 + 雙 Target 編譯**：源碼只需一份，簡化構建流程
4. **Transport 抽象**：下載邏輯與存儲介質完全解耦，易於擴展
5. **多渠道升級**：UART / SD卡 / SPI Flash / WiFi OTA，覆蓋各種場景
6. **差分升級**：HPatch + tuz 減少傳輸數據量
7. **交互式菜單**：功能豐富，便於調試和生產使用

---

## 參考資料

- [STM32F4xx Reference Manual](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [Ed25519: High-speed high-security signatures](https://ed25519.cr.yp.to/)
- [HKDF (RFC 5869)](https://tools.ietf.org/html/rfc5869)
- [HPatchLite - 差分補丁庫](https://github.com/sisong/HPatchLite)
- [LittleFS - 嵌入式文件系統](https://github.com/littlefs-project/littlefs)
