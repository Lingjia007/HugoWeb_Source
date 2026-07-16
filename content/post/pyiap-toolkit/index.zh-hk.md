---
title: "PyIAPToolKit：STM32 IAP 上位機工具套件設計與實現"
date: 2026-06-20
description: "基於 PyQt6 + Fluent Design 的 STM32 IAP 上位機工具套件，涵蓋 AES-256 加密 + Ed25519 簽名固件打包流水線、HKDF 設備綁定密鑰派生、YMODEM 串口傳輸、PyOCD SWD 燒錄、雙差分引擎、LVGL 資源提取等核心技術詳解"
image: PyIAPToolKit.png
categories:
  - "嵌入式"
  - "上位機開發"
tags:
  - "STM32"
  - "IAP"
  - "PyQt6"
  - "AES-256"
  - "Ed25519"
  - "YMODEM"
  - "OTA"
  - "差分升級"
math: true
---

## 前言

在上一篇 [STM32F407 安全 Bootloader 設計](../stm32f407-secure-bootloader/) 中，我們詳細剖析了設備端的 A/B 雙分區、加密簽名與多渠道升級機制。然而，一個完整的 IAP 系統還需要**上位機工具**來完成固件打包、加密簽名、傳輸燒錄等環節。本文將介紹 **PyIAPToolKit** —— 一個基於 PyQt6 + Fluent Design 的 STM32 IAP 上位機工具套件，它是 Bootloader 專案在 PC 端的配套工具。

> [!NOTE]
> [下載發行版 v1.0.0](https://github.com/Lingjia007/PyIAPToolKit/releases/tag/v1.0.0)

---

## 一、專案概覽

### 1.1 功能矩陣

| 功能         | 描述                                                |
| ------------ | --------------------------------------------------- |
| 串口終端     | 多標籤 VT100 終端，支持 YMODEM 固件傳輸             |
| AES 加解密   | 5 種模式（CBC/ECB/CTR/CFB/OFB），HKDF 密鑰派生      |
| 固件頭封裝   | 64 字節二進制頭，含版本/演算法/安全計數器/HMAC 校驗 |
| Ed25519 簽名 | 密鑰對生成、簽名、驗證，SHA-512 預雜湊              |
| PyOCD 燒錄   | SWD 調試探針固件燒錄，支持 CMSIS Pack 目標          |
| 增量差分     | bsdiff4 / HPatchLite 雙引擎，OTA 增量更新           |
| HTML UI 提取 | Playwright 截圖提取 LVGL 嵌入式顯示資源             |
| 支付寶沙箱   | IoT 售貨機支付集成，自定義串口協議與 STM32 通信     |

### 1.2 技術棧

| 層次         | 技術選型                             |
| ------------ | ------------------------------------ |
| 編程語言     | Python 3                             |
| GUI 框架     | PyQt6                                |
| UI 組件庫    | qfluentwidgets（Fluent Design 風格） |
| 串口通信     | pyserial                             |
| 終端仿真     | pyte（VT100 解析）                   |
| 固件傳輸     | ymodem                               |
| 加密庫       | pycryptodome（AES, HMAC, Ed25519）   |
| 調試燒錄     | pyocd（ARM Cortex SWD）              |
| 增量差分     | bsdiff4, hpatchlite                  |
| 瀏覽器自動化 | playwright（Chromium headless）      |

### 1.3 專案結構

| 模組            | 文件                                                 | 代碼量  | 功能                |
| --------------- | ---------------------------------------------------- | ------- | ------------------- |
| 入口            | `main.py`                                            | 145 行  | 應用啟動、視窗建立  |
| 串口工具        | `serial_tools/serial_interface.py`                   | 2072 行 | VT100 終端 + YMODEM |
| PyOCD 工具      | `pyocd_tools/pyocd_interface.py`                     | 945 行  | SWD 燒錄            |
| AES 工具        | `aes_tools/aes_interface.py`                         | 1099 行 | 加解密 + HKDF       |
| BSDiff 工具     | `bsdiff_tools/bsdiff_interface.py`                   | 613 行  | 增量差分            |
| HPatchLite 工具 | `hpatchlite_tools/hpatchlite_interface.py`           | 863 行  | 增量差分            |
| 固件頭工具      | `firmware_header_tools/header_interface.py`          | 2129 行 | 固件打包流水線      |
| Ed25519 工具    | `ed25519_tools/ed25519_interface.py`                 | 936 行  | 數字簽名            |
| HTML UI 提取    | `html_ui_extract_tools/html_ui_extract_interface.py` | 985 行  | LVGL 資源提取       |
| 支付寶沙箱      | `alipay_sandbox_tools/alipay_sandbox_interface.py`   | 1714 行 | 支付集成            |

---

## 二、應用架構

### 2.1 FluentWindow 側邊欄導航

主視窗採用 `FluentWindow` 側邊欄導航模式，10 個功能頁面各對應一個獨立模組：

```python
class Window(FluentWindow):
    def __init__(self):
        self.serialInterface = SerialTabWidget()
        self.pyocdInterface = Pyocd_Tools_Widget()
        self.aesInterface = AES_Tools_Widget()
        self.bsdiffInterface = BSDiff_Tools_Widget()
        self.hpatchliteInterface = HPatchLite_Tools_Widget()
        self.firmwareHeaderInterface = FirmwareHeader_Widget()
        self.ed25519Interface = Ed25519_Widget()
        self.htmlUIExtractInterface = HTML_UI_Extract_Widget()
        self.alipaySandboxInterface = AlipaySandbox_Widget()
        self.settingInterface = SettingInterface(self)
```

### 2.2 啟動流程

1. DPI 縮放初始化
2. 國際化翻譯加載（zh_CN, zh_HK, en_US）
3. 閃屏顯示
4. 主視窗建立與導航註冊

### 2.3 QThread + pyqtSignal 異步模式

所有耗時操作採用統一的異步工作模式，確保 GUI 主線程不被阻塞：

```python
class XxxThread(QThread):
    progress_signal = pyqtSignal(int, str)   # 進度回調
    finished_signal = pyqtSignal(bool, str)  # 完成通知

    def run(self):
        # 耗時操作
        self.progress_signal.emit(percent, message)
        self.finished_signal.emit(success, result)
```

---

## 三、固件打包流水線（核心）

這是整個工具套件最核心的功能，將原始固件經過完整的安全處理鏈路，生成可部署的 `.iap.bin` 固件包。

### 3.1 固件頭結構（64 字節）

| 偏移 | 字段                   | 大小 | 說明                                            |
| ---- | ---------------------- | ---- | ----------------------------------------------- |
| 0x00 | magic                  | 4B   | `IAP\x01` 魔數                                  |
| 0x04 | header_version         | 1B   | 頭版本號                                        |
| 0x05 | firmware_version_major | 1B   | 固件主版本                                      |
| 0x06 | firmware_version_minor | 1B   | 固件次版本                                      |
| 0x07 | firmware_version_patch | 1B   | 固件補丁版本                                    |
| 0x08 | total_payload_size     | 4B   | 加密後載荷總大小                                |
| 0x0C | image_type             | 1B   | 0x01=App, 0x02=Bootloader, 0x03=Resource        |
| 0x0D | encryption_algorithm   | 1B   | 0x00=None, 0x01=AES-256-CBC, 0x02=ECB, 0x03=CTR |
| 0x0E | signature_algorithm    | 1B   | 0x00=None, 0x01=Ed25519                         |
| 0x0F | hardware_compatibility | 1B   | 硬件兼容性標識                                  |
| 0x10 | security_counter       | 4B   | 安全計數器（防回滾）                            |
| 0x14 | build_timestamp        | 4B   | 構建時間戳                                      |
| 0x18 | reserved               | 5B   | 保留字段                                        |
| 0x1D | header_checksum        | 32B  | HMAC-SHA256 校驗（使用 DevKey）                 |

### 3.2 最終固件包格式

| 區域        | 大小     | 說明                                |
| ----------- | -------- | ----------------------------------- |
| Header      | 64 bytes | 魔術字、版本、加密/簽名演算法等     |
| DynamicSalt | 16 bytes | HKDF 鹽值，每固件獨立               |
| IV          | 16 bytes | AES 初始化向量                      |
| Ciphertext  | N bytes  | AES-256 加密的固件 + 追加的 SHA-256 |
| Signature   | 64 bytes | Ed25519 數字簽名                    |

### 3.3 打包線程核心邏輯

```python
class PackageThread(QThread):
    def run(self):
        # 1. 讀取原始固件
        firmware_data = open(firmware_path, 'rb').read()

        # 2. 追加 SHA-256 雜湊用於完整性校驗
        firmware_hash = SHA256.new(firmware_data).digest()
        data_to_encrypt = firmware_data + firmware_hash

        # 3. AES-256 加密（密鑰由 HKDF 派生）
        cipher = AES.new(aes_key, AES.MODE_CBC, iv=iv)
        ciphertext = cipher.encrypt(pad(data_to_encrypt, AES.block_size))

        # 4. 構建固件頭
        header = FirmwareHeader(...)
        header.header_checksum = HMAC.new(dev_key, header_prefix, SHA256).digest()

        # 5. Ed25519 簽名（對 header + salt + iv + ciphertext）
        signer = eddsa.new(private_key, 'rfc8032')
        signature = signer.sign(payload)

        # 6. 組裝最終包
        package = header_bytes + dynamic_salt + iv + ciphertext + signature
```

> [!NOTE]
> 加密前先追加 SHA-256 雜湊是關鍵設計。解密時，STM32 端先解密去除填充，再驗證末尾 32 字節 SHA-256 是否與解密數據的前部匹配，實現**解密即校驗**的雙重保障。

### 3.4 可視化對比

打包完成後，`PackCompareDialog` 用顏色編碼展示固件包各區域：

| 顏色 | 區域               |
| ---- | ------------------ |
| 藍色 | Header（64B）      |
| 紅色 | DynamicSalt（16B） |
| 紫色 | IV（16B）          |
| 綠色 | Ciphertext（變長） |
| 橙色 | Signature（64B）   |

---

## 四、HKDF 設備綁定密鑰派生

這是安全體系的核心機制，確保固件包只能由目標設備解密。

### 4.1 兩階段 HKDF

```python
def hkdf_extract(salt: bytes, ikm: bytes) -> bytes:
    """HKDF-Extract: 從輸入密鑰材料提取固定長度的偽隨機密鑰"""
    prk = HMAC.new(salt, ikm, digestmod=SHA256).digest()
    return prk

def hkdf_expand(prk: bytes, info: bytes, length: int) -> bytes:
    """HKDF-Expand: 將偽隨機密鑰擴展為所需長度的輸出密鑰材料"""
    hash_len = SHA256.digest_size
    n = (length + hash_len - 1) // hash_len
    okm = b''
    t = b''
    for i in range(1, n + 1):
        t = HMAC.new(prk, t + info + bytes([i]), digestmod=SHA256).digest()
        okm += t
    return okm[:length]
```

### 4.2 密鑰派生鏈路

1. **Extract 階段**：`HMAC-SHA256(salt=DynamicSalt, ikm=DevKey)` → PRK
2. **Expand 階段**：`HMAC-SHA256(PRK, info=UID || counter)` → 32 字節 AES 密鑰

三個安全要素的角色：

| 要素                  | 來源               | 作用                                         |
| --------------------- | ------------------ | -------------------------------------------- |
| DevKey (128-bit)      | STM32 OTP 熔絲     | 設備密鑰，不可讀取                           |
| UID (96-bit)          | STM32 晶片唯一 ID  | HKDF-Expand 的 info 參數，實現密鑰與晶片綁定 |
| DynamicSalt (128-bit) | 每固件版本隨機生成 | 確保同設備不同固件版本產生不同加密密鑰       |

> [!IMPORTANT]
> 三個要素缺一不可：DevKey 保證只有合法設備能解密，UID 保證固件包只能由特定晶片解密，DynamicSalt 保證同一設備的每次升級使用不同密鑰。這就是**設備綁定加密**的核心原理。

---

## 五、AES-256 加密模組

### 5.1 支持的加密模式

| 模式 | 特點                     | 適用場景               |
| ---- | ------------------------ | ---------------------- |
| CBC  | 需 IV，並行解密          | 預設推薦               |
| ECB  | 無 IV，相同明文→相同密文 | 不推薦（僅兼容舊方案） |
| CTR  | 流式加密，無需填充       | 高性能場景             |
| CFB  | 流式加密，自同步         | 誤碼容忍場景           |
| OFB  | 流式加密，無錯誤傳播     | 噪聲信道場景           |

### 5.2 加密流程

```python
# 加密前先追加 SHA-256 雜湊
firmware_hash = SHA256.new(firmware_data).digest()
data_to_encrypt = firmware_data + firmware_hash  # 原始固件 + 32字節雜湊

# PKCS7 填充後加密
cipher = AES.new(key, AES.MODE_CBC, iv=iv)
ciphertext = cipher.encrypt(pad(data_to_encrypt, AES.block_size))
```

### 5.3 解密與校驗

```python
# 解密
cipher = AES.new(key, AES.MODE_CBC, iv=iv)
plaintext = unpad(cipher.decrypt(ciphertext), AES.block_size)

# 分離固件和雜湊
firmware = plaintext[:-32]
embedded_hash = plaintext[-32:]

# 校驗完整性
computed_hash = SHA256.new(firmware).digest()
assert computed_hash == embedded_hash, "固件完整性校驗失敗"
```

---

## 六、Ed25519 數字簽名

### 6.1 簽名流程

```python
# 密鑰對生成
key = ECC.generate(curve='ed25519')

# 簽名（SHA-512 預雜湊，RFC8032 變體）
signer = eddsa.new(key, 'rfc8032')
signature = signer.sign(data)  # 輸出 64 字節簽名

# 驗證
verifier = eddsa.new(key, 'rfc8032')
verifier.verify(signature, data)
```

### 6.2 密鑰格式支持

| 格式     | 用途             |
| -------- | ---------------- |
| PEM      | 標準存儲格式     |
| 原始字節 | 嵌入式端使用     |
| 十六進制 | 調試顯示         |
| C 數組   | 直接嵌入固件源碼 |

> [!TIP]
> Ed25519 的公鑰只有 32 字節，簽名只有 64 字節，非常適合資源受限的嵌入式場景。相比 RSA-2048（256 字節簽名），Ed25519 在安全性和效率上都有顯著優勢。

---

## 七、串口終端與 YMODEM 傳輸

### 7.1 VT100 終端仿真

基於 pyte 實現 VT100 終端解析：

```python
class PyteTerminal:
    def __init__(self, cols=80, rows=24):
        self.screen = pyte.Screen(cols, rows)
        self.stream = pyte.Stream(self.screen)

    def feed(self, data):
        self.stream.feed(data)  # 解析 ANSI 轉義序列

    def get_display(self):
        return self.screen.display  # 獲取渲染後的文本行
```

`TerminalTextEdit` 實現了完整的終端模式：支持游標定位、顏色渲染（使用格式化快取優化性能）、滾動區域管理。

### 7.2 YMODEM 固件傳輸

```python
class YModem_Send_Thread(QThread):
    def run(self):
        def read(size, timeout=3):
            data = self.serial_port.read(size)
            return data
        def write(data, timeout=3):
            self.serial_port.write(data)
            return len(data)

        cli = ModemSocket(read, write)
        result = cli.send(self.file_paths, callback=ymodem_callback)
```

### 7.3 調試器自動發現

串口模組通過 USB VID/PID 自動識別 JLink/STLink/DAPLink，並與 PyOCD 模組聯動，減少用戶手動配置。

---

## 八、PyOCD SWD 燒錄

通過 ARM SWD 接口直接燒錄 Flash，不經過 Bootloader：

```python
class Pyocd_Program_Thread(QThread):
    def run(self):
        session = ConnectHelper.session_with_chosen_probe(
            target=self.target,
            connect_mode=self.connect_mode,  # halt/attach/pre-reset
        )
        with session:
            board = session.board
            flash = board.flash
            flash.build_image()
            flash.program(
                self.firmware_data,
                base_address=self.base_address,
                erase=self.erase_mode,  # chip/sector/no-erase
                trust_crc=self.trust_crc,
            )
```

| 特性            | 說明                               |
| --------------- | ---------------------------------- |
| 探針自動刷新    | 5 秒間隔，變更檢測                 |
| CMSIS Pack 支持 | 掃描 .pack/.pdsc 獲取目標 MCU 定義 |
| 預設目標        | `stm32f407vgtx`                    |
| 擦除模式        | 全片擦除 / 扇區擦除 / 不擦除       |
| 連接模式        | halt / attach / pre-reset          |

---

## 九、雙差分引擎

同時集成 bsdiff4 和 HPatchLite 兩種差分引擎，為不同場景提供選擇。

### 9.1 引擎對比

| 特性       | bsdiff4          | HPatchLite                |
| ---------- | ---------------- | ------------------------- |
| 類型       | Python 庫        | 外部可執行文件            |
| 壓縮演算法 | bzip2            | tuz / zlib / lzma / lzma2 |
| 集成方式   | `import bsdiff4` | 子進程調用                |
| 並行支持   | 無               | 支持多線程                |
| 原地補丁   | 不支持           | 支持                      |
| 適用場景   | 簡單快速         | 高級選項、嵌入式端兼容    |

### 9.2 差分升級效果

假設舊固件 384KB，新固件 386KB，差分文件可能只有 5-20KB。在頻寬受限的場景下，差分升級能顯著減少傳輸時間和流量費用。

---

## 十、支付寶沙箱與自定義串口協議

### 10.1 幀格式

```
0xAA 0x55 | CMD(1B) | LEN(2B, 大端序) | DATA(NB) | 0x0D 0x0A
```

```python
def _build_frame(self, cmd, data):
    header = bytes([0xAA, 0x55])
    tail = bytes([0x0D, 0x0A])
    length = struct.pack('>H', len(data))
    return header + bytes([cmd]) + length + data + tail
```

### 10.2 命令集

| 方向     | CMD  | 說明           |
| -------- | ---- | -------------- |
| PC→STM32 | 0x01 | 發送二維碼 URL |
| PC→STM32 | 0x02 | 支付狀態通知   |
| PC→STM32 | 0x03 | 發送交易號     |
| PC→STM32 | 0x04 | 心跳響應       |
| STM32→PC | 0x81 | 請求二維碼     |
| STM32→PC | 0x82 | 查詢支付狀態   |
| STM32→PC | 0x83 | 關閉訂單       |
| STM32→PC | 0x84 | 心跳           |

> [!NOTE]
> 命令編碼規則：`0x0x` 為 PC→STM32，`0x8x` 為 STM32→PC，最高位區分方向。接收線程 `SerialReceiveThread` 解析 `0xAA 0x55` 幀頭，提取命令碼和數據載荷。

---

## 十一、HTML UI 提取（LVGL 資源）

通過 Playwright（Chromium headless）自動截圖，將 HTML/CSS 設計稿轉換為 LVGL 嵌入式顯示資源：

1. 加載 HTML 文件到 headless 瀏覽器
2. 截取指定區域/元素
3. 轉換為 C 數組格式的圖像數據
4. 輸出為 LVGL 兼容的 `.c` / `.h` 文件

---

## 十二、安全鏈路全景

**上位機構建端：**

1. 原始固件 → SHA-256 雜湊追加
2. → HKDF 派生 AES 密鑰（DynamicSalt + DevKey + UID）
3. → AES-256 加密
4. → 構建 Header + HMAC-SHA256 校驗
5. → Ed25519 簽名
6. → `.iap.bin`

**設備端驗證（STM32 Bootloader）：**

1. `.iap.bin` → HMAC-SHA256 驗證 Header（DevKey）
2. → 硬件兼容檢查
3. → 防回滾檢查（安全計數器）
4. → HKDF 派生 AES 密鑰（Salt + DevKey + UID）
5. → AES 解密 → 寫入 Flash
6. → SHA-512 流式雜湊 → Ed25519 簽名驗證
7. → 回讀 Flash SHA-256 校驗

**安全分層防禦：**

| 層         | 機制             | 解決的問題     |
| ---------- | ---------------- | -------------- |
| 真實性     | Ed25519 簽名     | 誰簽的？       |
| 頭完整性   | HMAC-SHA256      | 頭被篡改？     |
| 機密性     | AES-256 加密     | 內容保密       |
| 載荷完整性 | SHA-256 雜湊     | 數據正確？     |
| 防回滾     | Security Counter | 舊版本？       |
| 密鑰隔離   | HKDF 派生        | 同設備不同密鑰 |
| 晶片綁定   | UID 綁定         | 密鑰與晶片關聯 |

---

## 十三、Nuitka 打包部署

專案使用 Nuitka 將 Python 應用編譯為獨立的 Windows 可執行文件，無需用戶安裝 Python 環境。

### 13.1 構建命令

```bash
nuitka \
  --standalone \
  --onefile-no-compression \
  --lto=yes \
  --show-memory \
  --show-progress \
  --windows-console-mode=disable \
  --enable-plugin=pyqt6 \
  --enable-plugin=upx \
  --output-dir=out \
  --include-data-dir=settings/resource=settings/resource \
  --include-data-file=settings/config/config.json=settings/config/config.json \
  --include-data-file=hpatchlite_tools/hdiffi.exe=hpatchlite_tools/hdiffi.exe \
  --include-data-file=hpatchlite_tools/hpatchi.exe=hpatchlite_tools/hpatchi.exe \
  --windows-icon-from-ico=settings/resource/images/logo.ico \
  main.py
```

### 13.2 參數說明

| 參數                             | 說明                                               |
| -------------------------------- | -------------------------------------------------- |
| `--standalone`                   | 生成獨立可執行文件，不依賴本機 Python              |
| `--onefile-no-compression`       | 單文件輸出，不壓縮（啟動更快）                     |
| `--lto=yes`                      | 啟用鏈接時優化（Link Time Optimization），減小體積 |
| `--show-memory`                  | 顯示內存使用情況                                   |
| `--show-progress`                | 顯示編譯進度                                       |
| `--windows-console-mode=disable` | 隱藏控制台視窗（GUI 應用）                         |
| `--enable-plugin=pyqt6`          | 啟用 PyQt6 插件支持                                |
| `--enable-plugin=upx`            | 啟用 UPX 壓縮進一步減小體積                        |
| `--output-dir=out`               | 輸出目錄                                           |
| `--include-data-dir`             | 包含資源目錄（圖標、翻譯等）                       |
| `--include-data-file`            | 包含單個數據文件（配置、外部工具）                 |
| `--windows-icon-from-ico`        | 設置應用圖標                                       |

### 13.3 PyQt6 打包注意事項

> [!WARNING]
> PyQt6 應用的 Nuitka 打包有幾個常見陷阱：
>
> 1. **必須使用 `--enable-plugin=pyqt6`**：否則 Nuitka 無法自動發現 PyQt6 的隱式導入和插件依賴（如平台插件 `qwindows.dll`、圖像格式插件等）
> 2. **qfluentwidgets 資源**：qfluentwidgets 的 QRC 資源文件需要通過 `--include-data-dir` 顯式包含，否則運行時圖標和樣式會丟失
> 3. **PyQt6 平台插件**：如果運行時出現 `This application failed to start because no Qt platform plugin could be initialized`，需要確保 `platforms/qwindows.dll` 被正確包含，`--enable-plugin=pyqt6` 通常會自動處理
> 4. **UPX 兼容性**：某些 PyQt6 的 DLL 被 UPX 壓縮後可能無法正常加載，如果出現運行時錯誤，嘗試移除 `--enable-plugin=upx`

### 13.4 外部工具嵌入

HPatchLite 的差分工具（`hdiffi.exe` / `hpatchi.exe`）作為外部可執行文件嵌入：

```bash
--include-data-file=hpatchlite_tools/hdiffi.exe=hpatchlite_tools/hdiffi.exe
--include-data-file=hpatchlite_tools/hpatchi.exe=hpatchlite_tools/hpatchi.exe
```

運行時通過 `sys._MEIPASS`（Nuitka standalone 模式下為 `.dist` 目錄）定位這些文件：

```python
import sys
import os

def get_resource_path(relative_path):
    """獲取資源文件的絕對路徑（兼容開發環境和打包後環境）"""
    if hasattr(sys, '_MEIPASS'):
        # PyInstaller 打包模式
        return os.path.join(sys._MEIPASS, relative_path)
    elif getattr(sys, 'frozen', False):
        # Nuitka standalone 模式
        return os.path.join(os.path.dirname(sys.executable), relative_path)
    else:
        # 開發環境
        return os.path.join(os.path.dirname(__file__), relative_path)
```

---

## 十四、配置系統

基於 qfluentwidgets 的 `QConfig` 框架，支持持久化存儲：

| 配置項              | 分組       | 預設值   | 說明            |
| ------------------- | ---------- | -------- | --------------- |
| `serialFontSize`    | Serial     | 12       | 終端字號        |
| `serialFontFamily`  | Serial     | Consolas | 終端字體        |
| `serialDTR`         | Serial     | True     | DTR 信號        |
| `serialRTS`         | Serial     | True     | RTS 信號        |
| `pyocdFirmwarePath` | PyOCD      | ""       | 固件路徑        |
| `pyocdTrustCRC`     | PyOCD      | True     | 信任 CRC        |
| `cmPackPath`        | PyOCD      | ""       | CMSIS Pack 路徑 |
| `language`          | MainWindow | Auto     | 界面語言        |
| `dpiScale`          | MainWindow | Auto     | DPI 縮放        |

---

## 十五、與 Bootloader 的協作關係

PyIAPToolKit 和 STM32F407 Bootloader 構成了完整的 IAP 系統：

| 環節     | 上位機（PyIAPToolKit）                | 設備端（Bootloader）                |
| -------- | ------------------------------------- | ----------------------------------- |
| 固件打包 | AES 加密 + Ed25519 簽名 + Header 封裝 | —                                   |
| 密鑰派生 | HKDF（Salt + DevKey + UID）           | HKDF（Salt + DevKey + UID）         |
| 固件傳輸 | YMODEM 發送 / PyOCD 燒錄              | YMODEM 接收 / SWD 寫入              |
| 固件驗證 | —                                     | HMAC → 防回滾 → AES 解密 → 簽名驗證 |
| 分區管理 | —                                     | A/B 雙分區 + 自動回滾               |
| 差分升級 | bsdiff4 / HPatchLite 生成差分包       | HPatchLite 應用差分包               |

> [!TIP]
> 上位機和設備端使用**完全相同的密鑰派生演算法**（HKDF），確保派生出的 AES 密鑰一致。DevKey 在設備端存儲在 OTP 中，在上位機端由用戶導入；UID 在設備端從寄存器讀取，在上位機端由用戶輸入或從設備自動獲取。

---

## 十六、總結

PyIAPToolKit 的核心亮點：

1. **完整固件打包流水線**：SHA-256 → AES-256 → HMAC → Ed25519，一站式完成
2. **設備綁定加密**：HKDF（DynamicSalt + DevKey + UID）確保固件包只能由目標設備解密
3. **雙傳輸通道**：YMODEM 串口傳輸 + PyOCD SWD 燒錄，覆蓋開發和生產場景
4. **雙差分引擎**：bsdiff4 + HPatchLite，靈活選擇
5. **Fluent Design UI**：qfluentwidgets 組件庫，現代化界面體驗
6. **模組化架構**：10 個獨立功能模組，互不耦合，易於擴展
7. **可視化對比**：顏色編碼展示固件包各區域，直觀清晰

---

## 參考資料

- [PyQt6 Documentation](https://www.riverbankcomputing.com/static/Docs/PyQt6/)
- [qfluentwidgets - Fluent Design for PyQt](https://qfluentwidgets.com/)
- [HKDF (RFC 5869)](https://tools.ietf.org/html/rfc5869)
- [Ed25519: High-speed high-security signatures](https://ed25519.cr.yp.to/)
- [PyOCD - Python library for programming and debugging ARM Cortex](https://pyocd.io/)
- [HPatchLite - 差分補丁庫](https://github.com/sisong/HPatchLite)
- [YMODEM Protocol](https://en.wikipedia.org/wiki/YMODEM)
