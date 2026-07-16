---
title: "STM32F407 Secure Bootloader Design: A/B Dual Partition, Encryption/Signing, and Multi-Channel Upgrade"
date: 2026-06-20
description: "A complete industrial-grade secure Bootloader design based on the STM32F407VGT6, covering core technologies such as A/B dual-partition seamless upgrade, AES-256 encryption + Ed25519 signature secure firmware packages, address relocation, differential upgrade, and multi-channel download (UART/SD card/SPI Flash/WiFi OTA)"
image: STM32F407安全Bootloader设计.png
categories:
  - "Embedded"
  - "Bootloader"
tags:
  - "STM32"
  - "Bootloader"
  - "IAP"
  - "A/B Partition"
  - "Firmware Encryption"
  - "Ed25519"
  - "OTA"
math: true
---

## Preface

In the IoT and embedded device domain, firmware upgrade (OTA) is an indispensable part of the product lifecycle. However, issues during the upgrade process such as **power-loss crashes, firmware tampering, and version rollback** often give developers headaches. This article details a secure Bootloader project based on the STM32F407VGT6, which systematically solves these pain points through mechanisms like **A/B dual-partition**, **secure firmware packages**, and **address relocation**.

---

## 1. What Problem Does It Solve?

| Problem                                          | Traditional Solution                                  | This Project's Solution                                                   |
| ------------------------------------------------ | ----------------------------------------------------- | ------------------------------------------------------------------------- |
| Power loss during upgrade bricks device          | Single partition direct overwrite, dies on power loss | A/B dual partition, auto-rollback on upgrade failure                      |
| Firmware tampered or forged                      | No verification or only CRC                           | Ed25519 signature + HMAC-SHA256 authentication                            |
| Firmware stolen and reverse-engineered           | Plaintext transmission                                | AES-256-CBC/CTR encryption + device-bound key                             |
| Old firmware version downgrade attack            | No protection                                         | Security counter anti-rollback                                            |
| Maintaining two build configs for two partitions | Maintaining two projects                              | Auto address relocation + dual Target compilation, only one source needed |
| Upgrade package too large, wastes bandwidth      | Full upgrade                                          | HPatch + tuz differential upgrade                                         |

---

## 2. Flash Memory Layout

The STM32F407VGT6 has 1MB of internal Flash, partitioned as follows:

| Start Address | Region         | Size   | Flash Sector        |
| ------------- | -------------- | ------ | ------------------- |
| `0x0800_0000` | Bootloader     | 128 KB | Sector 0-4          |
| `0x0802_0000` | Slot A         | 384 KB | Sector 5-7          |
| `0x0808_0000` | Slot B         | 384 KB | Sector 8-10         |
| `0x080E_0000` | Download Cache | 64 KB  | Sector 11 (partial) |
| `0x080F_0000` | Metadata       | 64 KB  | Sector 11 (partial) |

> [!TIP]
> Why does the Bootloader occupy 128KB? Because this project is currently a test demo, without slimming optimization — in practice, a streamlined Bootloader typically only needs 32-48KB. The current 128KB includes the complete menu system, crypto library (AES/Ed25519/HKDF), HPatch differential, ESP8266 WiFi/MQTT, and all other functional modules. In production, by trimming unnecessary features, the Bootloader can be reduced to a reasonable size, freeing up more Flash space for the App.

### Linker Script (Scatter File)

```
LR_IROM1 0x08000000 0x00020000  {    ; Bootloader 128KB
  ER_IROM1 0x08000000 0x00020000  {
   *.o (RESET, +First)
   *(InRoot$$Sections)
   .ANY (+RO)
   .ANY (+XO)
  }
  RW_IRAM1 0x20000000 0x0001BFFC  {  ; Main RAM
   .ANY (+RW +ZI)
  }
  RW_IRAM_NOINIT 0x2001BFFC UNINIT 0x00000004  {
   *(.bss.NoInit)                    ; Bootloader/App communication flag
  }
}
```

**Key design**: The `NoInit` region is located at `0x2001BFFC`, storing the `update_flag` variable. After a soft reset, this variable is not zeroed, used for state passing between the Bootloader and App.

> [!NOTE]
> The magic values of `update_flag`:
>
> - `0x5A5A5A5A` (`UPDATE_FLAG_MAGIC`) — App requests firmware update
> - `0xA5A5A5A5` (`JUMP_FLAG_MAGIC`) — Request jump to App (fast jump after soft reset)
> - `0x424F4F54` (`BOOT_CONFIRM_MAGIC`) — Boot confirmation mark

---

## 3. Boot Process and Jump Mechanism

### 3.1 Complete Boot Process

1. **Power-on/Reset**
2. Check `update_flag == JUMP_FLAG_MAGIC`? → Yes: clear flag → jump directly to App (fast boot)
3. `HAL_Init()` → `SystemClock_Config()` → peripheral initialization
4. A/B partition state check:
   - **TESTING** state → verify firmware validity
     - Invalid → rollback to other partition
     - Boot attempts exceed 3 → rollback
     - Valid → increment boot attempt counter
   - **CONFIRMED** state → normal boot
5. Wait for UART command (1.5s timeout)
   - Received 'M' → enter Bootloader menu
   - Timeout → continue
6. Check `update_flag == UPDATE_FLAG_MAGIC`? → Yes: enter Bootloader menu (App requests update)
7. Auto jump to active partition's App

### 3.2 Core Jump Code

The jump is implemented in two steps, ensuring clean hardware state via soft reset:

**Step 1: Request jump (set flag + soft reset)**

```c
void bootloader_request_jump(ab_slot_t slot) {
    uint32_t app_addr = bootloader_get_slot_addr(slot);
    if (app_addr == 0 || !bootloader_validate_sp(app_addr)) return;

    ab_partition_set_active_slot(slot);  // Set active partition
    update_flag = JUMP_FLAG_MAGIC;       // Set jump flag
    __DSB(); __ISB();
    NVIC_SystemReset();                  // Soft reset
}
```

**Step 2: Actual jump execution (executed after reset)**

```c
void bootloader_execute_jump(void) {
    uint32_t app_addr = bootloader_get_slot_addr(AB_SLOT_AUTO);
    uint32_t app_stack_ptr = *(__IO uint32_t *)app_addr;
    uint32_t app_reset_handler = *(__IO uint32_t *)(app_addr + 4);

    SysTick->CTRL = 0;           // Disable SysTick
    __set_PRIMASK(0);            // Enable interrupts
    SCB->VTOR = app_addr;        // Relocate interrupt vector table
    __DSB(); __ISB();
    __set_MSP(app_stack_ptr);    // Set main stack pointer
    ((pFunction)app_reset_handler)();  // Jump to App ResetHandler
}
```

**SP validation** ensures the stack pointer is within the valid SRAM range:

```c
bool bootloader_validate_sp(uint32_t app_addr) {
    uint32_t sp = *(__IO uint32_t *)app_addr;
    return (sp >= 0x20000000 && sp <= 0x2FFE0000);
}
```

> [!TIP]
> Why is the jump split into two steps? Direct jump is faster, but soft reset ensures all peripherals and interrupt states are cleanly reset. This project uses `request_jump` (soft reset) in the normal boot flow, while directly calling `execute_jump` for fast boot when `update_flag == JUMP_FLAG_MAGIC` — because at that point we just returned from App via soft reset, the hardware state is already clean.

---

## 4. A/B Dual-Partition System

### 4.1 Metadata Structure

Metadata is stored in the Metadata region at the end of Flash, using an **append-write** strategy to avoid frequent erasure:

```c
typedef struct __attribute__((packed)) {
    uint32_t fw_version;         // Firmware version
    uint32_t security_counter;   // Security counter (anti-rollback)
    uint32_t fw_size;            // Firmware size
    ab_slot_state_t state;       // Partition state
    uint8_t boot_attempts;       // Boot attempt count
    uint8_t sha256[32];          // Firmware SHA256
    uint8_t reserved[3];
} ab_slot_meta_t;

typedef struct __attribute__((packed)) {
    uint32_t magic;              // 0x41425441 ("ABTA")
    uint32_t version;            // Version = 2
    ab_slot_t active_slot;       // Active partition (A/B)
    ab_slot_meta_t slots[2];     // Metadata for both partitions
    uint32_t crc32;              // CRC32 checksum
} ab_metadata_t;
```

### 4.2 Partition State Machine

- **IDLE** → write new firmware → **TESTING**
- **TESTING** → App confirms → **CONFIRMED**
- **TESTING** → boot failure/3 retries exceeded → **ROLLBACK** → switch to other partition

State descriptions:

- **IDLE**: initial state, partition has no valid firmware
- **TESTING**: new firmware written and marked as testing, `boot_attempts` incremented on boot
- **CONFIRMED**: App confirms after running normally, marked as stable version
- **INVALID**: firmware verification failed

### 4.3 Append-Write Strategy

The 64KB Metadata region is divided into 128-byte-aligned instances (up to 512). Each time metadata is updated, it appends to the next instance position rather than erasing and rewriting. When instances are exhausted or bit-and mismatch occurs, **compaction** is performed: erase the entire sector and rewrite the latest data.

```c
// Append-write core logic
int ab_partition_write_metadata(const ab_metadata_t *meta) {
    // Find next free instance position
    uint32_t addr = find_next_free_instance();
    if (addr == 0) {
        // Instances exhausted, perform compaction
        return compact_metadata(meta);
    }
    // Append write
    return flash_write(addr, meta, sizeof(ab_metadata_t));
}
```

---

## 5. Secure Firmware Package System

### 5.1 Firmware Package Format

Custom `.iap.bin` format, integrating encryption, signature, and verification:

| Region      | Size     | Description                                     |
| ----------- | -------- | ----------------------------------------------- |
| Header      | 64 bytes | Magic, version, encryption/signature algo, etc. |
| DynamicSalt | 16 bytes | Random salt, different for each package         |
| IV          | 16 bytes | AES initialization vector                       |
| Ciphertext  | N bytes  | AES-256-CBC/CTR encrypted firmware              |
| Signature   | 64 bytes | Ed25519 digital signature                       |

### 5.2 Header Structure

```c
typedef struct __attribute__((packed)) {
    uint32_t magic;                  // 0x01504149
    uint8_t  header_version;         // 1
    uint8_t  firmware_major;         // Major version
    uint8_t  firmware_minor;         // Minor version
    uint8_t  firmware_patch;         // Patch version
    uint32_t total_payload_size;     // Salt + IV + Ciphertext total size
    uint8_t  image_type;             // APP=0x01, BOOTLOADER=0x02
    uint8_t  encryption_algo;        // NONE=0, AES256-CBC=1, ECB=2, CTR=3
    uint8_t  signature_algo;         // NONE=0, ED25519=1
    uint32_t hardware_compat;        // Hardware compatibility identifier
    uint32_t security_counter;       // Security counter (anti-rollback)
    uint32_t build_timestamp;        // Build timestamp
    uint8_t  reserved[5];
    uint8_t  header_checksum[32];    // HMAC-SHA256 checksum
} fw_pkg_header_t;
```

### 5.3 Complete Secure Download Flow

| Step | Operation                                                       |
| ---- | --------------------------------------------------------------- |
| 1    | Read Header → verify Magic + Header Version                     |
| 2    | HMAC-SHA256 verify Header (using device DevKey)                 |
| 3    | Hardware compatibility check                                    |
| 4    | Security counter anti-rollback check                            |
| 5    | Read DynamicSalt + IV                                           |
| 6    | HKDF key derivation: AES_Key = HKDF(Salt, DevKey, UID)          |
| 7    | Initialize AES-256 decryption context                           |
| 8    | Loop: read ciphertext (4KB) → AES decrypt → write to Flash      |
| 9    | Read Ed25519 signature                                          |
| 10   | SHA-512 streaming hash verification (Header+Salt+IV+Ciphertext) |
| 11   | Ed25519 signature verification                                  |
| 12   | Read back Flash → SHA256 integrity check                        |
| 13   | Update A/B metadata → set active partition (TESTING)            |

### 5.4 Key Derivation: Device-Bound Encryption

This is the core design of the security system. Each STM32 chip has a unique UID (12 bytes, address `0x1FFF7A10`), binding DevKey with UID via HKDF:

```text
AES_Key = HKDF(
    salt     = DynamicSalt,   // 16 bytes, randomly generated per firmware package
    IKM      = DevKey,        // 16 bytes, stored in OTP
    info     = UID,           // 12 bytes, STM32 unique ID
    key_len  = 32 bytes
)
```

**Effect**: Even with the same DevKey, different devices derive different AES keys. The firmware package can only be decrypted by the target device, **achieving device-bound encryption**.

### 5.5 Key Storage Scheme

| Key                      | Storage Location         | Characteristics                 |
| ------------------------ | ------------------------ | ------------------------------- |
| DevKey (16B)             | STM32 OTP region         | One-time write, immutable       |
| Ed25519 public key (32B) | Hardcoded in firmware    | Public                          |
| UID (12B)                | STM32 unique ID register | Factory-fixed                   |
| AES key (32B)            | Runtime HKDF derivation  | Not stored, discarded after use |

---

## 6. Address Relocation: One Firmware for Two Partitions

### 6.1 The Problem

Slot A starts at `0x08020000`, Slot B starts at `0x08080000`. Firmware compiled for Slot A has its interrupt vector table and absolute address jumps pointing to Slot A's address range, and won't run if directly written to Slot B.

### 6.2 Solution

When writing to Slot B, scan each 32-bit word, and if the value falls within Slot A's address range, automatically add the offset:

```c
#define RELOCATE_OFFSET  0x00060000  // Slot B - Slot A

static uint32_t internal_flash_relocate_word(uint32_t word) {
    if (word >= SLOT_A_START_ADDR && word < SLOT_A_END_ADDR) {
        return word + RELOCATE_OFFSET;
    }
    return word;
}
```

**Note**: Address relocation can handle most absolute address corrections (like interrupt vector tables, function pointers, etc.), but is not a panacea. In real projects, you still need to configure two Targets in Keil, controlling the build address through their respective linker scripts (Scatter File) to ensure correct addressing of absolute and relative addresses. The significance of the relocation mechanism is: **the App side only needs to maintain one source**, compiling Slot A / Slot B firmware versions by switching Targets, without doing address adaptation at the code level.

> [!WARNING]
> Limitations of address relocation: it can only identify and correct **absolute address literals falling within Slot A's address range**. For instructions addressed via PC-relative offset (Thumb-2 BL/BLX, etc.), since the offset is independent of the base address, relocation neither needs to nor will modify them. But if the code has addresses computed at runtime (like function pointer arrays, vtables, etc.), relocation may not cover all cases, so dual-Target compilation is still needed to ensure correctness.

---

## 7. Platform Abstraction Layer: Transport Architecture

### 7.1 Unified Interface Design

Through `platform_transport_base_t`, abstract Source (read) and Target (write) roles:

```c
// Source interface - read from somewhere
typedef struct {
    int16_t (*open)(const void* ctx, const char* path, uint32_t* total_size);
    int16_t (*read)(const void* ctx, uint8_t* buf, uint32_t size, uint32_t* bytes_read);
    int16_t (*close)(const void* ctx);
} platform_transport_source_ops_t;

// Target interface - write to somewhere
typedef struct {
    int16_t (*open)(const void* ctx, const char* path, uint32_t total_size);
    int16_t (*write)(const void* ctx, uint32_t offset, const uint8_t* data, uint32_t len);
    int16_t (*read)(const void* ctx, uint32_t offset, uint8_t* buf, uint32_t size, uint32_t* bytes_read);
    int16_t (*close)(const void* ctx);
} platform_transport_target_ops_t;
```

### 7.2 Implemented Transports

| Transport                | Source | Target | Description                             |
| ------------------------ | :----: | :----: | --------------------------------------- |
| `g_slot_a_flash`         |   -    |  Yes   | Slot A internal Flash                   |
| `g_slot_b_flash`         |   -    |  Yes   | Slot B internal Flash (with relocation) |
| `g_download_cache_flash` |   -    |  Yes   | Download cache region                   |
| `g_fatfs_transport`      |  Yes   |  Yes   | SD card FatFS                           |
| `g_lfs_transport`        |  Yes   |  Yes   | SPI Flash LittleFS                      |

### 7.3 Flexible Composition

The Transport architecture fully decouples download logic from storage media:

```c
// Download from SD card to Slot A
bootloader_download(g_fatfs_transport, "firmware.bin",
                    g_slot_a_flash, NULL);

// Download from SPI Flash to Slot B (auto-relocation)
bootloader_download(g_lfs_transport, "firmware.bin",
                    g_slot_b_flash, NULL);
```

---

## 8. Multi-Channel Firmware Upgrade

### 8.1 UART YMODEM

Firmware download via serial port, supporting 128-byte and 1K-byte packet sizes. `Ymodem_Receive_To_Slot()` supports direct download to a specified A/B partition.

### 8.2 SD Card Upgrade

Read `.bin` / `.iap.bin` / `.hdiff` files from SD card, read via FatFS Transport then write to internal Flash.

### 8.3 SPI Flash Upgrade

Read firmware files from W25Q128 SPI Flash (managed via LittleFS file system), suitable for products without an SD card slot.

### 8.4 WiFi OTA Upgrade

Connect to OneNET IoT platform via ESP8266 WiFi module, supporting MQTT protocol to receive upgrade notifications, HTTP download firmware packages, progress reporting, and result notifications.

### 8.5 Differential Upgrade (HPatch + tuz)

Use HPatch Lite + tuz compression algorithm for differential upgrade, dramatically reducing data transfer volume:

```c
// Differential upgrade core flow
int service_hpatch_apply(const char *patch_path,
                         platform_transport_base_t *patch_source,
                         ab_slot_t target_slot) {
    // 1. Read old firmware (current active partition)
    // 2. Read differential file
    // 3. HPatch synthesize new firmware → write to target partition
    // Buffer: 16KB cache + 4KB dict + 8KB stream
}
```

---

## 9. Interactive Menu System

The Bootloader provides a feature-rich interactive menu, accessed via UART4. All feature items are compile-controlled via macros, configurable in `menu.h` or via build options.

### 9.1 Macro Control Switches

| Macro Name                     | Default      | Description                    |
| ------------------------------ | ------------ | ------------------------------ |
| `MENU_ENABLE_DOWNLOAD`         | 1 (enabled)  | Firmware download              |
| `MENU_ENABLE_UPLOAD`           | 0 (disabled) | Firmware upload                |
| `MENU_ENABLE_SPI_FLASH_STORE`  | 1 (enabled)  | SPI Flash storage              |
| `MENU_ENABLE_EXECUTE_APP`      | 1 (enabled)  | Execute application            |
| `MENU_ENABLE_FLASH_PROTECTION` | 0 (disabled) | Flash write protection toggle  |
| `MENU_ENABLE_AES_DECRYPT`      | 0 (disabled) | AES decryption                 |
| `MENU_ENABLE_HPATCH`           | 1 (enabled)  | HPatch differential upgrade    |
| `MENU_ENABLE_UART_PASSTHROUGH` | 0 (disabled) | UART serial passthrough        |
| `MENU_ENABLE_ESP8266_WIFI`     | 0 (disabled) | ESP8266 WiFi & OTA             |
| `MENU_ENABLE_ED25519_VERIFY`   | 0 (disabled) | Ed25519 signature verification |
| `MENU_ENABLE_RNG_DEVKEY`       | 0 (disabled) | RNG device key generation      |
| `MENU_ENABLE_FIRMWARE_PACKAGE` | 1 (enabled)  | Firmware package parsing       |

> [!NOTE]
> All macros use the `#ifndef` guard pattern, and can be overridden externally via compiler option `-DMENU_ENABLE_XXX=1`. Only `[F] A/B Partition Management` is unconditionally compiled and always present.

### 9.2 Complete Menu Tree

| Shortcut | Menu                                    | Submenu                                          | Control Macro                  | Description                      |
| -------- | --------------------------------------- | ------------------------------------------------ | ------------------------------ | -------------------------------- |
| **[1]**  | Download image                          |                                                  | `MENU_ENABLE_DOWNLOAD`         | Firmware download                |
|          |                                         | [1] Download via Serial (Ymodem)                 |                                | Serial download                  |
|          |                                         | [2] Download from SD card (FATFS)                |                                | SD card download                 |
|          |                                         | [3] Download from SPI Flash (LittleFS)           |                                | SPI Flash download               |
| **[2]**  | Upload image from internal Flash        | -                                                | `MENU_ENABLE_UPLOAD`           | Ymodem upload                    |
| **[3]**  | Store image to SPI-Flash LFS            |                                                  | `MENU_ENABLE_SPI_FLASH_STORE`  | SPI Flash storage                |
|          |                                         | [1] Store image from TF card                     |                                | Transfer from SD card            |
|          |                                         | [2] Store image from Flash                       |                                | Transfer from internal Flash     |
|          |                                         | [3] Show stored images                           |                                | View stored files                |
|          |                                         | [4] Delete stored image                          |                                | Delete file                      |
|          |                                         | [5] Delete entire file system                    |                                | Delete entire file system        |
| **[4]**  | Execute the loaded application          | -                                                | `MENU_ENABLE_EXECUTE_APP`      | Jump to App                      |
| **[5]**  | Toggle Flash write protection           | -                                                | `MENU_ENABLE_FLASH_PROTECTION` | Flash write protection toggle    |
| **[6]**  | Decrypt and download encrypted firmware |                                                  | `MENU_ENABLE_AES_DECRYPT`      | AES decrypt download             |
|          |                                         | [1] Decrypt from SD card and download to Flash   |                                | SD card decrypt download         |
|          |                                         | [2] Decrypt from SPI Flash and download to Flash |                                | SPI Flash decrypt download       |
| **[7]**  | Decrypt .bin.aes file on SD card        | -                                                | `MENU_ENABLE_AES_DECRYPT`      | SD card AES decryption           |
| **[8]**  | HPatch differential upgrade             |                                                  | `MENU_ENABLE_HPATCH`           | Differential upgrade             |
|          |                                         | [1] HPatch upgrade from SD card                  |                                | SD card differential upgrade     |
|          |                                         | [2] HPatch upgrade from SPI Flash                |                                | SPI Flash differential upgrade   |
| **[9]**  | UART4 <-> USART1 Passthrough            | -                                                | `MENU_ENABLE_UART_PASSTHROUGH` | Serial passthrough               |
| **[A]**  | ESP8266 WiFi & OTA Test                 |                                                  | `MENU_ENABLE_ESP8266_WIFI`     | WiFi & OTA                       |
|          |                                         | [1] WiFi Init & Connect AP                       |                                | WiFi init connect                |
|          |                                         | [2] AT Command Test                              |                                | AT command test                  |
|          |                                         | [3] TCP Connect Test                             |                                | TCP connection test              |
|          |                                         | [4] Enter Transparent Mode                       |                                | Enter transparent mode           |
|          |                                         | [5] Exit Transparent Mode                        |                                | Exit transparent mode            |
|          |                                         | [6] Set OTA Target: Internal Flash               |                                | OTA target: Internal Flash       |
|          |                                         | [7] Set OTA Target: SD Card (FATFS)              |                                | OTA target: SD card              |
|          |                                         | [8] Set OTA Target: SPI Flash (LFS)              |                                | OTA target: SPI Flash            |
|          |                                         | [9] OneNET OTA Download                          |                                | OneNET OTA download              |
|          |                                         | [A] Show Current Time                            |                                | Show current time                |
|          |                                         | [B] MQTT Test Menu →                             |                                | MQTT submenu                     |
|          |                                         | [1] Check MQTT Connection Status                 |                                | Check MQTT connection            |
|          |                                         | [2] Configure MQTT User                          |                                | Configure MQTT user              |
|          |                                         | [3] Connect to MQTT Server                       |                                | Connect to MQTT server           |
|          |                                         | [4] Subscribe Property Topics                    |                                | Subscribe property topics        |
|          |                                         | [5] Publish Property                             |                                | Publish property                 |
|          |                                         | [6] Listen & Auto Reply Property Set             |                                | Listen and auto reply            |
|          |                                         | [7] Disconnect MQTT                              |                                | Disconnect MQTT                  |
|          |                                         | [8] Sync Time (SNTP)                             |                                | SNTP time sync                   |
|          |                                         | [9] Publish RTC Time (1s interval)               |                                | Periodic RTC time publish        |
| **[B]**  | Ed25519 Signature Verify                |                                                  | `MENU_ENABLE_ED25519_VERIFY`   | Signature verification           |
|          |                                         | [1] Verify firmware on SD card                   |                                | SD card signature verification   |
|          |                                         | [2] Verify firmware on SPI Flash                 |                                | SPI Flash signature verification |
|          |                                         | [3] Buffer verify test                           |                                | Buffer verification test         |
| **[C]**  | Generate Device Key (RNG)               | -                                                | `MENU_ENABLE_RNG_DEVKEY`       | RNG generate device key          |
| **[D]**  | Write DevKey to OTP                     | -                                                | `MENU_ENABLE_RNG_DEVKEY`       | Write OTP device key             |
| **[E]**  | Firmware Package Parse                  |                                                  | `MENU_ENABLE_FIRMWARE_PACKAGE` | Firmware package parsing         |
|          |                                         | [1] Parse package from SD card (FATFS)           |                                | SD card parse                    |
|          |                                         | [2] Parse package from SPI Flash (LFS)           |                                | SPI Flash parse                  |
|          |                                         | [3] Secure Download to A/B (SD Card)             |                                | Secure download to A/B partition |
|          |                                         | [4] Secure Download to A/B (SPI Flash)           |                                | Secure download to A/B partition |
| **[F]**  | A/B Partition Management                |                                                  | None (always enabled)          | A/B partition management         |
|          |                                         | [1] Show A/B Partition Status                    |                                | View partition status            |
|          |                                         | [2] Set Active Slot                              |                                | Set active partition             |
|          |                                         | [3] Confirm Current Slot                         |                                | Confirm current partition        |
|          |                                         | [4] Rollback                                     |                                | Rollback                         |
|          |                                         | [5] Jump to Slot                                 |                                | Jump to specified partition      |

---

## 10. Security Trust Chain: Anchor, Propagation, and Guarantees

### 10.1 Trust Chain Global Architecture

The core of the secure Bootloader is not any single cryptographic algorithm, but a trust chain that starts from a **hardware root of trust** and propagates layer by layer. The success of each verification gate is a prerequisite for the next gate to execute; failure at any gate aborts the entire process, and no firmware is written to Flash.

```
Root of Trust (RoT)
  ├── UID (0x1FFF7A10, factory-fixed, 12B)
  └── DevKey (OTP, one-time write, 16B)
       │
       ▼
Gate 1: HMAC-SHA256 Header Authentication
  "Was this package generated by someone holding DevKey?"
       │
       ▼
Gate 2: Hardware Compatibility + Security Counter Anti-Rollback
  "Was this package built for this hardware? Is the version not downgraded?"
       │
       ▼
Gate 3: HKDF Device-Bound Key Derivation
  AES_Key = HKDF(Salt, DevKey, UID)
  "Even if DevKey is leaked, only the target device can derive the correct AES key"
       │
       ▼
Gate 4: AES-256 Decryption
  "Only the target device can recover the plaintext firmware"
       │
       ▼
Gate 5: Ed25519 Digital Signature Verification
  SHA-512(Header‖Salt‖IV‖Ciphertext) → Ed25519 Verify
  "Firmware content was signed by the private key holder, not tampered with"
       │
       ▼
Gate 6: SHA-256 Flash Read-Back Verification
  "Data written to Flash is identical to the decryption output"
       │
       ▼
Trust Endpoint: A/B Partition State Transition (TESTING → CONFIRMED)
  "New firmware only becomes the official version after runtime validation"
```

### 10.2 Root of Trust: Hardware Anchor

The trust chain must have an unforgeorable starting point. The root of trust in this project consists of two hardware primitives:

**STM32 Unique ID (UID)**

- Address: `0x1FFF7A10`, 12 bytes (96 bits)
- Properties: Burned by ST at the factory for each chip, immutable, non-erasable
- Role: Serves as the `info` parameter for HKDF, binding key derivation to a specific chip

```c
#define STM32F4_UID_ADDR 0x1FFF7A10
// Read method
const uint8_t *uid = (const uint8_t *)STM32F4_UID_ADDR;
```

**Device Key (DevKey)**

- Storage location: STM32 OTP (One-Time Programmable) region
- Size: 16 bytes (128 bits)
- Properties: OTP region can only transition from 0 to 1; once written, it cannot be modified or erased
- Role: HMAC-SHA256 authentication key + HKDF input keying material (IKM)

> [!IMPORTANT]
> DevKey's OTP storage is the critical security property of the root of trust. The "one-time write" semantics of OTP mean that after DevKey is burned on the production line, even if an attacker gains JTAG access, they cannot overwrite or modify DevKey. This is fundamentally different from Flash storage — Flash can be erased and rewritten, but OTP cannot.

**Formal guarantee of the root of trust**:

$$\text{RoT} = \{\text{UID}_{\text{chip}}, \text{DevKey}_{\text{OTP}}\}$$

where $\text{UID}_{\text{chip}}$ satisfies unforgeability (factory-fixed), and $\text{DevKey}_{\text{OTP}}$ satisfies immutability (OTP semantics).

### 10.3 Gate 1: HMAC-SHA256 Header Authentication

**Purpose**: Before processing any content of the firmware package, first verify that the Header's source is trustworthy.

**Mechanism**:

```c
fw_pkg_err_t fw_pkg_verify_header_hmac(const fw_pkg_ctx_t *ctx,
                                       const fw_pkg_verify_config_t *config)
{
    uint8_t computed_hmac[32];
    const uint8_t *header_prefix = (const uint8_t *)&ctx->header;

    // HMAC-SHA256(DevKey, Header[0..31])  ← First 32 bytes of Header
    hmac_sha256(config->devkey, config->devkey_len,
                header_prefix, FW_PKG_HEADER_SIZE - FW_PKG_HMAC_SIZE,
                computed_hmac);

    // Compare with HMAC embedded in Header
    if (memcmp(computed_hmac, ctx->header.header_checksum, 32) != 0)
        return FW_PKG_ERR_HMAC;

    return FW_PKG_OK;
}
```

**Security properties**:

- **Authentication**: Only the packaging tool holding DevKey can generate a correct HMAC
- **Selectivity**: The HMAC covers only the Header, not the payload — this is by design, because payload integrity is guaranteed by the Ed25519 signature
- **Early rejection**: Header authentication is executed before reading Salt/IV/Ciphertext; invalid packages are rejected with minimal I/O

**Why HMAC instead of directly using Ed25519 to sign the Header?**

HMAC-SHA256 computation is far faster than Ed25519 signature verification (microseconds vs milliseconds). As the first line of defense, it can quickly reject packages in DoS scenarios with large volumes of invalid inputs. Ed25519 signature verification is placed last, performing integrity verification over the entire package (Header + Salt + IV + Ciphertext).

### 10.4 Gate 2: Anti-Rollback and Hardware Compatibility

**Security Counter (security_counter)**

Each firmware package's Header carries a monotonically increasing `security_counter`, and the device stores the current maximum confirmed counter value:

```c
fw_pkg_err_t fw_pkg_check_rollback(const fw_pkg_ctx_t *ctx,
                                   const fw_pkg_verify_config_t *config)
{
    if (ctx->header.security_counter < config->stored_security_counter)
    {
        // Rollback detected! Package counter < device stored counter
        return FW_PKG_ERR_ROLLBACK;
    }
    return FW_PKG_OK;
}
```

**Formal guarantee of anti-rollback**:

Let the device-stored counter be $c_{\text{stored}}$ and the firmware package counter be $c_{\text{pkg}}$, then:

$$c_{\text{pkg}} \geq c_{\text{stored}} \implies \text{allow upgrade}$$
$$c_{\text{pkg}} < c_{\text{stored}} \implies \text{reject (rollback attack)}$$

**Hardware compatibility**: The `hardware_compat` field ensures the firmware package matches the target hardware, preventing incompatible firmware from being written to the device and causing malfunctions.

### 10.5 Gate 3: HKDF Device-Bound Key Derivation

This is the most critical design of the entire security system — **device-bound encryption**. Even if an attacker intercepts both the firmware package and DevKey, without the target device's UID they cannot decrypt.

**HKDF (HMAC-based Extract-and-Expand Key Derivation Function, RFC 5869)**

```
AES_Key = HKDF(
    salt     = DynamicSalt,    // 16 bytes, randomly generated per firmware package
    IKM      = DevKey,         // 16 bytes, stored in OTP
    info     = UID,            // 12 bytes, STM32 unique ID
    key_len  = 32 bytes        // AES-256 key length
)
```

HKDF operates in two phases:

1. **Extract**: `PRK = HMAC-Hash(salt, IKM)` — extracts the entropy of DevKey into a pseudorandom key PRK
2. **Expand**: `OKM = HMAC-Hash(PRK, info ∥ 0x01)` — expands PRK into an output key of the specified length; info (i.e., UID) ensures different devices derive different keys

**Security analysis of device binding**:

| Attack Scenario | Attacker Possesses | Can Decrypt? | Reason |
|-----------------|--------------------|-------------|--------|
| Intercepted firmware package | Salt, IV, Ciphertext | No | Without DevKey and UID, cannot derive AES_Key |
| Intercepted firmware package + DevKey | Salt, IV, Ciphertext, DevKey | No | Without UID, HKDF(info) output differs |
| Same DevKey, different device | Salt, IV, Ciphertext, DevKey, UID_B | No | UID_B ≠ UID_A, derived AES_Key differs |
| Target device itself | Salt, IV, Ciphertext, DevKey, UID | **Yes** | All three elements present, correctly derives AES_Key |

**Role of DynamicSalt**: A 16-byte salt randomly generated per package ensures that even when pushing the same firmware to the same device, two AES key derivations produce different keys, eliminating key reuse risk.

### 10.6 Gate 4: AES-256 Decryption and Streaming

Three encryption modes are supported, selected based on the `encryption_algo` field in the Header:

| Mode | Constant | Characteristics |
|------|----------|-----------------|
| AES-256-CBC | `FW_PKG_ENC_AES256_CBC` | Most common, PKCS7 padding, requires IV |
| AES-256-CTR | `FW_PKG_ENC_AES256_CTR` | Streaming-friendly, no padding needed |
| AES-256-ECB | `FW_PKG_ENC_AES256_ECB` | Not recommended, for compatibility only |

**Streaming decrypt-write**: The firmware package may be far larger than available RAM, so a 4KB buffer is used in a loop:

```c
static uint8_t process_buf[4096] __attribute__((aligned(4)));

while (ciphertext_remaining > 0) {
    uint32_t to_read = MIN(ciphertext_remaining, sizeof(process_buf));
    SOURCE_READ(source, process_buf, to_read, &bytes_read);   // Read ciphertext
    fw_pkg_decrypt_payload(&ctx, process_buf, bytes_read,      // AES decrypt
                           is_final, &actual_len);
    TARGET_WRITE(target, flash_offset, process_buf, bytes_read); // Write Flash
    flash_offset += bytes_read;
    ciphertext_remaining -= bytes_read;
}
```

**Critical security property**: The AES key (`aes_key[32]`) is a local variable derived at runtime via HKDF; after the function returns, it disappears from the stack and is **never persisted**. Even if an attacker subsequently gains device access, they cannot find the AES key in memory.

### 10.7 Gate 5: Ed25519 Digital Signature Verification

Ed25519 is a digital signature scheme designed by Bernstein et al. on Curve25519, providing 128-bit security strength.

**Signature scope**: The Ed25519 signature covers the entire plaintext portion of the firmware package:

$$\text{hash} = \text{SHA-512}(\text{Header} \| \text{Salt} \| \text{IV} \| \text{Ciphertext})$$
$$\text{Verify}_{\text{PK}}(\text{signature}, \text{hash}) \stackrel{?}{=} \text{true}$$

**Streaming SHA-512 computation**: Since the firmware package may be large, SHA-512 uses streaming updates:

```c
sha512_init(&sig_state);  // Initialize before reading Header

// Feed each data chunk into SHA-512
fw_pkg_sha512_feed(&sig_state, data, len, pending, &pending_len);

// After all data is read, compute final hash
fw_pkg_sha512_finish(&sig_state, pending, pending_len, total_len, hash);

// Ed25519 verification
edsign_verify(signature, ed25519_pubkey, hash, 64);
```

**Complementary relationship between Signature and HMAC**:

| Property | HMAC-SHA256 | Ed25519 |
|----------|-------------|---------|
| Key type | Symmetric key (DevKey) | Asymmetric key pair |
| Verification scope | Header only | Header + Salt + IV + Ciphertext |
| Verification speed | Fast (microseconds) | Slow (milliseconds) |
| Security guarantee | Source authentication | Source authentication + integrity |
| Key distribution | DevKey pre-shared | Public key hardcoded, private key stored offline |

HMAC uses a symmetric key, meaning anyone holding DevKey (the packaging tool and the device) can generate a valid HMAC. Ed25519 uses an asymmetric key — **only the holder of the private key can sign** — while the device holds only the public key for verification. This ensures that even if DevKey is leaked, an attacker cannot forge a valid Ed25519 signature.

**Public key storage**:

```c
static const uint8_t FW_PKG_ED25519_PUBLIC_KEY[32] = {
    0xc2, 0xe9, 0xbf, 0x62, 0x02, 0x92, ...
};
```

The public key is hardcoded in the Bootloader firmware, serving as the trust anchor for Ed25519 verification. The corresponding private key exists only on the offline firmware signing server and is never deployed to the device.

### 10.8 Gate 6: SHA-256 Flash Read-Back Verification

The first five gates guarantee "the received package is trusted" and "the decrypted firmware is correct", but Flash writes may experience bit flips due to hardware faults. Gate 6 verifies write correctness through read-back:

```c
// After decryption, read back the entire Flash region and compute SHA-256
sha256_init(&sha256_ctx);
for (offset = 0; offset < firmware_size; offset += chunk) {
    TARGET_READ(target, offset, process_buf, chunk, &bytes_read);
    sha256_update(&sha256_ctx, process_buf, bytes_read);
}
sha256_final(&sha256_ctx, computed_sha256);

// Compare with SHA-256 embedded at firmware end
if (memcmp(computed_sha256, stored_sha256, 32) == 0)
    printf("  Result: MATCH\r\n");
else
    printf("  Result: MISMATCH!\r\n");
```

This gate provides **end-to-end integrity assurance**: from the packaging tool's SHA-256 embedding, to Flash writing, to read-back verification, it forms a complete data integrity closed loop.

### 10.9 A/B Partition as Safety Net

The endpoint of the trust chain is not "firmware written successfully", but "firmware runs correctly". The A/B partition provides a safety net for runtime validation:

```
New firmware written to inactive partition
  ├── State set to TESTING
  ├── boot_attempts = 0
  │
  ▼
Boot new firmware
  ├── Runs normally → App calls ab_partition_mark_slot_confirmed()
  │                     State transitions to CONFIRMED
  │
  └── Runs abnormally
      ├── boot_attempts incremented
      ├── boot_attempts >= 3 → auto-rollback to old partition
      └── SP validation fails → immediate rollback
```

Dual checks in `ab_partition_validate_slot()`:

```c
ab_err_t ab_partition_validate_slot(ab_slot_t slot) {
    uint32_t sp = *(__IO uint32_t *)addr;
    if (!ab_is_valid_sp(sp))           // 1. Stack pointer must be within SRAM range
        return AB_ERR_SLOT_INVALID_FW;

    uint32_t reset_handler = *(__IO uint32_t *)(addr + 4);
    if (reset_handler < addr ||        // 2. ResetHandler must fall within this partition's address range
        reset_handler > end_addr)
        return AB_ERR_SLOT_INVALID_FW;

    return AB_OK;
}
```

### 10.10 Formal Summary of Trust Chain

The entire trust chain can be expressed as a chained composition of verification functions, where each function's success is a prerequisite for the next function's execution:

$$V_{\text{total}} = V_{\text{HMAC}} \circ V_{\text{compat}} \circ V_{\text{rollback}} \circ V_{\text{HKDF}} \circ V_{\text{AES}} \circ V_{\text{Ed25519}} \circ V_{\text{SHA256}} \circ V_{\text{A/B}}$$

where each $V_i$ outputs $\{\text{PASS}, \text{FAIL}\}$, and:

$$V_{\text{total}} = \text{PASS} \iff \forall i, V_i = \text{PASS}$$

| Verification Gate | Security Property | Attack Mitigation | Root of Trust Dependency |
|-------------------|-------------------|-------------------|------------------------|
| HMAC-SHA256 | Source authentication | Forged Header | DevKey |
| Hardware compatibility | Correctness | Wrong hardware flash | None (plaintext comparison) |
| Security counter | Version monotonicity | Rollback attack | stored_counter |
| HKDF | Device binding | Cross-device decryption | DevKey + UID |
| AES-256 | Confidentiality | Firmware theft | HKDF-derived key |
| Ed25519 | Integrity + authentication | Content tampering | Ed25519 public key |
| SHA-256 read-back | Write integrity | Flash bit flip | None (read-back comparison) |
| A/B partition | Availability | Bricking attack | SP + ResetHandler |

---

## 11. Source Code Deep Dive

### 11.1 main.c: Boot Decision Entry Point

`main()` is the decision center of the entire Bootloader, with core logic as follows:

```c
int main(void) {
    // ===== Soft reset fast jump (before HAL_Init) =====
    if (update_flag == JUMP_FLAG_MAGIC) {
        update_flag = 0;
        __DSB(); __ISB();
        ab_slot_t slot = ab_partition_get_active_slot_from_flash();
        bootloader_execute_jump();  // Direct jump, fastest boot
    }

    // ===== Standard initialization =====
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_UART4_Init();
    MX_SPI1_Init();
    // ... other peripheral initialization
    platform_config_init();       // Register all Transport drivers
    ab_partition_init();          // Read A/B metadata

    // ===== A/B partition boot check =====
    ab_slot_t active = ab_partition_get_active_slot();
    ab_slot_state_t state = g_ab_metadata.slots[active].state;

    if (state == AB_SLOT_TESTING) {
        if (!ab_partition_validate_slot(active)) {
            ab_partition_rollback();        // Firmware invalid, rollback
        } else if (g_ab_metadata.slots[active].boot_attempts >= AB_MAX_BOOT_RETRIES) {
            ab_partition_rollback();        // Exceeded 3 attempts, rollback
        } else {
            ab_partition_increment_boot_attempts(active);  // Increment attempt count
        }
    }

    // ===== SD card mount =====
    if (SD_Detect() == SD_PRESENT) {
        f_mount(&SDFatFS, SDPath, 1);
    }

    // ===== Menu entry check =====
    Serial_PutString("\r\nPress 'M' to enter Bootloader menu (1.5s timeout)...\r\n");
    if (WaitForSerialCommand(1500) == 'M') {
        Main_Menu();             // Enter interactive menu
    } else if (update_flag == UPDATE_FLAG_MAGIC) {
        Main_Menu();             // App requests update
    } else {
        bootloader_request_jump(active);  // Auto jump to App
    }
}
```

> [!NOTE]
> The soft-reset fast jump is placed **before** `HAL_Init()`, meaning all peripherals are uninitialized at this point, but `bootloader_execute_jump()` only operates on core registers (MSP, VTOR), not relying on any peripherals, so it is safe. This design minimizes App boot latency.

### 11.2 bootloader_core.c: Download Engine

`bootloader_download()` is the core of normal download, implementing source-to-target data transfer via the Transport abstraction layer:

```c
int bootloader_download(platform_transport_base_t *source,
                        platform_transport_base_t *target,
                        const char *path) {
    uint8_t buf[DOWNLOAD_BUF_SIZE];  // 4096-byte buffer
    uint32_t total_size = 0;
    uint32_t bytes_read = 0;
    uint32_t offset = 0;

    // 1. Open source transport
    int ret = TRANSPORT_SOURCE_OPEN(source, path, &total_size);
    if (ret != TRANSPORT_STATUS_OK) return BL_ERR_OPEN_SRC;

    // 2. Open target transport
    ret = TRANSPORT_TARGET_OPEN(target, NULL, total_size);
    if (ret != TRANSPORT_STATUS_OK) {
        TRANSPORT_SOURCE_CLOSE(source);
        return BL_ERR_OPEN_DST;
    }

    // 3. Loop read → write
    do {
        ret = TRANSPORT_SOURCE_READ(source, buf, sizeof(buf), &bytes_read);
        if (ret != TRANSPORT_STATUS_OK) break;

        if (bytes_read > 0) {
            ret = TRANSPORT_TARGET_WRITE(target, offset, buf, bytes_read);
            if (ret != TRANSPORT_STATUS_OK) break;
            offset += bytes_read;
        }
    } while (bytes_read > 0);

    // 4. Close transport
    TRANSPORT_TARGET_CLOSE(target);
    TRANSPORT_SOURCE_CLOSE(source);

    return (ret == TRANSPORT_STATUS_OK) ? BL_OK : transport_status_to_bootloader(ret);
}
```

`bootloader_secure_download()` adds a complete security verification chain on top of normal download:

```c
int bootloader_secure_download(platform_transport_base_t *source,
                                platform_transport_base_t *target,
                                const char *path,
                                const fw_pkg_verify_config_t *config,
                                fw_pkg_ctx_t *result) {
    // Call firmware package processing engine to complete:
    // HMAC verify → anti-rollback → HKDF key derivation → AES decrypt → Ed25519 signature verify
    int ret = fw_pkg_process_ex(source, target, path, config, result);
    if (ret != FW_PKG_OK) return fw_pkg_err_to_bootloader(ret);
    return BL_OK;
}
```

### 11.3 firmware_package.c: Secure Firmware Package Engine

This is the most critical file of the security system (1107 lines), the complete flow of `fw_pkg_process_ex()`:

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

    // ===== 1. Read and parse Header =====
    SOURCE_READ(source, &header, sizeof(header));
    fw_pkg_parse_header(&header);  // Verify Magic + Version

    // ===== 2. HMAC-SHA256 verify Header =====
    fw_pkg_verify_header_hmac(&header, config->devkey);

    // ===== 3. Hardware compatibility check =====
    if (header.hardware_compat != config->hardware_id)
        return FW_PKG_ERR_HW_INCOMPAT;

    // ===== 4. Anti-rollback check =====
    fw_pkg_check_rollback(header.security_counter, config->min_security_counter);

    // ===== 5. Read DynamicSalt + IV =====
    SOURCE_READ(source, salt, SALT_SIZE);
    SOURCE_READ(source, iv, IV_SIZE);

    // ===== 6. HKDF key derivation =====
    uint8_t aes_key[32];
    fw_pkg_derive_aes_key(salt, config->devkey, config->uid, aes_key);

    // ===== 7. Initialize AES decryption =====
    fw_pkg_decrypt_init(&aes_ctx, header.encryption_algo, aes_key, iv);

    // ===== 8. Open target, loop decrypt and write =====
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

    // ===== 9. Read and verify Ed25519 signature =====
    SOURCE_READ(source, signature, SIGNATURE_SIZE);
    fw_pkg_verify_signature(&header, salt, iv, ciphertext,
                            ciphertext_size, signature, config->ed25519_pubkey);

    // ===== 10. Read back Flash SHA-256 check =====
    sha256_init(&sha256_ctx);
    for (offset = 0; offset < firmware_size; offset += sizeof(buf)) {
        TARGET_READ(target, offset, buf, chunk);
        sha256_update(&sha256_ctx, buf, chunk);
    }
    sha256_final(&sha256_ctx, computed_hash);
    // Compare with SHA-256 embedded at firmware end

    // ===== 11. Fill result context =====
    ctx->fw_version = header.firmware_version;
    ctx->security_counter = header.security_counter;
    memcpy(ctx->sha256, computed_hash, 32);

    return FW_PKG_OK;
}
```

> [!IMPORTANT]
> HKDF key derivation is the key to device-bound encryption. Of the three inputs to `fw_pkg_derive_aes_key()`, `DynamicSalt` is randomly generated per package, `DevKey` is stored in OTP and different per device, and `UID` is the chip's factory-unique ID. The combination of all three ensures: even if the firmware package is intercepted, it cannot be decrypted on other devices.

### 11.4 ab_partition.c: A/B Partition Management

The metadata append-write strategy is the essence of A/B partition management:

```c
// Metadata instance alignment: 128-byte aligned, up to 512 instances
#define METADATA_INSTANCE_ALIGN  128
#define METADATA_MAX_INSTANCES   (METADATA_SIZE / METADATA_INSTANCE_ALIGN)

static int ab_write_metadata_raw(const ab_metadata_t *meta) {
    uint32_t addr = METADATA_START_ADDR + g_ab_current_instance * METADATA_INSTANCE_ALIGN;

    // Check if can write directly (Flash only allows 1→0)
    if (can_write_at(addr, meta, sizeof(*meta))) {
        ab_flash_write_words(addr, (const uint32_t *)meta,
                             sizeof(*meta) / 4);
        g_ab_current_instance++;
    } else {
        // Cannot append write, perform compaction
        ab_compact_and_write(meta);
    }
}
```

The rollback logic ensures the device can always boot:

```c
int ab_partition_rollback(void) {
    ab_slot_t current = g_ab_metadata.active_slot;
    ab_slot_t fallback = (current == AB_SLOT_A) ? AB_SLOT_B : AB_SLOT_A;

    // Mark current slot as invalid
    g_ab_metadata.slots[current].state = AB_SLOT_INVALID;

    // Verify fallback slot has valid firmware
    if (!ab_partition_validate_slot(fallback)) {
        return AB_ERR_NO_VALID_SLOT;  // Both slots invalid!
    }

    // Switch to fallback slot
    g_ab_metadata.active_slot = fallback;
    g_ab_metadata.slots[fallback].state = AB_SLOT_CONFIRMED;
    g_ab_metadata.slots[fallback].boot_attempts = 0;
    ab_partition_metadata_flush();

    return AB_OK;
}
```

> [!WARNING]
> If both slots are invalid (`ab_partition_rollback` returns `AB_ERR_NO_VALID_SLOT`), the device cannot auto-boot and can only stay in the Bootloader menu waiting for manual firmware flashing. This is the last line of defense for A/B partitioning — the device won't brick, but requires manual intervention.

### 11.5 service_hpatch.c: Differential Upgrade Implementation

Differential upgrade uses HPatch Lite + tuz compression, with controlled memory usage:

```c
// Statically allocated buffers (compile-time fixed)
static uint8_t tuz_dict_and_cache[HPATCH_DICT_SIZE + HPATCH_CACHE_SIZE];  // 4KB + 16KB
static uint8_t stream_diff_buf[HPATCH_STREAM_BUF_SIZE];                   // 8KB
static uint8_t patch_temp_buf[HPATCH_CACHE_SIZE];                         // 16KB

int hpatch_upgrade(hpatch_config_t *config) {
    // 1. Open diff file, old firmware, create output file
    fs_instance->open(&diff_file, config->diff_path, "rb");
    fs_instance->open(&old_file, config->old_path, "rb");
    fs_instance->open(&out_file, config->out_path, "wb");

    // 2. Parse diff file header
    hpatch_lite_open(&listener, &new_size, &compress_type);

    // 3. If using tuz compression, initialize decompression stream
    if (compress_type == hpi_compress_tuz) {
        tuz_TStream_open(&tuz_stream_obj, tuz_dict_and_cache,
                         dict_size, tuz_fs_read_code);
        // Replace read callback with streaming decompression
        listener.read_diff = stream_read_diff;
    }

    // 4. Apply differential patch
    hpatch_lite_patch(&listener, new_size, patch_temp_buf);

    // 5. Cleanup
    fs_instance->close(&diff_file);
    fs_instance->close(&old_file);
    fs_instance->close(&out_file);
}
```

> [!TIP]
> Typical effect of differential upgrade: assuming old firmware is 384KB, new firmware is 386KB, the differential file might only be 5-20KB. In bandwidth-constrained scenarios (like NB-IoT, LoRa), differential upgrade can significantly reduce transfer time and data costs.

### 11.6 esp8266_ota_api.c: WiFi OTA Wrapper

WiFi OTA is a thin wrapper over the OneNET IoT platform:

```c
static onenet_ota_ctx_t g_ota_ctx;

void esp8266_ota_init(void) {
    // Inject three dependencies: WiFi module, RTC (for MQTT auth timestamp), MQTT client
    onenet_ota_ctx_init(&g_ota_ctx, &g_esp8266_wifi.base,
                        &g_rtc.base, &g_esp8266_mqtt.base);
}

int esp8266_ota_download(void) {
    onenet_ota_process_upgrade(&g_ota_ctx);  // MQTT receive → HTTP download → write to target
    return 1;
}
```

Supports three download targets:

```c
void esp8266_ota_set_target_internal_flash(void);  // Write directly to internal Flash
void esp8266_ota_set_target_sd_card(void);          // Save to SD card
void esp8266_ota_set_target_spi_flash(void);         // Save to SPI Flash
```

### 11.7 Overall Layered Architecture

| Layer                 | Module                                         | Description                                  |
| --------------------- | ---------------------------------------------- | -------------------------------------------- |
| **Application Layer** | Menu                                           | Interactive menu, 4600+ lines                |
| **Core Layer**        | Bootloader Core                                | Download/jump/secure download                |
|                       | ├ firmware_package                             | Package parsing/HMAC/AES/Ed25519             |
|                       | └ ab_partition                                 | A/B partition/rollback/metadata management   |
| **Service Layer**     | Service Layer                                  | Functional services                          |
|                       | ├ service_hpatch                               | Differential upgrade                         |
|                       | ├ esp8266_ota_api                              | WiFi OTA (OneNET)                            |
|                       | └ service_aes_decrypt / service_ed25519_verify | Encryption-decryption/signature verification |
| **Abstraction Layer** | Platform Transport                             | Transport abstraction                        |
|                       | ├ Source                                       | FatFS / LittleFS / Ymodem / HTTP             |
|                       | └ Target                                       | Internal Flash / FatFS / LittleFS            |
| **Hardware Layer**    | HAL                                            | STM32F407: Flash/RNG/SPI/UART/SDIO/RTC       |

---

## 12. Summary

This project is a feature-complete industrial-grade secure Bootloader, with core highlights:

1. **A/B Dual Partition**: seamless upgrade + auto-rollback, device never bricks
2. **Complete Security System**: HMAC authentication + AES-256 encryption + Ed25519 signature + anti-rollback + device-bound key
3. **Address Relocation + Dual Target Compilation**: only one source needed, simplifying build process
4. **Transport Abstraction**: download logic fully decoupled from storage media, easy to extend
5. **Multi-Channel Upgrade**: UART / SD card / SPI Flash / WiFi OTA, covering various scenarios
6. **Differential Upgrade**: HPatch + tuz reduces data transfer volume
7. **Interactive Menu**: feature-rich, convenient for debugging and production use

---

## References

- [STM32F4xx Reference Manual](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [Ed25519: High-speed high-security signatures](https://ed25519.cr.yp.to/)
- [HKDF (RFC 5869)](https://tools.ietf.org/html/rfc5869)
- [HPatchLite - Differential Patch Library](https://github.com/sisong/HPatchLite)
- [LittleFS - Embedded File System](https://github.com/littlefs-project/littlefs)
