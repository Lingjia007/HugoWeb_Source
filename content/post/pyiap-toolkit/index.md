---
title: "PyIAPToolKit：STM32 IAP 上位机工具套件设计与实现"
date: 2026-06-20
description: "基于 PyQt6 + Fluent Design 的 STM32 IAP 上位机工具套件，涵盖 AES-256 加密 + Ed25519 签名固件打包流水线、HKDF 设备绑定密钥派生、YMODEM 串口传输、PyOCD SWD 烧录、双差分引擎、LVGL 资源提取等核心技术详解"
image: PyIAPToolKit.png
categories:
  - "嵌入式"
tags:
  - "STM32"
  - "IAP"
  - "PyQt6"
  - "AES-256"
  - "Ed25519"
  - "YMODEM"
  - "OTA"
  - "差分升级"
math: true
---

## 前言

在上一篇 [STM32F407 安全 Bootloader 设计](../stm32f407-secure-bootloader/) 中，我们详细剖析了设备端的 A/B 双分区、加密签名与多渠道升级机制。然而，一个完整的 IAP 系统还需要**上位机工具**来完成固件打包、加密签名、传输烧录等环节。本文将介绍 **PyIAPToolKit** —— 一个基于 PyQt6 + Fluent Design 的 STM32 IAP 上位机工具套件，它是 Bootloader 项目在 PC 端的配套工具。

> [!NOTE]
> [下载发行版 v1.0.0](https://github.com/Lingjia007/PyIAPToolKit/releases/tag/v1.0.0)

---

## 一、项目概览

### 1.1 功能矩阵

| 功能         | 描述                                              |
| ------------ | ------------------------------------------------- |
| 串口终端     | 多标签 VT100 终端，支持 YMODEM 固件传输           |
| AES 加解密   | 5 种模式（CBC/ECB/CTR/CFB/OFB），HKDF 密钥派生    |
| 固件头封装   | 64 字节二进制头，含版本/算法/安全计数器/HMAC 校验 |
| Ed25519 签名 | 密钥对生成、签名、验证，SHA-512 预哈希            |
| PyOCD 烧录   | SWD 调试探针固件烧录，支持 CMSIS Pack 目标        |
| 增量差分     | bsdiff4 / HPatchLite 双引擎，OTA 增量更新         |
| HTML UI 提取 | Playwright 截图提取 LVGL 嵌入式显示资源           |
| 支付宝沙箱   | IoT 售货机支付集成，自定义串口协议与 STM32 通信   |

### 1.2 技术栈

| 层次         | 技术选型                             |
| ------------ | ------------------------------------ |
| 编程语言     | Python 3                             |
| GUI 框架     | PyQt6                                |
| UI 组件库    | qfluentwidgets（Fluent Design 风格） |
| 串口通信     | pyserial                             |
| 终端仿真     | pyte（VT100 解析）                   |
| 固件传输     | ymodem                               |
| 加密库       | pycryptodome（AES, HMAC, Ed25519）   |
| 调试烧录     | pyocd（ARM Cortex SWD）              |
| 增量差分     | bsdiff4, hpatchlite                  |
| 浏览器自动化 | playwright（Chromium headless）      |

### 1.3 项目结构

| 模块            | 文件                                                 | 代码量  | 功能                |
| --------------- | ---------------------------------------------------- | ------- | ------------------- |
| 入口            | `main.py`                                            | 145 行  | 应用启动、窗口创建  |
| 串口工具        | `serial_tools/serial_interface.py`                   | 2072 行 | VT100 终端 + YMODEM |
| PyOCD 工具      | `pyocd_tools/pyocd_interface.py`                     | 945 行  | SWD 烧录            |
| AES 工具        | `aes_tools/aes_interface.py`                         | 1099 行 | 加解密 + HKDF       |
| BSDiff 工具     | `bsdiff_tools/bsdiff_interface.py`                   | 613 行  | 增量差分            |
| HPatchLite 工具 | `hpatchlite_tools/hpatchlite_interface.py`           | 863 行  | 增量差分            |
| 固件头工具      | `firmware_header_tools/header_interface.py`          | 2129 行 | 固件打包流水线      |
| Ed25519 工具    | `ed25519_tools/ed25519_interface.py`                 | 936 行  | 数字签名            |
| HTML UI 提取    | `html_ui_extract_tools/html_ui_extract_interface.py` | 985 行  | LVGL 资源提取       |
| 支付宝沙箱      | `alipay_sandbox_tools/alipay_sandbox_interface.py`   | 1714 行 | 支付集成            |

---

## 二、应用架构

### 2.1 FluentWindow 侧边栏导航

主窗口采用 `FluentWindow` 侧边栏导航模式，10 个功能页面各对应一个独立模块：

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

### 2.2 启动流程

1. DPI 缩放初始化
2. 国际化翻译加载（zh_CN, zh_HK, en_US）
3. 闪屏显示
4. 主窗口创建与导航注册

### 2.3 QThread + pyqtSignal 异步模式

所有耗时操作采用统一的异步工作模式，确保 GUI 主线程不被阻塞：

```python
class XxxThread(QThread):
    progress_signal = pyqtSignal(int, str)   # 进度回调
    finished_signal = pyqtSignal(bool, str)  # 完成通知

    def run(self):
        # 耗时操作
        self.progress_signal.emit(percent, message)
        self.finished_signal.emit(success, result)
```

---

## 三、固件打包流水线（核心）

这是整个工具套件最核心的功能，将原始固件经过完整的安全处理链路，生成可部署的 `.iap.bin` 固件包。

### 3.1 固件头结构（64 字节）

| 偏移 | 字段                   | 大小 | 说明                                            |
| ---- | ---------------------- | ---- | ----------------------------------------------- |
| 0x00 | magic                  | 4B   | `IAP\x01` 魔数                                  |
| 0x04 | header_version         | 1B   | 头版本号                                        |
| 0x05 | firmware_version_major | 1B   | 固件主版本                                      |
| 0x06 | firmware_version_minor | 1B   | 固件次版本                                      |
| 0x07 | firmware_version_patch | 1B   | 固件补丁版本                                    |
| 0x08 | total_payload_size     | 4B   | 加密后载荷总大小                                |
| 0x0C | image_type             | 1B   | 0x01=App, 0x02=Bootloader, 0x03=Resource        |
| 0x0D | encryption_algorithm   | 1B   | 0x00=None, 0x01=AES-256-CBC, 0x02=ECB, 0x03=CTR |
| 0x0E | signature_algorithm    | 1B   | 0x00=None, 0x01=Ed25519                         |
| 0x0F | hardware_compatibility | 1B   | 硬件兼容性标识                                  |
| 0x10 | security_counter       | 4B   | 安全计数器（防回滚）                            |
| 0x14 | build_timestamp        | 4B   | 构建时间戳                                      |
| 0x18 | reserved               | 5B   | 保留字段                                        |
| 0x1D | header_checksum        | 32B  | HMAC-SHA256 校验（使用 DevKey）                 |

### 3.2 最终固件包格式

| 区域        | 大小     | 说明                                |
| ----------- | -------- | ----------------------------------- |
| Header      | 64 bytes | 魔术字、版本、加密/签名算法等       |
| DynamicSalt | 16 bytes | HKDF 盐值，每固件独立               |
| IV          | 16 bytes | AES 初始化向量                      |
| Ciphertext  | N bytes  | AES-256 加密的固件 + 追加的 SHA-256 |
| Signature   | 64 bytes | Ed25519 数字签名                    |

### 3.3 打包线程核心逻辑

```python
class PackageThread(QThread):
    def run(self):
        # 1. 读取原始固件
        firmware_data = open(firmware_path, 'rb').read()

        # 2. 追加 SHA-256 哈希用于完整性校验
        firmware_hash = SHA256.new(firmware_data).digest()
        data_to_encrypt = firmware_data + firmware_hash

        # 3. AES-256 加密（密钥由 HKDF 派生）
        cipher = AES.new(aes_key, AES.MODE_CBC, iv=iv)
        ciphertext = cipher.encrypt(pad(data_to_encrypt, AES.block_size))

        # 4. 构建固件头
        header = FirmwareHeader(...)
        header.header_checksum = HMAC.new(dev_key, header_prefix, SHA256).digest()

        # 5. Ed25519 签名（对 header + salt + iv + ciphertext）
        signer = eddsa.new(private_key, 'rfc8032')
        signature = signer.sign(payload)

        # 6. 组装最终包
        package = header_bytes + dynamic_salt + iv + ciphertext + signature
```

> [!NOTE]
> 加密前先追加 SHA-256 哈希是关键设计。解密时，STM32 端先解密去除填充，再验证末尾 32 字节 SHA-256 是否与解密数据的前部匹配，实现**解密即校验**的双重保障。

### 3.4 可视化对比

打包完成后，`PackCompareDialog` 用颜色编码展示固件包各区域：

| 颜色 | 区域               |
| ---- | ------------------ |
| 蓝色 | Header（64B）      |
| 红色 | DynamicSalt（16B） |
| 紫色 | IV（16B）          |
| 绿色 | Ciphertext（变长） |
| 橙色 | Signature（64B）   |

---

## 四、HKDF 设备绑定密钥派生

这是安全体系的核心机制，确保固件包只能由目标设备解密。

### 4.1 两阶段 HKDF

```python
def hkdf_extract(salt: bytes, ikm: bytes) -> bytes:
    """HKDF-Extract: 从输入密钥材料提取固定长度的伪随机密钥"""
    prk = HMAC.new(salt, ikm, digestmod=SHA256).digest()
    return prk

def hkdf_expand(prk: bytes, info: bytes, length: int) -> bytes:
    """HKDF-Expand: 将伪随机密钥扩展为所需长度的输出密钥材料"""
    hash_len = SHA256.digest_size
    n = (length + hash_len - 1) // hash_len
    okm = b''
    t = b''
    for i in range(1, n + 1):
        t = HMAC.new(prk, t + info + bytes([i]), digestmod=SHA256).digest()
        okm += t
    return okm[:length]
```

### 4.2 密钥派生链路

1. **Extract 阶段**：`HMAC-SHA256(salt=DynamicSalt, ikm=DevKey)` → PRK
2. **Expand 阶段**：`HMAC-SHA256(PRK, info=UID || counter)` → 32 字节 AES 密钥

三个安全要素的角色：

| 要素                  | 来源               | 作用                                         |
| --------------------- | ------------------ | -------------------------------------------- |
| DevKey (128-bit)      | STM32 OTP 熔丝     | 设备密钥，不可读取                           |
| UID (96-bit)          | STM32 芯片唯一 ID  | HKDF-Expand 的 info 参数，实现密钥与芯片绑定 |
| DynamicSalt (128-bit) | 每固件版本随机生成 | 确保同设备不同固件版本产生不同加密密钥       |

> [!IMPORTANT]
> 三个要素缺一不可：DevKey 保证只有合法设备能解密，UID 保证固件包只能由特定芯片解密，DynamicSalt 保证同一设备的每次升级使用不同密钥。这就是**设备绑定加密**的核心原理。

---

## 五、AES-256 加密模块

### 5.1 支持的加密模式

| 模式 | 特点                     | 适用场景               |
| ---- | ------------------------ | ---------------------- |
| CBC  | 需 IV，并行解密          | 默认推荐               |
| ECB  | 无 IV，相同明文→相同密文 | 不推荐（仅兼容旧方案） |
| CTR  | 流式加密，无需填充       | 高性能场景             |
| CFB  | 流式加密，自同步         | 误码容忍场景           |
| OFB  | 流式加密，无错误传播     | 噪声信道场景           |

### 5.2 加密流程

```python
# 加密前先追加 SHA-256 哈希
firmware_hash = SHA256.new(firmware_data).digest()
data_to_encrypt = firmware_data + firmware_hash  # 原始固件 + 32字节哈希

# PKCS7 填充后加密
cipher = AES.new(key, AES.MODE_CBC, iv=iv)
ciphertext = cipher.encrypt(pad(data_to_encrypt, AES.block_size))
```

### 5.3 解密与校验

```python
# 解密
cipher = AES.new(key, AES.MODE_CBC, iv=iv)
plaintext = unpad(cipher.decrypt(ciphertext), AES.block_size)

# 分离固件和哈希
firmware = plaintext[:-32]
embedded_hash = plaintext[-32:]

# 校验完整性
computed_hash = SHA256.new(firmware).digest()
assert computed_hash == embedded_hash, "固件完整性校验失败"
```

---

## 六、Ed25519 数字签名

### 6.1 签名流程

```python
# 密钥对生成
key = ECC.generate(curve='ed25519')

# 签名（SHA-512 预哈希，RFC8032 变体）
signer = eddsa.new(key, 'rfc8032')
signature = signer.sign(data)  # 输出 64 字节签名

# 验证
verifier = eddsa.new(key, 'rfc8032')
verifier.verify(signature, data)
```

### 6.2 密钥格式支持

| 格式     | 用途             |
| -------- | ---------------- |
| PEM      | 标准存储格式     |
| 原始字节 | 嵌入式端使用     |
| 十六进制 | 调试显示         |
| C 数组   | 直接嵌入固件源码 |

> [!TIP]
> Ed25519 的公钥只有 32 字节，签名只有 64 字节，非常适合资源受限的嵌入式场景。相比 RSA-2048（256 字节签名），Ed25519 在安全性和效率上都有显著优势。

---

## 七、串口终端与 YMODEM 传输

### 7.1 VT100 终端仿真

基于 pyte 实现 VT100 终端解析：

```python
class PyteTerminal:
    def __init__(self, cols=80, rows=24):
        self.screen = pyte.Screen(cols, rows)
        self.stream = pyte.Stream(self.screen)

    def feed(self, data):
        self.stream.feed(data)  # 解析 ANSI 转义序列

    def get_display(self):
        return self.screen.display  # 获取渲染后的文本行
```

`TerminalTextEdit` 实现了完整的终端模式：支持光标定位、颜色渲染（使用格式化缓存优化性能）、滚动区域管理。

### 7.2 YMODEM 固件传输

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

### 7.3 调试器自动发现

串口模块通过 USB VID/PID 自动识别 JLink/STLink/DAPLink，并与 PyOCD 模块联动，减少用户手动配置。

---

## 八、PyOCD SWD 烧录

通过 ARM SWD 接口直接烧录 Flash，不经过 Bootloader：

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

| 特性            | 说明                               |
| --------------- | ---------------------------------- |
| 探针自动刷新    | 5 秒间隔，变更检测                 |
| CMSIS Pack 支持 | 扫描 .pack/.pdsc 获取目标 MCU 定义 |
| 默认目标        | `stm32f407vgtx`                    |
| 擦除模式        | 全片擦除 / 扇区擦除 / 不擦除       |
| 连接模式        | halt / attach / pre-reset          |

---

## 九、双差分引擎

同时集成 bsdiff4 和 HPatchLite 两种差分引擎，为不同场景提供选择。

### 9.1 引擎对比

| 特性     | bsdiff4          | HPatchLite                |
| -------- | ---------------- | ------------------------- |
| 类型     | Python 库        | 外部可执行文件            |
| 压缩算法 | bzip2            | tuz / zlib / lzma / lzma2 |
| 集成方式 | `import bsdiff4` | 子进程调用                |
| 并行支持 | 无               | 支持多线程                |
| 原地补丁 | 不支持           | 支持                      |
| 适用场景 | 简单快速         | 高级选项、嵌入式端兼容    |

### 9.2 差分升级效果

假设旧固件 384KB，新固件 386KB，差分文件可能只有 5-20KB。在带宽受限的场景下，差分升级能显著减少传输时间和流量费用。

---

## 十、支付宝沙箱与自定义串口协议

### 10.1 帧格式

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

| 方向     | CMD  | 说明           |
| -------- | ---- | -------------- |
| PC→STM32 | 0x01 | 发送二维码 URL |
| PC→STM32 | 0x02 | 支付状态通知   |
| PC→STM32 | 0x03 | 发送交易号     |
| PC→STM32 | 0x04 | 心跳响应       |
| STM32→PC | 0x81 | 请求二维码     |
| STM32→PC | 0x82 | 查询支付状态   |
| STM32→PC | 0x83 | 关闭订单       |
| STM32→PC | 0x84 | 心跳           |

> [!NOTE]
> 命令编码规则：`0x0x` 为 PC→STM32，`0x8x` 为 STM32→PC，最高位区分方向。接收线程 `SerialReceiveThread` 解析 `0xAA 0x55` 帧头，提取命令码和数据载荷。

---

## 十一、HTML UI 提取（LVGL 资源）

通过 Playwright（Chromium headless）自动截图，将 HTML/CSS 设计稿转换为 LVGL 嵌入式显示资源：

1. 加载 HTML 文件到 headless 浏览器
2. 截取指定区域/元素
3. 转换为 C 数组格式的图像数据
4. 输出为 LVGL 兼容的 `.c` / `.h` 文件

---

## 十二、安全链路全景

**上位机构建端：**

1. 原始固件 → SHA-256 哈希追加
2. → HKDF 派生 AES 密钥（DynamicSalt + DevKey + UID）
3. → AES-256 加密
4. → 构建 Header + HMAC-SHA256 校验
5. → Ed25519 签名
6. → `.iap.bin`

**设备端验证（STM32 Bootloader）：**

1. `.iap.bin` → HMAC-SHA256 验证 Header（DevKey）
2. → 硬件兼容检查
3. → 防回滚检查（安全计数器）
4. → HKDF 派生 AES 密钥（Salt + DevKey + UID）
5. → AES 解密 → 写入 Flash
6. → SHA-512 流式哈希 → Ed25519 签名验证
7. → 回读 Flash SHA-256 校验

**安全分层防御：**

| 层         | 机制             | 解决的问题     |
| ---------- | ---------------- | -------------- |
| 真实性     | Ed25519 签名     | 谁签的？       |
| 头完整性   | HMAC-SHA256      | 头被篡改？     |
| 机密性     | AES-256 加密     | 内容保密       |
| 载荷完整性 | SHA-256 哈希     | 数据正确？     |
| 防回滚     | Security Counter | 旧版本？       |
| 密钥隔离   | HKDF 派生        | 同设备不同密钥 |
| 芯片绑定   | UID 绑定         | 密钥与芯片关联 |

---

## 十三、Nuitka 打包部署

项目使用 Nuitka 将 Python 应用编译为独立的 Windows 可执行文件，无需用户安装 Python 环境。

### 13.1 构建命令

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

### 13.2 参数说明

| 参数                             | 说明                                               |
| -------------------------------- | -------------------------------------------------- |
| `--standalone`                   | 生成独立可执行文件，不依赖本机 Python              |
| `--onefile-no-compression`       | 单文件输出，不压缩（启动更快）                     |
| `--lto=yes`                      | 启用链接时优化（Link Time Optimization），减小体积 |
| `--show-memory`                  | 显示内存使用情况                                   |
| `--show-progress`                | 显示编译进度                                       |
| `--windows-console-mode=disable` | 隐藏控制台窗口（GUI 应用）                         |
| `--enable-plugin=pyqt6`          | 启用 PyQt6 插件支持                                |
| `--enable-plugin=upx`            | 启用 UPX 压缩进一步减小体积                        |
| `--output-dir=out`               | 输出目录                                           |
| `--include-data-dir`             | 包含资源目录（图标、翻译等）                       |
| `--include-data-file`            | 包含单个数据文件（配置、外部工具）                 |
| `--windows-icon-from-ico`        | 设置应用图标                                       |

### 13.3 PyQt6 打包注意事项

> [!WARNING]
> PyQt6 应用的 Nuitka 打包有几个常见陷阱：
>
> 1. **必须使用 `--enable-plugin=pyqt6`**：否则 Nuitka 无法自动发现 PyQt6 的隐式导入和插件依赖（如平台插件 `qwindows.dll`、图像格式插件等）
> 2. **qfluentwidgets 资源**：qfluentwidgets 的 QRC 资源文件需要通过 `--include-data-dir` 显式包含，否则运行时图标和样式会丢失
> 3. **PyQt6 平台插件**：如果运行时出现 `This application failed to start because no Qt platform plugin could be initialized`，需要确保 `platforms/qwindows.dll` 被正确包含，`--enable-plugin=pyqt6` 通常会自动处理
> 4. **UPX 兼容性**：某些 PyQt6 的 DLL 被 UPX 压缩后可能无法正常加载，如果出现运行时错误，尝试移除 `--enable-plugin=upx`

### 13.4 外部工具嵌入

HPatchLite 的差分工具（`hdiffi.exe` / `hpatchi.exe`）作为外部可执行文件嵌入：

```bash
--include-data-file=hpatchlite_tools/hdiffi.exe=hpatchlite_tools/hdiffi.exe
--include-data-file=hpatchlite_tools/hpatchi.exe=hpatchlite_tools/hpatchi.exe
```

运行时通过 `sys._MEIPASS`（Nuitka standalone 模式下为 `.dist` 目录）定位这些文件：

```python
import sys
import os

def get_resource_path(relative_path):
    """获取资源文件的绝对路径（兼容开发环境和打包后环境）"""
    if hasattr(sys, '_MEIPASS'):
        # PyInstaller 打包模式
        return os.path.join(sys._MEIPASS, relative_path)
    elif getattr(sys, 'frozen', False):
        # Nuitka standalone 模式
        return os.path.join(os.path.dirname(sys.executable), relative_path)
    else:
        # 开发环境
        return os.path.join(os.path.dirname(__file__), relative_path)
```

---

## 十四、配置系统

基于 qfluentwidgets 的 `QConfig` 框架，支持持久化存储：

| 配置项              | 分组       | 默认值   | 说明            |
| ------------------- | ---------- | -------- | --------------- |
| `serialFontSize`    | Serial     | 12       | 终端字号        |
| `serialFontFamily`  | Serial     | Consolas | 终端字体        |
| `serialDTR`         | Serial     | True     | DTR 信号        |
| `serialRTS`         | Serial     | True     | RTS 信号        |
| `pyocdFirmwarePath` | PyOCD      | ""       | 固件路径        |
| `pyocdTrustCRC`     | PyOCD      | True     | 信任 CRC        |
| `cmPackPath`        | PyOCD      | ""       | CMSIS Pack 路径 |
| `language`          | MainWindow | Auto     | 界面语言        |
| `dpiScale`          | MainWindow | Auto     | DPI 缩放        |

---

## 十五、与 Bootloader 的协作关系

PyIAPToolKit 和 STM32F407 Bootloader 构成了完整的 IAP 系统：

| 环节     | 上位机（PyIAPToolKit）                | 设备端（Bootloader）                |
| -------- | ------------------------------------- | ----------------------------------- |
| 固件打包 | AES 加密 + Ed25519 签名 + Header 封装 | —                                   |
| 密钥派生 | HKDF（Salt + DevKey + UID）           | HKDF（Salt + DevKey + UID）         |
| 固件传输 | YMODEM 发送 / PyOCD 烧录              | YMODEM 接收 / SWD 写入              |
| 固件验证 | —                                     | HMAC → 防回滚 → AES 解密 → 签名验证 |
| 分区管理 | —                                     | A/B 双分区 + 自动回滚               |
| 差分升级 | bsdiff4 / HPatchLite 生成差分包       | HPatchLite 应用差分包               |

> [!TIP]
> 上位机和设备端使用**完全相同的密钥派生算法**（HKDF），确保派生出的 AES 密钥一致。DevKey 在设备端存储在 OTP 中，在上位机端由用户导入；UID 在设备端从寄存器读取，在上位机端由用户输入或从设备自动获取。

---

## 十六、总结

PyIAPToolKit 的核心亮点：

1. **完整固件打包流水线**：SHA-256 → AES-256 → HMAC → Ed25519，一站式完成
2. **设备绑定加密**：HKDF（DynamicSalt + DevKey + UID）确保固件包只能由目标设备解密
3. **双传输通道**：YMODEM 串口传输 + PyOCD SWD 烧录，覆盖开发和生产场景
4. **双差分引擎**：bsdiff4 + HPatchLite，灵活选择
5. **Fluent Design UI**：qfluentwidgets 组件库，现代化界面体验
6. **模块化架构**：10 个独立功能模块，互不耦合，易于扩展
7. **可视化对比**：颜色编码展示固件包各区域，直观清晰

---

## 参考资料

- [PyQt6 Documentation](https://www.riverbankcomputing.com/static/Docs/PyQt6/)
- [qfluentwidgets - Fluent Design for PyQt](https://qfluentwidgets.com/)
- [HKDF (RFC 5869)](https://tools.ietf.org/html/rfc5869)
- [Ed25519: High-speed high-security signatures](https://ed25519.cr.yp.to/)
- [PyOCD - Python library for programming and debugging ARM Cortex](https://pyocd.io/)
- [HPatchLite - 差分补丁库](https://github.com/sisong/HPatchLite)
- [YMODEM Protocol](https://en.wikipedia.org/wiki/YMODEM)
