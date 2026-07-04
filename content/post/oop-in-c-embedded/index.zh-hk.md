---
title: "C 語言的精髓：結構體與指標——嵌入式面向物件平台化分層設計"
date: 2026-07-03
description: "從 struct + me 指標出發，經資料歸位、繼承嵌套、操作表、虛指標、多態分發、container_of、平台抽象層到 Linux 內核風格，用一顆 LED 講透嵌入式 C 面向物件編程的全部精髓"
categories:
  - "嵌入式"
tags:
  - "C語言"
  - "面向物件"
  - "OOP-in-C"
  - "嵌入式"
  - "平台抽象"
  - "Linux內核"
math: true
---

## 前言

C 語言沒有 `class`、沒有 `virtual`、沒有 `interface`，但 Linux 內核、GObject、Zephyr RTOS 這些大型專案全用 C 寫出了完整的面向物件體系。它們的武器只有兩件：**結構體**和**指標**。

本文以一個嵌入式 LED 驅動的漸進式重構為主線，從最樸素的 `struct + me` 出發，一路走到 Linux 內核風格的 `gpio_chip` + initcall 自動註冊，用同一顆 LED 講透嵌入式 C 面向物件編程的全部精髓。

---

## 一、封裝的起點：struct + me 指標

### 1.1 最樸素的形態

```c
// led.h
struct led {
    uint8_t pin;
    uint8_t brightness;
    bool    is_on;
};

int  led_init(struct led *me, uint8_t pin);
void led_on(struct led *me);
void led_off(struct led *me);
void led_toggle(struct led *me);
```

每個函式第一個參數是 `struct led *me`——這就是 C++ 隱藏的 `this` 指標。

### 1.2 多實例共享同一份程式碼

```c
// main.c
struct led red_led, green_led, blue_led;

led_init(&red_led, 5);
led_init(&green_led, 3);
led_init(&blue_led, 7);

led_on(&red_led);    // 點亮紅燈
led_on(&green_led);  // 點亮綠燈
```

三顆 LED 共用同一份 `led_on` 程式碼，函式透過 `me` 指標知道操作哪一顆。想加第 100 顆 LED，多開一個 `struct led` 實例即可，函式一行不用動。

### 1.3 C++ 等價對照

| C 寫法             | C++ 等價       |
| ------------------ | -------------- |
| `struct led`       | `class Led`    |
| `led_on(&red_led)` | `red_led.on()` |
| `struct led *me`   | 隱式 `this`    |
| `me->pin`          | `this->pin`    |

---

## 二、手搓 class：函式前綴 + init/deinit

### 2.1 命名規範 = namespace

```c
// led.h — LED 類
int  led_init(struct led *me, uint8_t pin);
void led_deinit(struct led *me);
void led_on(struct led *me);

// motor.h — Motor 類
int  motor_init(struct motor *me, uint8_t pin);
void motor_deinit(struct motor *me);
void motor_start(struct motor *me);
```

`led_xxx` 前綴保證鏈結器符號唯一。不加前綴的話 `led.c` 寫 `void init()`，`motor.c` 也寫 `void init()`，立刻多重定義衝突。C++ 的 namespace + name mangling 做的就是同一件事，編譯器替你操作字串，C 裡手寫。

### 2.2 建構函式的三件事

```c
int led_init(struct led *me, uint8_t pin) {
    if (!me || !pin_valid(pin)) return -1;  // 1. 參數校驗
    me->pin = pin;
    me->is_on = false;
    platform_gpio_init(pin, GPIO_OUTPUT);    // 2. 硬體初始化
    platform_gpio_write(pin, false);         // 3. 預設狀態
    me->initialized = true;
    return 0;
}
```

### 2.3 const me = 唯讀語義

```c
int led_get_state(const struct led *me, bool *is_on);
```

`const` 修飾 `me` 保證查詢函式不會修改物件狀態，編譯器幫你在編譯期擋住誤寫。

---

## 三、資料三級歸位：資訊隱藏的完成形態

### 3.1 三類資料的歸屬

| 類別         | 存放位置       | 生命週期     | 可見性                |
| ------------ | -------------- | ------------ | --------------------- |
| 實例資料     | `struct` 欄位  | 跟著 `me` 走 | `.h` 暴露或 `.c` 隱藏 |
| 模組共享資料 | `static` 變數  | 程式全程     | 僅本 `.c` 檔案        |
| 唯讀常量     | `static const` | 編譯期確定   | 僅本 `.c` 檔案        |

```c
// led.c
static const int MAX_BRIGHTNESS = 100;        // 第三類：唯讀常量
static struct led led_pool[8];                 // 第二類：模組共享（物件池）
static int s_init_count = 0;                   // 第二類：模組共享（計數器）

struct led *led_acquire(uint8_t pin) {         // 工廠：從池中分配
    for (int i = 0; i < 8; i++) {
        if (!led_pool[i].in_use) {
            led_pool[i].in_use = true;
            led_init(&led_pool[i], pin);
            s_init_count++;
            return &led_pool[i];
        }
    }
    return NULL;
}
```

**靜態物件池工廠**：MCU 上零堆碎片、O(1) 分配，比 `malloc` 安全得多。

### 3.2 反面教材：全域變數的災難

```c
// led_bad.c — 千萬別這樣寫！
int g_pin;           // 實例資料用全域——最致命
int g_brightness;    // 誰都能改，沒有主人
int init_count;      // 沒有 static，外部 extern 能改
```

`bad_led_init(5)` 設 `g_pin=5`，隨後 `bad_led_init(3)` 設 `g_pin=3`——紅燈的 pin 被覆蓋，程式碼不報錯，邏輯全錯，查一週也查不出來。

**資料沒有主人，bug 就是主人。**

---

## 四、繼承：struct 嵌套

### 4.1 提公因式

八種 LED 都有 `name + is_on`，抄八遍不如提到 base 裡寫一份：

```c
// led_base.h
struct led_base {
    const char *name;
    bool is_on;
};

// led_gpio.h
struct led_gpio {
    struct led_base base;   // 基類放第一個欄位！
    uint8_t pin;            // GPIO 特有
};

// led_pwm.h
struct led_pwm {
    struct led_base base;   // 基類放第一個欄位！
    uint8_t channel;        // PWM 特有
    uint8_t duty;
};
```

**基類放第一個欄位**是硬規則——C11 標準 6.7.2.1 節保證 struct 第一個成員偏移量為 0，因此 `&gpio->base` 的位址 == `gpio` 本身的位址。Linux 內核、Zephyr、GObject 全部遵循。

### 4.2 建構函式鏈

```c
int led_gpio_init(struct led_gpio *me, const char *name, uint8_t pin) {
    led_base_init(&me->base, name);   // 先呼叫基類建構
    me->pin = pin;                     // 再處理子類欄位
    return 0;
}
```

C++ 的 `led_gpio g;` 編譯器自動先呼叫 base 建構函式再呼叫 derived 建構函式，就是這裡手寫的事情。

---

## 五、操作表：函式指標打包

### 5.1 ops 結構體 = vtable 的雛形

```c
// led_base.h
typedef int (*led_action_fn)(struct led_base *me);

struct led_ops {
    led_action_fn on;
    led_action_fn off;
    led_action_fn toggle;
};
```

將同類 LED 的可變行為打包為一張函式指標表，呼叫方按名字 `ops->on` 存取，不再按位置傳參。

### 5.2 子類填表

```c
// led_gpio.c
static int gpio_on(struct led_base *me) {
    struct led_gpio *self = (struct led_gpio *)me;
    platform_gpio_write(self->pin, true);
    self->base.is_on = true;
    return 0;
}

const struct led_ops led_ops_gpio = {
    .on     = gpio_on,
    .off    = gpio_off,
    .toggle = gpio_toggle,
};
```

100 顆 GPIO LED 共享同一張 12 位元組的 ops 表，`const` 使其落入 `.rodata` 段，執行時不可改寫。

---

## 六、虛指標：ops 欄位進 base

### 6.1 從外部參數變成內建欄位

```c
// ch09: ops 作為參數傳入
test_led(&red_led.base, &led_ops_gpio);

// ch10: ops 跟著物件自己跑
struct led_base {
    const struct led_ops *ops;   // 虛指標
    const char *name;
    bool is_on;
};

test_led(&red_led.base);  // 只需一個參數！
```

呼叫方不再需要記住「紅燈用 gpio 表、藍燈用 pwm 表」，徹底消除傳錯 ops 的風險。

### 6.2 C++ vptr 對照

| C 寫法                      | C++ 等價    |
| --------------------------- | ----------- |
| `me->ops`                   | 隱式 vptr   |
| `me->ops->on(me)`           | 虛函式呼叫  |
| `const struct led_ops *ops` | vtable 指標 |

---

## 七、多態：父類統一介面

### 7.1 膠水函式

```c
// led_base.c
int led_on(struct led_base *me) {
    if (!me || !me->ops || !me->ops->on) return -1;
    return me->ops->on(me);     // 一行 dispatch
}
```

應用層只呼叫 `led_on(base)`，不再寫 `me->ops->on(me)` 長串。

### 7.2 執行時多態

```c
// main.c
struct led_base *all_leds[] = {
    &red_led.base,    // GPIO LED
    &blue_led.base,   // PWM LED
    &green_led.base,  // I2C LED
};

for (int i = 0; i < 3; i++)
    led_on(all_leds[i]);   // 同一行程式碼，三種行為
```

三種 LED 透過向上轉型裝進同一陣列，統一遍歷呼叫——執行時多態的完整形態。

---

## 八、向上轉型與板級封裝

### 8.1 子類 static 隱藏 + 全域 base 句柄

```c
// led_board_init.c (BSP 層)
static struct led_gpio  s_led_err;       // 子類物件，檔案私有
static struct led_pwm   s_led_status;    // 外部不可見

struct led_base *g_led_error;            // 全域基類指標句柄
struct led_base *g_led_status;

void led_board_init(void) {
    led_gpio_init(&s_led_err, "error", 5);
    led_pwm_init(&s_led_status, "status", 0, 50);

    g_led_error  = &s_led_err.base;      // 向上轉型
    g_led_status = &s_led_status.base;   // 向上轉型
}
```

**板級封裝的價值**：

- 應用層只看 `struct led_base *`，不知道背後掛的是哪種 LED
- 換硬體方案（GPIO → PWM）只改本檔案，`main.c` 一字不動
- 寫 `&s_led_err.base` 而非 `(struct led_base *)&s_led_err`——欄位存取讓編譯器算偏移，安全

---

## 九、container_of：從成員反推外層

### 9.1 問題的提出

子類實作函式只收到 `struct led_base *me`，但要操作的欄位在外層子類裡。之前用 `(struct led_gpio *)me` 強轉——前提是 base 在第 0 偏移，一旦 base 換位置就崩。

### 9.2 巨集定義

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
```

三步：(1) 把 `ptr` 轉成位元組指標，(2) 減去成員偏移（`offsetof` 編譯期常量），(3) 轉回 `type *`。

### 9.3 使用

```c
static int gpio_on(struct led_base *me) {
    struct led_gpio *self = container_of(me, struct led_gpio, base);
    //                 ↑ 編譯期算偏移，執行時只剩一條 sub 指令
    platform_gpio_write(self->pin, true);
    self->base.is_on = true;
    return 0;
}
```

**base 位置不再受限**，強轉退役。這是 C 用編譯期數學解決 C++ 用 `dynamic_cast` 解決的同一個問題——零執行時開銷。

---

## 十、純虛函式與選填策略

### 10.1 兩種 ops 槽位

```c
struct led_ops {
    led_action_fn on;             // 必填
    led_action_fn off;            // 必填
    led_set_brightness_fn set_brightness;  // 選填
};
```

### 10.2 必填：assert 模擬純虛函式

```c
int led_on(struct led_base *me) {
    assert(me && me->ops && me->ops->on && "subclass must implement on()");
    return me->ops->on(me);
}
```

- Debug 建構：`assert` 觸發 abort，暴露「忘填」
- Release 建構（`-DNDEBUG`）：`assert` 消失，零執行時開銷
- 等價 C++：`virtual void on() = 0`

### 10.3 選填：NULL 預設行為

```c
int led_set_brightness(struct led_base *me, uint8_t brightness) {
    if (!me->ops->set_brightness) {
        printf("no dimming support\n");
        return 0;    // 安靜返回
    }
    return me->ops->set_brightness(me, brightness);
}
```

GPIO LED 不填 `set_brightness`，走預設行為；PWM LED 填了，走自己的實作。等價 C++ 帶預設實作的虛函式。

---

## 十一、平台抽象層：跨 MCU 移植的根基

### 11.1 三層架構

| 層級   | 檔案                           | 關鍵操作                                                         |
| ------ | ------------------------------ | ---------------------------------------------------------------- |
| 應用層 | `app.c`                        | `led_on(g_led_error)` — 只呼叫基類介面                           |
| 驅動層 | `led_gpio.c` / `led_pwm.c`     | `container_of` → `platform_gpio_write` — 只呼叫 platform 封裝    |
| 平台層 | `platform_pin.c`               | `_g_ops->write(pin, value)` — ops dispatch                       |
| 實作層 | `pin_board.c`（每種 MCU 一份） | STM32: `HAL_GPIO_WritePin` / NXP: `GPIO_PinWrite` / PC: `printf` |

### 11.2 platform 層的 ops + register 機制

```c
// platform_pin.h
struct platform_pin_ops {
    int (*mode)(int32_t pin, int32_t mode);
    int (*write)(int32_t pin, bool value);
    bool (*read)(int32_t pin);
};

int platform_pin_register(const struct platform_pin_ops *ops);  // 啟動期註冊
void platform_pin_write(int32_t pin, bool value);               // 上層呼叫
```

```c
// platform_pin.c
static const struct platform_pin_ops *_g_ops = NULL;

int platform_pin_register(const struct platform_pin_ops *ops) {
    _g_ops = ops;
    return 0;
}

void platform_pin_write(int32_t pin, bool value) {
    assert(_g_ops && _g_ops->write);
    _g_ops->write(pin, value);    // dispatch
}
```

### 11.3 跨 MCU 移植：只換實作層

```c
// STM32: platform/arch/stm32/pin_board.c
static int stm32_pin_write(int32_t pin, bool value) {
    HAL_GPIO_WritePin(GPIOA, 1 << pin, value);
    return 0;
}
const struct platform_pin_ops stm32_pin_ops = { .write = stm32_pin_write, ... };

// NXP: platform/arch/nxp/pin_board.c
static int nxp_pin_write(int32_t pin, bool value) {
    GPIO_PinWrite(GPIOA, pin, value);
    return 0;
}
const struct platform_pin_ops nxp_pin_ops = { .write = nxp_pin_write, ... };
```

**同一份 `led_gpio.c` 在 PC、STM32、NXP 上編譯執行，原始碼零修改**——這就是平台抽象層的威力。

---

## 十二、Linux 內核風格：gpio_chip

### 12.1 ops 內嵌物件 + 多實例 chip

Linux 內核允許同一 SoC 上並存多家廠商 GPIO 控制器（片內 + 外擴 IO），每個 chip 實例自帶 ops 表：

```c
// gpio_chip.h
struct gpio_chip {
    const char *label;          // "stm32-gpioa"
    int base;                   // 起始 GPIO 編號
    int ngpio;                  // 該 chip 管多少腳
    int (*request)(struct gpio_chip *gc, unsigned offset);
    void (*free)(struct gpio_chip *gc, unsigned offset);
    int (*direction_output)(struct gpio_chip *gc, unsigned offset, int value);
    int (*get)(struct gpio_chip *gc, unsigned offset);
    void (*set)(struct gpio_chip *gc, unsigned offset, int value);
    void *driver_data;         // 廠商私有上下文
};

struct gpio_desc {
    struct gpio_chip *gc;      // 反查到 chip
    unsigned int offset;       // 腳號偏移
};
```

### 12.2 desc → chip → ops dispatch

```c
// gpiolib.c
void gpiod_set_value(struct gpio_desc *desc, int value) {
    desc->gc->set(desc->gc, desc->offset, value);
    //         ↑ dispatch 到對應廠商的實作
}
```

同一行 `gpiod_set_value` 呼叫，根據 `desc->gc` 指向哪家廠商的 `gpio_chip`，落到不同實作。這就是 Linux 內核「一份 leds-gpio 驅動跑遍 N 家 SoC」的根基。

---

## 十三、initcall 自動註冊：開閉原則的終極實作

### 13.1 問題：每加一個驅動就改 main

```c
// main.c — 違反開閉原則
int main(void) {
    led_init();       // 每加一個驅動
    motor_init();     // 就要在這裡加一行
    sensor_init();    // 改完 main 還要加 #include
    uart_init();
    // ...無窮無盡
}
```

### 13.2 解決：編譯期 + 鏈結器 + 啟動程式碼三方配合

```c
// initcall.h
typedef int (*initcall_t)(void);

#define MODULE_INIT(fn) \
    static initcall_t __initcall_##fn \
    __attribute__((used, section("my_initcall"))) = fn

// 宣告鏈結器自動生成的段邊界符號
extern initcall_t __start_my_initcall[];
extern initcall_t __stop_my_initcall[];
```

```c
// drv_led.c
static int drv_led_init(void) {
    // ... LED 驅動初始化
    return 0;
}
MODULE_INIT(drv_led_init);  // 一行註冊，main 不用改
```

```c
// initcall.c
void do_initcalls(void) {
    initcall_t *fn;
    for (fn = __start_my_initcall; fn < __stop_my_initcall; fn++)
        (*fn)();
}
```

### 13.3 魔法的全部祕密

1. `__attribute__((section("my_initcall")))`：把函式指標放進自定義段
2. 鏈結器自動合併所有 `.o` 的同名段，生成 `__start_` / `__stop_` 邊界符號
3. `__attribute__((used))`：防止編譯器最佳化掉看似「死程式碼」的 static 變數
4. `do_initcalls()` 遍歷段區間，挨個呼叫

**加新驅動 = 寫新檔案 + 一行 `MODULE_INIT`**，`main.c` 一字不動——這就是 Linux 內核 `module_init` 那一行魔法的全部祕密。

---

## 十四、OOP-in-C 全景圖

| 章節         | C 寫法                     | C++ 等價              | 核心變化                               |
| ------------ | -------------------------- | --------------------- | -------------------------------------- |
| 封裝         | `struct` + `me` 指標       | `class` + `this`      | 資料與程式碼分離，多實例共享同一份函式 |
| 手搓 class   | 函式前綴 + init/deinit     | namespace + 建構/解構 | 命名規範防符號衝突，生命週期管理       |
| 資料歸位     | static / static const      | private / const       | 三類資料各歸其位，資訊隱藏完成         |
| 繼承         | struct 嵌套                | class 繼承            | 基類放第一個欄位，建構函式鏈           |
| 操作表       | ops 結構體                 | vtable 雛形           | 函式指標打包，按名存取                 |
| 虛指標       | ops 欄位進 base            | vptr                  | 物件自帶操作表，呼叫只需 base 指標     |
| 多態         | 父類統一介面               | virtual dispatch      | 同一行程式碼，不同行為                 |
| 向上轉型     | `&sub.base` 賦值           | implicit upcast       | 子類隱藏，base 句柄暴露，BSP 封裝      |
| container_of | 巨集反推外層結構體         | `dynamic_cast`        | base 位置不受限，零執行時開銷          |
| 純虛/選填    | assert / NULL 預設         | `= 0` / 預設實作      | 必填除錯期斷言，選填安靜預設           |
| 平台抽象     | HAL ops + register         | 抽象基類 + 工廠       | 跨 MCU 移植只換實作層                  |
| Linux 風格   | gpio_chip 內嵌 ops         | 多態 + 多實例         | 同一 SoC 多廠商 chip 並存              |
| initcall     | `__attribute__((section))` | 開閉原則              | 編譯期 + 鏈結器自動註冊，main 不動     |

---

## 十五、為什麼嵌入式必須用 C 寫 OOP

### 15.1 C++ 在嵌入式的現實困境

- **程式碼膨脹**：虛函式表、RTTI、異常處理增加 10-30% Flash
- **ABI 不穩定**：不同編譯器 name mangling 不相容，二進制模組不能混鏈
- **確定性缺失**：異常處理堆疊展開、placement new 失敗路徑難以預測
- **工具鏈限制**：許多 MCU 只支援 C89/C99 編譯器

### 15.2 C OOP 的工程優勢

- **零抽象開銷**：struct 嵌套 = 零偏移，container_of = 一條 sub 指令，ops 表 = 間接跳轉
- **ABI 穩定**：結構體佈局由 C ABI 決定，不同編譯器、不同語言都能互操作
- **顯式控制**：沒有隱式建構/解構，沒有隱式 this，每一行程式碼都可見
- **可審計**：所有「面向物件魔法」都是幾行巨集 + 結構體，程式碼審查時一目了然

### 15.3 一句話總結

> C 語言的面向物件不是「沒有 C++ 就湊合過」，而是嵌入式場景下的最優解——用最少的語言特性獲得最多的設計自由度，每一位元組開銷都可審計。

---

## 參考資料

- [zhaoming-embedded/oop-in-c](https://github.com/ZhaoChengBo/zhaoming-embedded) — 本文源碼倉庫
- [Linux Kernel — gpio_chip](https://www.kernel.org/doc/html/latest/driver-api/gpio/)
- [GObject Type System](https://docs.gtk.org/gobject/)
- [Zephyr RTOS Device Driver Model](https://docs.zephyrproject.org/latest/contribute/device-drivers/index.html)
- [C11 Standard — 6.7.2.1 Structure and union specifiers](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)
- [Linux Kernel container_of](https://elixir.bootlin.com/linux/latest/source/include/linux/container_of.h)
