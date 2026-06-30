---
title: "PyIAPToolKit: Design and Implementation of an STM32 IAP Host Tool Suite"
date: 2026-06-20
description: "An STM32 IAP host tool suite built on PyQt6 + Fluent Design, covering an AES-256 encryption + Ed25519 signing firmware packaging pipeline, HKDF device-bound key derivation, YMODEM serial transmission, PyOCD SWD flashing, a dual differential engine, LVGL resource extraction, and other core technologies"
image: PyIAPToolKit.png
categories:
  - "Embedded"
tags:
  - "STM32"
  - "IAP"
  - "PyQt6"
  - "AES-256"
  - "Ed25519"
  - "YMODEM"
  - "OTA"
  - "Differential Update"
math: true
---

## Foreword

In the previous post [STM32F407 Secure Bootloader Design](../stm32f407-secure-bootloader/), we examined the device-side A/B dual-partition, encryption/signing, and multi-channel upgrade mechanisms in detail. However, a complete IAP system also requires **host-side tools** to handle firmware packaging, encryption/signing, transmission/flashing, and other steps. This article introduces **PyIAPToolKit** — an STM32 IAP host tool suite based on PyQt6 + Fluent Design, which serves as the PC-side companion tool for the Bootloader project.

> [!NOTE]
> [Download Release v1.0.0](https://github.com/Lingjia007/PyIAPToolKit/releases/tag/v1.0.0)

---

## Project Overview

### Feature Matrix

| Feature                | Description                                                          |
| ---------------------- | ------------------------------------------------------------------- |
| Serial Terminal        | Multi-tab VT100 terminal with YMODEM firmware transfer support     |
| AES Encrypt/Decrypt    | 5 modes (CBC/ECB/CTR/CFB/OFB), HKDF key derivation                 |
| Firmware Header Pack   | 64-byte binary header with version/algorithm/security counter/HMAC |
| Ed25519 Signing        | Key pair generation, signing, verification, SHA-512 pre-hash        |
| PyOCD Flashing         | SWD debug probe firmware flashing, supports CMSIS Pack targets      |
| Incremental Diff       | bsdiff4 / HPatchLite dual engine, OTA incremental updates           |
| HTML UI Extraction     | Playwright screenshots to extract LVGL embedded display resources   |
| Alipay Sandbox         | IoT vending machine payment integration, custom serial protocol with STM32 |

### Tech Stack

| Layer                | Technology Choice                              |
| ------------------- | ---------------------------------------------- |
| Programming Language | Python 3                                       |
| GUI Framework       | PyQt6                                          |
| UI Component Library | qfluentwidgets (Fluent Design style)           |
| Serial Communication | pyserial                                       |
| Terminal Emulation  | pyte (VT100 parser)                            |
| Firmware Transfer   | ymodem                                         |
| Cryptography Library | pycryptodome (AES, HMAC, Ed25519)              |
| Debug Flashing      | pyocd (ARM Cortex SWD)                         |
| Incremental Diff    | bsdiff4, hpatchlite                            |
| Browser Automation  | playwright (Chromium headless)                 |

### Project Structure

| Module              | File                                                 | LOC     | Feature                  |
| ------------------- | ---------------------------------------------------- | ------- | ------------------------ |
| Entry               | `main.py`                                            | 145     | App launch, window setup  |
| Serial Tools        | `serial_tools/serial_interface.py`                   | 2072    | VT100 terminal + YMODEM  |
| PyOCD Tools         | `pyocd_tools/pyocd_interface.py`                     | 945     | SWD flashing             |
| AES Tools           | `aes_tools/aes_interface.py`                         | 1099    | Encrypt/decrypt + HKDF   |
| BSDiff Tools        | `bsdiff_tools/bsdiff_interface.py`                   | 613     | Incremental diff         |
| HPatchLite Tools    | `hpatchlite_tools/hpatchlite_interface.py`           | 863     | Incremental diff         |
| Firmware Header Tools | `firmware_header_tools/header_interface.py`          | 2129    | Firmware packaging pipeline |
| Ed25519 Tools       | `ed25519_tools/ed25519_interface.py`                 | 936     | Digital signature        |
| HTML UI Extraction  | `html_ui_extract_tools/html_ui_extract_interface.py` | 985     | LVGL resource extraction |
| Alipay Sandbox      | `alipay_sandbox_tools/alipay_sandbox_interface.py`   | 1714    | Payment integration      |

---

## Application Architecture

### FluentWindow Sidebar Navigation

The main window uses the `FluentWindow` sidebar navigation pattern, with 10 feature pages each corresponding to an independent module:

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

### Startup Flow

1. DPI scaling initialization
2. Internationalization translation loading (zh_CN, zh_HK, en_US)
3. Splash screen display
4. Main window creation and navigation registration

### QThread + pyqtSignal Async Pattern

All time-consuming operations follow a unified async worker pattern to ensure the GUI main thread is never blocked:

```python
class XxxThread(QThread):
    progress_signal = pyqtSignal(int, str)   # progress callback
    finished_signal = pyqtSignal(bool, str)  # completion notification

    def run(self):
        # time-consuming operation
        self.progress_signal.emit(percent, message)
        self.finished_signal.emit(success, result)
```

---

## Firmware Packaging Pipeline (Core)

This is the most central feature of the entire tool suite, taking raw firmware through a complete security processing chain to generate a deployable `.iap.bin` firmware package.

### Firmware Header Structure (64 bytes)

| Offset | Field                 | Size | Description                                                                  |
| ------ | --------------------- | ---- | ---------------------------------------------------------------------------- |
| 0x00   | magic                 | 4B   | `IAP\x01` magic number                                                       |
| 0x04   | header_version        | 1B   | Header version number                                                        |
| 0x05   | firmware_version_major | 1B   | Firmware major version                                                       |
| 0x06   | firmware_version_minor | 1B   | Firmware minor version                                                       |
| 0x07   | firmware_version_patch | 1B   | Firmware patch version                                                       |
| 0x08   | total_payload_size    | 4B   | Total size of encrypted payload                                              |
| 0x0C   | image_type            | 1B   | 0x01=App, 0x02=Bootloader, 0x03=Resource                                    |
| 0x0D   | encryption_algorithm  | 1B   | 0x00=None, 0x01=AES-256-CBC, 0x02=ECB, 0x03=CTR                             |
| 0x0E   | signature_algorithm   | 1B   | 0x00=None, 0x01=Ed25519                                                      |
| 0x0F   | hardware_compatibility | 1B   | Hardware compatibility flag                                                  |
| 0x10   | security_counter      | 4B   | Security counter (anti-rollback)                                             |
| 0x14   | build_timestamp       | 4B   | Build timestamp                                                              |
| 0x18   | reserved              | 5B   | Reserved field                                                               |
| 0x1D   | header_checksum       | 32B  | HMAC-SHA256 verification (using DevKey)                                       |

### Final Firmware Package Format

| Region       | Size     | Description                                       |
| ------------ | -------- | ------------------------------------------------- |
| Header       | 64 bytes | Magic, version, encryption/signing algorithms     |
| DynamicSalt  | 16 bytes | HKDF salt, unique per firmware                    |
| IV           | 16 bytes | AES initialization vector                         |
| Ciphertext   | N bytes  | AES-256 encrypted firmware + appended SHA-256     |
| Signature   | 64 bytes | Ed25519 digital signature                         |

### Packaging Thread Core Logic

```python
class PackageThread(QThread):
    def run(self):
        # 1. Read raw firmware
        firmware_data = open(firmware_path, 'rb').read()

        # 2. Append SHA-256 hash for integrity verification
        firmware_hash = SHA256.new(firmware_data).digest()
        data_to_encrypt = firmware_data + firmware_hash

        # 3. AES-256 encryption (key derived via HKDF)
        cipher = AES.new(aes_key, AES.MODE_CBC, iv=iv)
        ciphertext = cipher.encrypt(pad(data_to_encrypt, AES.block_size))

        # 4. Build firmware header
        header = FirmwareHeader(...)
        header.header_checksum = HMAC.new(dev_key, header_prefix, SHA256).digest()

        # 5. Ed25519 signature (over header + salt + iv + ciphertext)
        signer = eddsa.new(private_key, 'rfc8032')
        signature = signer.sign(payload)

        # 6. Assemble final package
        package = header_bytes + dynamic_salt + iv + ciphertext + signature
```

> [!NOTE]
> Appending the SHA-256 hash before encryption is a key design decision. During decryption, the STM32 side first decrypts and removes padding, then verifies that the trailing 32-byte SHA-256 matches the front part of the decrypted data, achieving a **decrypt-and-verify** dual safeguard.

### Visual Comparison

Once packaging is complete, `PackCompareDialog` displays each region of the firmware package using color coding:

| Color  | Region               |
| ------ | -------------------- |
| Blue   | Header (64B)         |
| Red    | DynamicSalt (16B)    |
| Purple | IV (16B)             |
| Green  | Ciphertext (variable)|
| Orange | Signature (64B)      |

---

## HKDF Device-Bound Key Derivation

This is the core mechanism of the security system, ensuring that firmware packages can only be decrypted by the target device.

### Two-Stage HKDF

```python
def hkdf_extract(salt: bytes, ikm: bytes) -> bytes:
    """HKDF-Extract: extract a fixed-length pseudo-random key from input key material"""
    prk = HMAC.new(salt, ikm, digestmod=SHA256).digest()
    return prk

def hkdf_expand(prk: bytes, info: bytes, length: int) -> bytes:
    """HKDF-Expand: expand the pseudo-random key into output key material of the desired length"""
    hash_len = SHA256.digest_size
    n = (length + hash_len - 1) // hash_len
    okm = b''
    t = b''
    for i in range(1, n + 1):
        t = HMAC.new(prk, t + info + bytes([i]), digestmod=SHA256).digest()
        okm += t
    return okm[:length]
```

### Key Derivation Chain

1. **Extract stage**: `HMAC-SHA256(salt=DynamicSalt, ikm=DevKey)` → PRK
2. **Expand stage**: `HMAC-SHA256(PRK, info=UID || counter)` → 32-byte AES key

Roles of the three security elements:

| Element              | Source                  | Role                                                              |
| -------------------- | ----------------------- | ----------------------------------------------------------------- |
| DevKey (128-bit)     | STM32 OTP fuses         | Device key, unreadable                                            |
| UID (96-bit)         | STM32 chip unique ID    | info parameter for HKDF-Expand, binding the key to the chip       |
| DynamicSalt (128-bit)| Randomly generated per firmware version | Ensures different encryption keys across firmware versions on the same device |

> [!IMPORTANT]
> All three elements are indispensable: DevKey ensures only legitimate devices can decrypt, UID ensures the firmware package can only be decrypted by a specific chip, and DynamicSalt ensures each upgrade on the same device uses a different key. This is the core principle of **device-bound encryption**.

---

## AES-256 Encryption Module

### Supported Encryption Modes

| Mode | Characteristics                          | Use Case                          |
| ---- | ---------------------------------------- | --------------------------------- |
| CBC  | Requires IV, parallel decryption         | Recommended default               |
| ECB  | No IV, identical plaintext → identical ciphertext | Not recommended (legacy compat only) |
| CTR  | Stream cipher, no padding                | High-performance scenarios        |
| CFB  | Stream cipher, self-synchronizing        | Error-tolerant scenarios          |
| OFB  | Stream cipher, no error propagation      | Noisy channel scenarios           |

### Encryption Flow

```python
# Append SHA-256 hash before encryption
firmware_hash = SHA256.new(firmware_data).digest()
data_to_encrypt = firmware_data + firmware_hash  # raw firmware + 32-byte hash

# Encrypt after PKCS7 padding
cipher = AES.new(key, AES.MODE_CBC, iv=iv)
ciphertext = cipher.encrypt(pad(data_to_encrypt, AES.block_size))
```

### Decryption and Verification

```python
# Decrypt
cipher = AES.new(key, AES.MODE_CBC, iv=iv)
plaintext = unpad(cipher.decrypt(ciphertext), AES.block_size)

# Separate firmware and hash
firmware = plaintext[:-32]
embedded_hash = plaintext[-32:]

# Verify integrity
computed_hash = SHA256.new(firmware).digest()
assert computed_hash == embedded_hash, "Firmware integrity check failed"
```

---

## Ed25519 Digital Signature

### Signing Flow

```python
# Key pair generation
key = ECC.generate(curve='ed25519')

# Signing (SHA-512 pre-hash, RFC8032 variant)
signer = eddsa.new(key, 'rfc8032')
signature = signer.sign(data)  # outputs a 64-byte signature

# Verification
verifier = eddsa.new(key, 'rfc8032')
verifier.verify(signature, data)
```

### Supported Key Formats

| Format     | Use Case                          |
| ---------- | --------------------------------- |
| PEM        | Standard storage format           |
| Raw bytes  | Used on the embedded side          |
| Hex        | Debug display                     |
| C array    | Direct embedding in firmware source |

> [!TIP]
> Ed25519 public keys are only 32 bytes and signatures only 64 bytes, making it well suited for resource-constrained embedded scenarios. Compared to RSA-2048 (256-byte signatures), Ed25519 offers significant advantages in both security and efficiency.

---

## Serial Terminal and YMODEM Transfer

### VT100 Terminal Emulation

VT100 terminal parsing is implemented based on pyte:

```python
class PyteTerminal:
    def __init__(self, cols=80, rows=24):
        self.screen = pyte.Screen(cols, rows)
        self.stream = pyte.Stream(self.screen)

    def feed(self, data):
        self.stream.feed(data)  # parse ANSI escape sequences

    def get_display(self):
        return self.screen.display  # retrieve rendered text lines
```

`TerminalTextEdit` implements a full terminal mode: cursor positioning, color rendering (using a formatted cache for performance optimization), and scroll region management.

### YMODEM Firmware Transfer

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

### Debugger Auto-Discovery

The serial module automatically recognizes JLink/STLink/DAPLink via USB VID/PID and integrates with the PyOCD module, reducing manual user configuration.

---

## PyOCD SWD Flashing

Flashes the Flash memory directly through the ARM SWD interface, bypassing the Bootloader:

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

| Feature             | Description                                       |
| ------------------- | ------------------------------------------------- |
| Probe auto-refresh  | 5-second interval, change detection               |
| CMSIS Pack support  | Scans .pack/.pdsc to obtain target MCU definitions |
| Default target      | `stm32f407vgtx`                                   |
| Erase modes         | Full-chip erase / sector erase / no erase         |
| Connect modes       | halt / attach / pre-reset                         |

---

## Dual Differential Engine

Integrates both bsdiff4 and HPatchLite differential engines, providing options for different scenarios.

### Engine Comparison

| Feature           | bsdiff4          | HPatchLite                |
| ----------------- | ---------------- | ------------------------- |
| Type              | Python library   | External executable       |
| Compression algo  | bzip2            | tuz / zlib / lzma / lzma2 |
| Integration       | `import bsdiff4` | Subprocess call           |
| Parallel support  | None             | Multi-threaded            |
| In-place patching | Not supported    | Supported                 |
| Use case          | Simple and fast  | Advanced options, embedded-side compatibility |

### Differential Upgrade Effect

Assume the old firmware is 384KB and the new firmware is 386KB; the diff file may only be 5-20KB. In bandwidth-constrained scenarios, differential upgrades can significantly reduce transfer time and data costs.

---

## Alipay Sandbox and Custom Serial Protocol

### Frame Format

```
0xAA 0x55 | CMD(1B) | LEN(2B, big-endian) | DATA(NB) | 0x0D 0x0A
```

```python
def _build_frame(self, cmd, data):
    header = bytes([0xAA, 0x55])
    tail = bytes([0x0D, 0x0A])
    length = struct.pack('>H', len(data))
    return header + bytes([cmd]) + length + data + tail
```

### Command Set

| Direction  | CMD  | Description           |
| ---------- | ---- | --------------------- |
| PC→STM32   | 0x01 | Send QR code URL      |
| PC→STM32   | 0x02 | Payment status notification |
| PC→STM32   | 0x03 | Send transaction ID   |
| PC→STM32   | 0x04 | Heartbeat response    |
| STM32→PC   | 0x81 | Request QR code       |
| STM32→PC   | 0x82 | Query payment status  |
| STM32→PC   | 0x83 | Close order           |
| STM32→PC   | 0x84 | Heartbeat             |

> [!NOTE]
> Command encoding rule: `0x0x` is PC→STM32, `0x8x` is STM32→PC; the highest bit distinguishes direction. The receive thread `SerialReceiveThread` parses the `0xAA 0x55` frame header and extracts the command code and data payload.

---

## HTML UI Extraction (LVGL Resources)

Automatically captures screenshots via Playwright (Chromium headless) to convert HTML/CSS designs into LVGL embedded display resources:

1. Load the HTML file into the headless browser
2. Capture the specified region/element
3. Convert to C-array-format image data
4. Output LVGL-compatible `.c` / `.h` files

---

## End-to-End Security Chain Overview

**Host-side build stage:**

1. Raw firmware → SHA-256 hash appended
2. → HKDF derives AES key (DynamicSalt + DevKey + UID)
3. → AES-256 encryption
4. → Build Header + HMAC-SHA256 verification
5. → Ed25519 signature
6. → `.iap.bin`

**Device-side verification (STM32 Bootloader):**

1. `.iap.bin` → HMAC-SHA256 verifies Header (DevKey)
2. → Hardware compatibility check
3. → Anti-rollback check (security counter)
4. → HKDF derives AES key (Salt + DevKey + UID)
5. → AES decrypt → write to Flash
6. → SHA-512 streaming hash → Ed25519 signature verification
7. → Read-back Flash SHA-256 verification

**Layered security defense:**

| Layer            | Mechanism         | Problem Solved              |
| ---------------- | ----------------- | --------------------------- |
| Authenticity     | Ed25519 signature | Who signed it?              |
| Header integrity | HMAC-SHA256       | Header tampered?            |
| Confidentiality  | AES-256 encryption| Content confidentiality     |
| Payload integrity| SHA-256 hash      | Is data correct?            |
| Anti-rollback    | Security Counter  | Old version?                |
| Key isolation    | HKDF derivation   | Different keys on same device |
| Chip binding     | UID binding       | Key tied to chip            |

---

## Nuitka Packaging and Deployment

The project uses Nuitka to compile the Python application into a standalone Windows executable, eliminating the need for users to install a Python environment.

### Build Command

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

### Parameter Description

| Parameter                         | Description                                                                |
| --------------------------------- | -------------------------------------------------------------------------- |
| `--standalone`                    | Generate standalone executable, independent of local Python                 |
| `--onefile-no-compression`        | Single-file output, no compression (faster startup)                        |
| `--lto=yes`                       | Enable Link Time Optimization, reducing size                               |
| `--show-memory`                   | Show memory usage                                                          |
| `--show-progress`                 | Show compile progress                                                      |
| `--windows-console-mode=disable`  | Hide console window (GUI app)                                              |
| `--enable-plugin=pyqt6`           | Enable PyQt6 plugin support                                                |
| `--enable-plugin=upx`             | Enable UPX compression to further reduce size                              |
| `--output-dir=out`                | Output directory                                                           |
| `--include-data-dir`              | Include resource directory (icons, translations, etc.)                     |
| `--include-data-file`             | Include individual data files (config, external tools)                     |
| `--windows-icon-from-ico`         | Set application icon                                                       |

### PyQt6 Packaging Notes

> [!WARNING]
> Nuitka packaging for PyQt6 applications has several common pitfalls:
>
> 1. **You must use `--enable-plugin=pyqt6`**: otherwise Nuitka cannot automatically discover PyQt6's implicit imports and plugin dependencies (such as the platform plugin `qwindows.dll`, image format plugins, etc.)
> 2. **qfluentwidgets resources**: qfluentwidgets' QRC resource files must be explicitly included via `--include-data-dir`, otherwise icons and styles will be missing at runtime
> 3. **PyQt6 platform plugin**: if you encounter `This application failed to start because no Qt platform plugin could be initialized` at runtime, ensure that `platforms/qwindows.dll` is properly included; `--enable-plugin=pyqt6` usually handles this automatically
> 4. **UPX compatibility**: some PyQt6 DLLs may fail to load properly after UPX compression; if runtime errors occur, try removing `--enable-plugin=upx`

### External Tool Embedding

HPatchLite's diff tools (`hdiffi.exe` / `hpatchi.exe`) are embedded as external executables:

```bash
--include-data-file=hpatchlite_tools/hdiffi.exe=hpatchlite_tools/hdiffi.exe
--include-data-file=hpatchlite_tools/hpatchi.exe=hpatchlite_tools/hpatchi.exe
```

At runtime, these files are located via `sys._MEIPASS` (the `.dist` directory in Nuitka standalone mode):

```python
import sys
import os

def get_resource_path(relative_path):
    """Get the absolute path of a resource file (compatible with both dev and packaged environments)"""
    if hasattr(sys, '_MEIPASS'):
        # PyInstaller packaging mode
        return os.path.join(sys._MEIPASS, relative_path)
    elif getattr(sys, 'frozen', False):
        # Nuitka standalone mode
        return os.path.join(os.path.dirname(sys.executable), relative_path)
    else:
        # Development environment
        return os.path.join(os.path.dirname(__file__), relative_path)
```

---

## Configuration System

Based on qfluentwidgets' `QConfig` framework, supports persistent storage:

| Config Item          | Group       | Default  | Description          |
| -------------------- | ----------- | -------- | -------------------- |
| `serialFontSize`     | Serial      | 12       | Terminal font size   |
| `serialFontFamily`   | Serial      | Consolas | Terminal font        |
| `serialDTR`          | Serial      | True     | DTR signal           |
| `serialRTS`          | Serial      | True     | RTS signal           |
| `pyocdFirmwarePath`  | PyOCD       | ""       | Firmware path        |
| `pyocdTrustCRC`      | PyOCD       | True     | Trust CRC            |
| `cmPackPath`         | PyOCD       | ""       | CMSIS Pack path      |
| `language`           | MainWindow  | Auto     | UI language          |
| `dpiScale`           | MainWindow  | Auto     | DPI scaling          |

---

## Collaboration with the Bootloader

PyIAPToolKit and the STM32F407 Bootloader form a complete IAP system:

| Stage              | Host (PyIAPToolKit)                       | Device (Bootloader)                       |
| ------------------ | ----------------------------------------- | ----------------------------------------- |
| Firmware packaging | AES encryption + Ed25519 signing + Header | —                                         |
| Key derivation     | HKDF (Salt + DevKey + UID)                | HKDF (Salt + DevKey + UID)                |
| Firmware transfer  | YMODEM send / PyOCD flash                 | YMODEM receive / SWD write                |
| Firmware verification | —                                      | HMAC → anti-rollback → AES decrypt → signature verification |
| Partition management | —                                       | A/B dual partition + auto rollback         |
| Differential update | bsdiff4 / HPatchLite generate diff pack  | HPatchLite apply diff pack                |

> [!TIP]
> The host and device sides use the **exact same key derivation algorithm** (HKDF) to ensure the derived AES keys match. On the device side, DevKey is stored in OTP; on the host side, it is imported by the user. UID is read from a register on the device side, and entered by the user or auto-fetched from the device on the host side.

---

## Summary

Core highlights of PyIAPToolKit:

1. **Complete firmware packaging pipeline**: SHA-256 → AES-256 → HMAC → Ed25519, all in one place
2. **Device-bound encryption**: HKDF (DynamicSalt + DevKey + UID) ensures firmware packages can only be decrypted by the target device
3. **Dual transmission channels**: YMODEM serial transfer + PyOCD SWD flashing, covering both development and production scenarios
4. **Dual differential engine**: bsdiff4 + HPatchLite, flexible choice
5. **Fluent Design UI**: qfluentwidgets component library, modern interface experience
6. **Modular architecture**: 10 independent feature modules, decoupled and easy to extend
7. **Visual comparison**: color-coded display of each firmware package region, intuitive and clear

---

## References

- [PyQt6 Documentation](https://www.riverbankcomputing.com/static/Docs/PyQt6/)
- [qfluentwidgets - Fluent Design for PyQt](https://qfluentwidgets.com/)
- [HKDF (RFC 5869)](https://tools.ietf.org/html/rfc5869)
- [Ed25519: High-speed high-security signatures](https://ed25519.cr.yp.to/)
- [PyOCD - Python library for programming and debugging ARM Cortex](https://pyocd.io/)
- [HPatchLite - Diff Patch Library](https://github.com/sisong/HPatchLite)
- [YMODEM Protocol](https://en.wikipedia.org/wiki/YMODEM)
