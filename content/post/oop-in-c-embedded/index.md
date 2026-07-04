---
title: "C 语言的精髓：结构体与指针——嵌入式面向对象平台化分层设计"
date: 2026-07-03
description: "从 struct + me 指针出发，经数据归位、继承嵌套、操作表、虚指针、多态分发、container_of、平台抽象层到 Linux 内核风格，用一颗 LED 讲透嵌入式 C 面向对象编程的全部精髓"
categories:
  - "嵌入式"
tags:
  - "C语言"
  - "面向对象"
  - "OOP-in-C"
  - "嵌入式"
  - "平台抽象"
  - "Linux内核"
math: true
---

## 前言

C 语言没有 `class`、没有 `virtual`、没有 `interface`，但 Linux 内核、GObject、Zephyr RTOS 这些大型项目全用 C 写出了完整的面向对象体系。它们的武器只有两件：**结构体**和**指针**。

本文以一个嵌入式 LED 驱动的渐进式重构为主线，从最朴素的 `struct + me` 出发，一路走到 Linux 内核风格的 `gpio_chip` + initcall 自动注册，用同一颗 LED 讲透嵌入式 C 面向对象编程的全部精髓。

---

## 一、封装的起点：struct + me 指针

### 1.1 最朴素的形态

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

每个函数第一个参数是 `struct led *me`——这就是 C++ 隐藏的 `this` 指针。

### 1.2 多实例共享同一份代码

```c
// main.c
struct led red_led, green_led, blue_led;

led_init(&red_led, 5);
led_init(&green_led, 3);
led_init(&blue_led, 7);

led_on(&red_led);    // 点亮红灯
led_on(&green_led);  // 点亮绿灯
```

三颗 LED 共用同一份 `led_on` 代码，函数通过 `me` 指针知道操作哪一颗。想加第 100 颗 LED，多开一个 `struct led` 实例即可，函数一行不用动。

### 1.3 C++ 等价对照

| C 写法             | C++ 等价       |
| ------------------ | -------------- |
| `struct led`       | `class Led`    |
| `led_on(&red_led)` | `red_led.on()` |
| `struct led *me`   | 隐式 `this`    |
| `me->pin`          | `this->pin`    |

---

## 二、手搓 class：函数前缀 + init/deinit

### 2.1 命名规范 = namespace

```c
// led.h — LED 类
int  led_init(struct led *me, uint8_t pin);
void led_deinit(struct led *me);
void led_on(struct led *me);

// motor.h — Motor 类
int  motor_init(struct motor *me, uint8_t pin);
void motor_deinit(struct motor *me);
void motor_start(struct motor *me);
```

`led_xxx` 前缀保证链接器符号唯一。不加前缀的话 `led.c` 写 `void init()`，`motor.c` 也写 `void init()`，立刻多重定义冲突。C++ 的 namespace + name mangling 做的就是同一件事，编译器替你操作字符串，C 里手写。

### 2.2 构造函数的三件事

```c
int led_init(struct led *me, uint8_t pin) {
    if (!me || !pin_valid(pin)) return -1;  // 1. 参数校验
    me->pin = pin;
    me->is_on = false;
    platform_gpio_init(pin, GPIO_OUTPUT);    // 2. 硬件初始化
    platform_gpio_write(pin, false);         // 3. 默认状态
    me->initialized = true;
    return 0;
}
```

### 2.3 const me = 只读语义

```c
int led_get_state(const struct led *me, bool *is_on);
```

`const` 修饰 `me` 保证查询函数不会修改对象状态，编译器帮你在编译期挡住误写。

---

## 三、数据三级归位：信息隐藏的完成形态

### 3.1 三类数据的归属

| 类别         | 存放位置       | 生命周期     | 可见性                |
| ------------ | -------------- | ------------ | --------------------- |
| 实例数据     | `struct` 字段  | 跟着 `me` 走 | `.h` 暴露或 `.c` 隐藏 |
| 模块共享数据 | `static` 变量  | 程序全程     | 仅本 `.c` 文件        |
| 只读常量     | `static const` | 编译期确定   | 仅本 `.c` 文件        |

```c
// led.c
static const int MAX_BRIGHTNESS = 100;        // 第三类：只读常量
static struct led led_pool[8];                 // 第二类：模块共享（对象池）
static int s_init_count = 0;                   // 第二类：模块共享（计数器）

struct led *led_acquire(uint8_t pin) {         // 工厂：从池中分配
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

**静态对象池工厂**：MCU 上零堆碎片、O(1) 分配，比 `malloc` 安全得多。

### 3.2 反面教材：全局变量的灾难

```c
// led_bad.c — 千万别这样写！
int g_pin;           // 实例数据用全局——最致命
int g_brightness;    // 谁都能改，没有主人
int init_count;      // 没有 static，外部 extern 能改
```

`bad_led_init(5)` 设 `g_pin=5`，随后 `bad_led_init(3)` 设 `g_pin=3`——红灯的 pin 被覆盖，代码不报错，逻辑全错，查一周也查不出来。

**数据没有主人，bug 就是主人。**

---

## 四、继承：struct 嵌套

### 4.1 提公因式

八种 LED 都有 `name + is_on`，抄八遍不如提到 base 里写一份：

```c
// led_base.h
struct led_base {
    const char *name;
    bool is_on;
};

// led_gpio.h
struct led_gpio {
    struct led_base base;   // 基类放第一个字段！
    uint8_t pin;            // GPIO 特有
};

// led_pwm.h
struct led_pwm {
    struct led_base base;   // 基类放第一个字段！
    uint8_t channel;        // PWM 特有
    uint8_t duty;
};
```

**基类放第一个字段**是硬规则——C11 标准 6.7.2.1 节保证 struct 第一个成员偏移量为 0，因此 `&gpio->base` 的地址 == `gpio` 本身的地址。Linux 内核、Zephyr、GObject 全部遵循。

### 4.2 构造函数链

```c
int led_gpio_init(struct led_gpio *me, const char *name, uint8_t pin) {
    led_base_init(&me->base, name);   // 先调基类构造
    me->pin = pin;                     // 再处理子类字段
    return 0;
}
```

C++ 的 `led_gpio g;` 编译器自动先调 base 构造函数再调 derived 构造函数，就是这里手写的事情。

---

## 五、操作表：函数指针打包

### 5.1 ops 结构体 = vtable 的雏形

```c
// led_base.h
typedef int (*led_action_fn)(struct led_base *me);

struct led_ops {
    led_action_fn on;
    led_action_fn off;
    led_action_fn toggle;
};
```

将同类 LED 的可变行为打包为一张函数指针表，调用方按名字 `ops->on` 访问，不再按位置传参。

### 5.2 子类填表

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

100 颗 GPIO LED 共享同一张 12 字节的 ops 表，`const` 使其落入 `.rodata` 段，运行时不可改写。

---

## 六、虚指针：ops 字段进 base

### 6.1 从外部参数变成内置字段

```c
// ch09: ops 作为参数传入
test_led(&red_led.base, &led_ops_gpio);

// ch10: ops 跟着对象自己跑
struct led_base {
    const struct led_ops *ops;   // 虚指针
    const char *name;
    bool is_on;
};

test_led(&red_led.base);  // 只需一个参数！
```

调用方不再需要记住"红灯用 gpio 表、蓝灯用 pwm 表"，彻底消除传错 ops 的风险。

### 6.2 C++ vptr 对照

| C 写法                      | C++ 等价    |
| --------------------------- | ----------- |
| `me->ops`                   | 隐式 vptr   |
| `me->ops->on(me)`           | 虚函数调用  |
| `const struct led_ops *ops` | vtable 指针 |

---

## 七、多态：父类统一接口

### 7.1 胶水函数

```c
// led_base.c
int led_on(struct led_base *me) {
    if (!me || !me->ops || !me->ops->on) return -1;
    return me->ops->on(me);     // 一行 dispatch
}
```

应用层只调 `led_on(base)`，不再写 `me->ops->on(me)` 长串。

### 7.2 运行时多态

```c
// main.c
struct led_base *all_leds[] = {
    &red_led.base,    // GPIO LED
    &blue_led.base,   // PWM LED
    &green_led.base,  // I2C LED
};

for (int i = 0; i < 3; i++)
    led_on(all_leds[i]);   // 同一行代码，三种行为
```

三种 LED 通过向上转型装进同一数组，统一遍历调用——运行时多态的完整形态。

---

## 八、向上转型与板级封装

### 8.1 子类 static 隐藏 + 全局 base 句柄

```c
// led_board_init.c (BSP 层)
static struct led_gpio  s_led_err;       // 子类对象，文件私有
static struct led_pwm   s_led_status;    // 外部不可见

struct led_base *g_led_error;            // 全局基类指针句柄
struct led_base *g_led_status;

void led_board_init(void) {
    led_gpio_init(&s_led_err, "error", 5);
    led_pwm_init(&s_led_status, "status", 0, 50);

    g_led_error  = &s_led_err.base;      // 向上转型
    g_led_status = &s_led_status.base;   // 向上转型
}
```

**板级封装的价值**：

- 应用层只看 `struct led_base *`，不知道背后挂的是哪种 LED
- 换硬件方案（GPIO → PWM）只改本文件，`main.c` 一字不动
- 写 `&s_led_err.base` 而非 `(struct led_base *)&s_led_err`——字段访问让编译器算偏移，安全

---

## 九、container_of：从成员反推外层

### 9.1 问题的提出

子类实现函数只收到 `struct led_base *me`，但要操作的字段在外层子类里。之前用 `(struct led_gpio *)me` 强转——前提是 base 在第 0 偏移，一旦 base 换位置就崩。

### 9.2 宏定义

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
```

三步：(1) 把 `ptr` 转成字节指针，(2) 减去成员偏移（`offsetof` 编译期常量），(3) 转回 `type *`。

### 9.3 使用

```c
static int gpio_on(struct led_base *me) {
    struct led_gpio *self = container_of(me, struct led_gpio, base);
    //                 ↑ 编译期算偏移，运行时只剩一条 sub 指令
    platform_gpio_write(self->pin, true);
    self->base.is_on = true;
    return 0;
}
```

**base 位置不再受限**，强转退役。这是 C 用编译期数学解决 C++ 用 `dynamic_cast` 解决的同一个问题——零运行时开销。

---

## 十、纯虚函数与选填策略

### 10.1 两种 ops 槽位

```c
struct led_ops {
    led_action_fn on;             // 必填
    led_action_fn off;            // 必填
    led_set_brightness_fn set_brightness;  // 选填
};
```

### 10.2 必填：assert 模拟纯虚函数

```c
int led_on(struct led_base *me) {
    assert(me && me->ops && me->ops->on && "subclass must implement on()");
    return me->ops->on(me);
}
```

- Debug 构建：`assert` 触发 abort，暴露"忘填"
- Release 构建（`-DNDEBUG`）：`assert` 消失，零运行时开销
- 等价 C++：`virtual void on() = 0`

### 10.3 选填：NULL 默认行为

```c
int led_set_brightness(struct led_base *me, uint8_t brightness) {
    if (!me->ops->set_brightness) {
        printf("no dimming support\n");
        return 0;    // 安静返回
    }
    return me->ops->set_brightness(me, brightness);
}
```

GPIO LED 不填 `set_brightness`，走默认行为；PWM LED 填了，走自己的实现。等价 C++ 带默认实现的虚函数。

---

## 十一、平台抽象层：跨 MCU 移植的根基

### 11.1 三层架构

| 层级   | 文件                           | 关键操作                                                         |
| ------ | ------------------------------ | ---------------------------------------------------------------- |
| 应用层 | `app.c`                        | `led_on(g_led_error)` — 只调基类接口                             |
| 驱动层 | `led_gpio.c` / `led_pwm.c`     | `container_of` → `platform_gpio_write` — 只调 platform 封装      |
| 平台层 | `platform_pin.c`               | `_g_ops->write(pin, value)` — ops dispatch                       |
| 实现层 | `pin_board.c`（每种 MCU 一份） | STM32: `HAL_GPIO_WritePin` / NXP: `GPIO_PinWrite` / PC: `printf` |

### 11.2 platform 层的 ops + register 机制

```c
// platform_pin.h
struct platform_pin_ops {
    int (*mode)(int32_t pin, int32_t mode);
    int (*write)(int32_t pin, bool value);
    bool (*read)(int32_t pin);
};

int platform_pin_register(const struct platform_pin_ops *ops);  // 启动期注册
void platform_pin_write(int32_t pin, bool value);               // 上层调用
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

### 11.3 跨 MCU 移植：只換实现层

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

**同一份 `led_gpio.c` 在 PC、STM32、NXP 上编译运行，源码零修改**——这就是平台抽象层的威力。

---

## 十二、Linux 内核风格：gpio_chip

### 12.1 ops 内嵌对象 + 多实例 chip

Linux 内核允许同一 SoC 上并存多家厂商 GPIO 控制器（片内 + 外扩 IO），每个 chip 实例自带 ops 表：

```c
// gpio_chip.h
struct gpio_chip {
    const char *label;          // "stm32-gpioa"
    int base;                   // 起始 GPIO 编号
    int ngpio;                  // 该 chip 管多少脚
    int (*request)(struct gpio_chip *gc, unsigned offset);
    void (*free)(struct gpio_chip *gc, unsigned offset);
    int (*direction_output)(struct gpio_chip *gc, unsigned offset, int value);
    int (*get)(struct gpio_chip *gc, unsigned offset);
    void (*set)(struct gpio_chip *gc, unsigned offset, int value);
    void *driver_data;         // 厂商私有上下文
};

struct gpio_desc {
    struct gpio_chip *gc;      // 反查到 chip
    unsigned int offset;       // 脚号偏移
};
```

### 12.2 desc → chip → ops dispatch

```c
// gpiolib.c
void gpiod_set_value(struct gpio_desc *desc, int value) {
    desc->gc->set(desc->gc, desc->offset, value);
    //         ↑ dispatch 到对应厂商的实现
}
```

同一行 `gpiod_set_value` 调用，根据 `desc->gc` 指向哪家厂商的 `gpio_chip`，落到不同实现。这就是 Linux 内核"一份 leds-gpio 驱动跑遍 N 家 SoC"的根基。

---

## 十三、initcall 自动注册：开闭原则的终极实现

### 13.1 问题：每加一个驱动就改 main

```c
// main.c — 违反开闭原则
int main(void) {
    led_init();       // 每加一个驱动
    motor_init();     // 就要在这里加一行
    sensor_init();    // 改完 main 还要加 #include
    uart_init();
    // ...无穷无尽
}
```

### 13.2 解决：编译期 + 链接器 + 启动代码三方配合

```c
// initcall.h
typedef int (*initcall_t)(void);

#define MODULE_INIT(fn) \
    static initcall_t __initcall_##fn \
    __attribute__((used, section("my_initcall"))) = fn

// 声明链接器自动生成的段边界符号
extern initcall_t __start_my_initcall[];
extern initcall_t __stop_my_initcall[];
```

```c
// drv_led.c
static int drv_led_init(void) {
    // ... LED 驱动初始化
    return 0;
}
MODULE_INIT(drv_led_init);  // 一行注册，main 不用改
```

```c
// initcall.c
void do_initcalls(void) {
    initcall_t *fn;
    for (fn = __start_my_initcall; fn < __stop_my_initcall; fn++)
        (*fn)();
}
```

### 13.3 魔法的全部秘密

1. `__attribute__((section("my_initcall")))`：把函数指针放进自定义段
2. 链接器自动合并所有 `.o` 的同名段，生成 `__start_` / `__stop_` 边界符号
3. `__attribute__((used))`：防止编译器优化掉看似"死代码"的 static 变量
4. `do_initcalls()` 遍历段区间，挨个调用

**加新驱动 = 写新文件 + 一行 `MODULE_INIT`**，`main.c` 一字不动——这就是 Linux 内核 `module_init` 那一行魔法的全部秘密。

---

## 十四、OOP-in-C 全景图

| 章节         | C 写法                     | C++ 等价              | 核心变化                             |
| ------------ | -------------------------- | --------------------- | ------------------------------------ |
| 封装         | `struct` + `me` 指针       | `class` + `this`      | 数据与代码分离，多实例共享同一份函数 |
| 手搓 class   | 函数前缀 + init/deinit     | namespace + 构造/析构 | 命名规范防符号冲突，生命周期管理     |
| 数据归位     | static / static const      | private / const       | 三类数据各归其位，信息隐藏完成       |
| 继承         | struct 嵌套                | class 继承            | 基类放第一个字段，构造函数链         |
| 操作表       | ops 结构体                 | vtable 雏形           | 函数指针打包，按名访问               |
| 虚指针       | ops 字段进 base            | vptr                  | 对象自带操作表，调用只需 base 指针   |
| 多态         | 父类统一接口               | virtual dispatch      | 同一行代码，不同行为                 |
| 向上转型     | `&sub.base` 赋值           | implicit upcast       | 子类隐藏，base 句柄暴露，BSP 封装    |
| container_of | 宏反推外层结构体           | `dynamic_cast`        | base 位置不受限，零运行时开销        |
| 纯虚/选填    | assert / NULL 默认         | `= 0` / 默认实现      | 必填调试期断言，选填安静默认         |
| 平台抽象     | HAL ops + register         | 抽象基类 + 工厂       | 跨 MCU 移植只换实现层                |
| Linux 风格   | gpio_chip 内嵌 ops         | 多态 + 多实例         | 同一 SoC 多厂商 chip 并存            |
| initcall     | `__attribute__((section))` | 开闭原则              | 编译期 + 链接器自动注册，main 不动   |

---

## 十五、为什么嵌入式必须用 C 写 OOP

### 15.1 C++ 在嵌入式的现实困境

- **代码膨胀**：虚函数表、RTTI、异常处理增加 10-30% Flash
- **ABI 不稳定**：不同编译器 name mangling 不兼容，二进制模块不能混链
- **确定性缺失**：异常处理栈展开、placement new 失败路径难以预测
- **工具链限制**：许多 MCU 只支持 C89/C99 编译器

### 15.2 C OOP 的工程优势

- **零抽象开销**：struct 嵌套 = 零偏移，container_of = 一条 sub 指令，ops 表 = 间接跳转
- **ABI 稳定**：结构体布局由 C ABI 决定，不同编译器、不同语言都能互操作
- **显式控制**：没有隐式构造/析构，没有隐式 this，每一行代码都可见
- **可审计**：所有"面向对象魔法"都是几行宏 + 结构体，代码审查时一目了然

### 15.3 一句话总结

> C 语言的面向对象不是"没有 C++ 就凑合过"，而是嵌入式场景下的最优解——用最少的语言特性获得最多的设计自由度，每一字节开销都可审计。

---

## 参考资料

- [zhaoming-embedded/oop-in-c](https://github.com/ZhaoChengBo/zhaoming-embedded) — 本文源码仓库
- [Linux Kernel — gpio_chip](https://www.kernel.org/doc/html/latest/driver-api/gpio/)
- [GObject Type System](https://docs.gtk.org/gobject/)
- [Zephyr RTOS Device Driver Model](https://docs.zephyrproject.org/latest/contribute/device-drivers/index.html)
- [C11 Standard — 6.7.2.1 Structure and union specifiers](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)
- [Linux Kernel container_of](https://elixir.bootlin.com/linux/latest/source/include/linux/container_of.h)
