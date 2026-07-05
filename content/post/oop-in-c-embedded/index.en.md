---
title: "The Essence of C: Structs and Pointers — Embedded Object-Oriented Platform-Agnostic Layered Design"
date: 2026-07-03
description: "Starting from struct + me pointer, through data ownership, inheritance nesting, operation tables, virtual pointers, polymorphic dispatch, container_of, platform abstraction layer, to Linux kernel style — using a single LED to explain the full essence of embedded C object-oriented programming"
image: C语言结构体指针博客封面.png
categories:
  - "Embedded"
tags:
  - "C Language"
  - "Object-Oriented"
  - "OOP-in-C"
  - "Embedded"
  - "Platform Abstraction"
  - "Linux Kernel"
math: true
---

## Preface

C has no `class`, no `virtual`, no `interface`, yet large-scale projects like the Linux kernel, GObject, and Zephyr RTOS all build complete object-oriented systems in C. Their weapons are just two: **structs** and **pointers**.

This article follows the progressive refactoring of an embedded LED driver as its main thread, starting from the most primitive `struct + me` and going all the way to Linux kernel-style `gpio_chip` + initcall auto-registration, using a single LED to explain the full essence of embedded C object-oriented programming.

---

## 0. Why Structs and Pointers Are the Essence of C

### 0.1 C Has Only Three Cornerstones

C provides very few composite type tools: basic types, arrays, structs, unions, enums, and pointers. Among these, only two can truly carry design complexity: **structs** and **pointers**.

- **Structs** are C's only data aggregation tool — binding multiple attributes together, forming the embryo of an "object"
- **Pointers** are C's only indirection tool — operating on arbitrary objects through addresses, the foundation for implementing "polymorphism"

Without structs, data can only be scattered as global variables; without pointers, functions can only operate on fixed objects, making multi-instance, polymorphism, and callbacks impossible. C++'s `class`, `virtual`, and `interface` are essentially syntactic sugar over these two.

### 0.2 Peripheral Address Mapping: The Natural Stage for Struct Pointers

Open the standard peripheral library or HAL library for any ARM Cortex-M chip, and you'll see code like this:

**STM32 HAL (`stm32f407xx.h`):**

```c
typedef struct {
    __IO uint32_t SR;         // Status register, offset 0x00
    __IO uint32_t DR;         // Data register, offset 0x04
    __IO uint32_t BRR;        // Baud rate register, offset 0x08
    __IO uint32_t CR1;        // Control register 1, offset 0x0C
    __IO uint32_t CR2;        // Control register 2, offset 0x10
    __IO uint32_t CR3;        // Control register 3, offset 0x14
    // ... more registers
} USART_TypeDef;

#define USART1  ((USART_TypeDef *)0x40011000)  // Forced cast!
#define USART2  ((USART_TypeDef *)0x40004400)
#define USART6  ((USART_TypeDef *)0x40011400)
```

**What's happening here?** Chip designers arrange peripheral registers at fixed offsets in a contiguous address space at the hardware level. C struct members are also arranged in declaration order by default, with the compiler calculating offsets based on member size and alignment rules — when the struct member declaration order and types match the hardware register layout, the struct's memory layout **perfectly aligns** with the hardware register layout.

Then, by casting the peripheral base address to a pointer to that struct, you can directly read and write hardware registers using `USART1->CR1`.

> [!NOTE]
> Modern C/C++ editors or IDEs (VS Code + clangd, STM32CubeIDE, Keil MDK, etc.) provide **intelligent autocompletion** for struct members — typing `USART1->` automatically lists all members like `SR`, `DR`, `BRR`, `CR1`, while magic offset `0x0C` provides no hints at all. This means struct pointer mapping is not only compile-time safe, but also **editing-time safe** — typos are eliminated at the input stage, not exposed at runtime.

### 0.3 Why This Is Not a "Hack" but a Natural Design

| Advantage                          | Explanation                                                                                                                                                                              |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Zero overhead**                  | `USART1->CR1` compiles to a single `LDR`/`STR` instruction plus a fixed offset, completely equivalent to `*(volatile uint32_t *)0x4001100C` — no extra overhead at all                   |
| **Type safety**                    | `USART1` is of type `USART_TypeDef *`, so you can't accidentally write `GPIOA->CR1` — the compiler will error because `GPIO_TypeDef` has no `CR1` member                                 |
| **Readability**                    | `usart_baudrate_set(USART0, 115200)` is far more human-readable than `*(volatile uint32_t *)(0x40011000 + 0x08) = 0x1A0B`                                                                |
| **Compile-time checking**          | Wrong struct member name → compile error; wrong magic offset → runtime crash                                                                                                             |
| **Natural multi-instance support** | Same `USART_TypeDef` struct, different base addresses = different peripheral instances. `usart_init(USART0)` and `usart_init(USART1)` share the same code — isn't this the `me` pointer? |

### 0.4 Unified Logic: From Hardware Mapping to Software OOP

Chip vendors use struct pointers to map hardware registers, and we use struct pointers to implement object-oriented programming — **the underlying logic is identical**:

| Hardware Mapping                         | Software OOP                      | Unified Logic                                                          |
| ---------------------------------------- | --------------------------------- | ---------------------------------------------------------------------- |
| `USART_TypeDef` struct                   | `struct led` struct               | Structs aggregate data, forming "objects"                              |
| `USART1` base address cast               | `&red_led` address-of             | Pointer points to a specific instance                                  |
| `USART1->CR1 = val`                      | `me->pin = 5`                     | Access instance members through pointer                                |
| `usart_init(USART0)`                     | `led_init(&red_led)`              | Function operates on different instances via pointer                   |
| `USART_TypeDef` reused across all USARTs | `led_on()` reused across all LEDs | Same code operates on different instances — the embryo of polymorphism |

> [!IMPORTANT]
> Struct + pointer is not a "patch" for C, but the **native way** C interacts with hardware. Chip vendors use it to map registers, kernel developers use it to implement polymorphism, we use it to build platform abstractions — all from the same lineage.

---

## 1. The Starting Point of Encapsulation: struct + me Pointer

### 1.1 The Most Primitive Form

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

The first parameter of every function is `struct led *me` — this is the `this` pointer that C++ hides.

### 1.2 Multiple Instances Sharing the Same Code

```c
// main.c
struct led red_led, green_led, blue_led;

led_init(&red_led, 5);
led_init(&green_led, 3);
led_init(&blue_led, 7);

led_on(&red_led);    // Turn on the red LED
led_on(&green_led);  // Turn on the green LED
```

Three LEDs share the same `led_on` code; the function knows which LED to operate on through the `me` pointer. Want to add a 100th LED? Just create another `struct led` instance — not a single line of function code needs to change.

### 1.3 C++ Equivalent Comparison

| C Approach         | C++ Equivalent  |
| ------------------ | --------------- |
| `struct led`       | `class Led`     |
| `led_on(&red_led)` | `red_led.on()`  |
| `struct led *me`   | implicit `this` |
| `me->pin`          | `this->pin`     |

---

## 2. Hand-Rolled class: Function Prefix + init/deinit

### 2.1 Naming Convention = namespace

```c
// led.h — LED class
int  led_init(struct led *me, uint8_t pin);
void led_deinit(struct led *me);
void led_on(struct led *me);

// motor.h — Motor class
int  motor_init(struct motor *me, uint8_t pin);
void motor_deinit(struct motor *me);
void motor_start(struct motor *me);
```

The `led_xxx` prefix guarantees unique linker symbols. Without prefixes, if `led.c` defines `void init()` and `motor.c` also defines `void init()`, you immediately get a multiple-definition conflict. C++'s namespace + name mangling does the same thing — the compiler manipulates strings for you; in C, you write them by hand.

### 2.2 The Three Responsibilities of a Constructor

```c
int led_init(struct led *me, uint8_t pin) {
    if (!me || !pin_valid(pin)) return -1;  // 1. Parameter validation
    me->pin = pin;
    me->is_on = false;
    platform_gpio_init(pin, GPIO_OUTPUT);    // 2. Hardware initialization
    platform_gpio_write(pin, false);         // 3. Default state
    me->initialized = true;
    return 0;
}
```

### 2.3 const me = Read-Only Semantics

```c
int led_get_state(const struct led *me, bool *is_on);
```

`const` on `me` guarantees that query functions won't modify the object's state — the compiler catches accidental writes at compile time.

---

## 3. Three Levels of Data Ownership: The Complete Form of Information Hiding

### 3.1 The Ownership of Three Types of Data

| Category            | Storage Location | Lifetime                   | Visibility                        |
| ------------------- | ---------------- | -------------------------- | --------------------------------- |
| Instance data       | `struct` fields  | Follows `me`               | Exposed in `.h` or hidden in `.c` |
| Module-shared data  | `static` vars    | Entire program             | Only within this `.c` file        |
| Read-only constants | `static const`   | Determined at compile time | Only within this `.c` file        |

```c
// led.c
static const int MAX_BRIGHTNESS = 100;        // Category 3: read-only constant
static struct led led_pool[8];                 // Category 2: module-shared (object pool)
static int s_init_count = 0;                   // Category 2: module-shared (counter)

struct led *led_acquire(uint8_t pin) {         // Factory: allocate from pool
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

**Static object pool factory**: Zero heap fragmentation on MCU, O(1) allocation — far safer than `malloc`.

### 3.2 Anti-Pattern: The Disaster of Global Variables

```c
// led_bad.c — Never write like this!
int g_pin;           // Instance data as global — most fatal
int g_brightness;    // Anyone can modify, no owner
int init_count;      // No static, external extern can modify
```

`bad_led_init(5)` sets `g_pin=5`, then `bad_led_init(3)` sets `g_pin=3` — the red LED's pin is overwritten, the code doesn't error out, the logic is completely wrong, and you can debug for a week without finding it.

**When data has no owner, bugs become the owner.**

---

## 4. Inheritance: Struct Nesting

### 4.1 Extracting the Common Factor

Eight types of LED all have `name + is_on`. Instead of copying eight times, extract them into a base and write once:

```c
// led_base.h
struct led_base {
    const char *name;
    bool is_on;
};

// led_gpio.h
struct led_gpio {
    struct led_base base;   // Base class as the first field!
    uint8_t pin;            // GPIO-specific
};

// led_pwm.h
struct led_pwm {
    struct led_base base;   // Base class as the first field!
    uint8_t channel;        // PWM-specific
    uint8_t duty;
};
```

**Base class as the first field** is a hard rule — C11 standard section 6.7.2.1 guarantees that the first member of a struct has offset 0, so the address of `&gpio->base` equals the address of `gpio` itself. The Linux kernel, Zephyr, and GObject all follow this convention.

### 4.2 Constructor Chaining

```c
int led_gpio_init(struct led_gpio *me, const char *name, uint8_t pin) {
    led_base_init(&me->base, name);   // Call base constructor first
    me->pin = pin;                     // Then handle derived fields
    return 0;
}
```

When C++ executes `led_gpio g;`, the compiler automatically calls the base constructor then the derived constructor — that's exactly what we write by hand here.

---

## 5. Operation Table: Packing Function Pointers

### 5.1 ops Struct = The Embryonic Form of vtable

```c
// led_base.h
typedef int (*led_action_fn)(struct led_base *me);

struct led_ops {
    led_action_fn on;
    led_action_fn off;
    led_action_fn toggle;
};
```

Pack the variable behaviors of the same LED type into a function pointer table. Callers access by name `ops->on` instead of by positional parameters.

### 5.2 Subclass Fills the Table

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

100 GPIO LEDs share the same 12-byte ops table. `const` places it in the `.rodata` section, making it non-writable at runtime.

---

## 6. Virtual Pointer: ops Field Inside base

### 6.1 From External Parameter to Built-In Field

```c
// ch09: ops passed as a parameter
test_led(&red_led.base, &led_ops_gpio);

// ch10: ops travels with the object itself
struct led_base {
    const struct led_ops *ops;   // Virtual pointer
    const char *name;
    bool is_on;
};

test_led(&red_led.base);  // Only one parameter needed!
```

Callers no longer need to remember "red LED uses gpio table, blue LED uses pwm table" — the risk of passing the wrong ops is completely eliminated.

### 6.2 C++ vptr Comparison

| C Approach                  | C++ Equivalent        |
| --------------------------- | --------------------- |
| `me->ops`                   | implicit vptr         |
| `me->ops->on(me)`           | virtual function call |
| `const struct led_ops *ops` | vtable pointer        |

---

## 7. Polymorphism: Unified Base Class Interface

### 7.1 Glue Functions

```c
// led_base.c
int led_on(struct led_base *me) {
    if (!me || !me->ops || !me->ops->on) return -1;
    return me->ops->on(me);     // One-line dispatch
}
```

The application layer only calls `led_on(base)`, never writing the long `me->ops->on(me)` chain.

### 7.2 Runtime Polymorphism

```c
// main.c
struct led_base *all_leds[] = {
    &red_led.base,    // GPIO LED
    &blue_led.base,   // PWM LED
    &green_led.base,  // I2C LED
};

for (int i = 0; i < 3; i++)
    led_on(all_leds[i]);   // Same line of code, three behaviors
```

Three types of LED are placed into the same array through upcasting, and uniformly iterated and called — the complete form of runtime polymorphism.

---

## 8. Upcasting and Board-Level Encapsulation

### 8.1 Subclass static Hiding + Global base Handle

```c
// led_board_init.c (BSP layer)
static struct led_gpio  s_led_err;       // Subclass object, file-private
static struct led_pwm   s_led_status;    // Not visible externally

struct led_base *g_led_error;            // Global base class pointer handle
struct led_base *g_led_status;

void led_board_init(void) {
    led_gpio_init(&s_led_err, "error", 5);
    led_pwm_init(&s_led_status, "status", 0, 50);

    g_led_error  = &s_led_err.base;      // Upcast
    g_led_status = &s_led_status.base;   // Upcast
}
```

**Value of board-level encapsulation**:

- The application layer only sees `struct led_base *`, unaware of which LED type is behind it
- Changing the hardware scheme (GPIO to PWM) only requires modifying this file — `main.c` stays untouched
- Writing `&s_led_err.base` instead of `(struct led_base *)&s_led_err` — field access lets the compiler compute the offset, which is safer

---

## 9. container_of: Recovering the Outer Struct from a Member

### 9.1 The Problem

Subclass implementation functions only receive `struct led_base *me`, but the fields to operate on are in the outer subclass. Previously we used `(struct led_gpio *)me` cast — this requires base to be at offset 0; if base moves, it crashes.

### 9.2 The Macro

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
```

Three steps: (1) cast `ptr` to a byte pointer, (2) subtract the member offset (`offsetof` is a compile-time constant), (3) cast back to `type *`.

### 9.3 Usage

```c
static int gpio_on(struct led_base *me) {
    struct led_gpio *self = container_of(me, struct led_gpio, base);
    //                 ^ compile-time offset calculation, runtime only a single sub instruction
    platform_gpio_write(self->pin, true);
    self->base.is_on = true;
    return 0;
}
```

**The position of base is no longer constrained**, and the cast is retired. This is C using compile-time math to solve the same problem that C++ solves with `dynamic_cast` — with zero runtime overhead.

---

## 10. Pure Virtual Functions and Optional Strategy

### 10.1 Two Types of ops Slots

```c
struct led_ops {
    led_action_fn on;             // Required
    led_action_fn off;            // Required
    led_set_brightness_fn set_brightness;  // Optional
};
```

### 10.2 Required: assert Simulates Pure Virtual Functions

```c
int led_on(struct led_base *me) {
    assert(me && me->ops && me->ops->on && "subclass must implement on()");
    return me->ops->on(me);
}
```

- Debug build: `assert` triggers abort, exposing "forgot to fill in"
- Release build (`-DNDEBUG`): `assert` disappears, zero runtime overhead
- C++ equivalent: `virtual void on() = 0`

### 10.3 Optional: NULL Default Behavior

```c
int led_set_brightness(struct led_base *me, uint8_t brightness) {
    if (!me->ops->set_brightness) {
        printf("no dimming support\n");
        return 0;    // Quiet return
    }
    return me->ops->set_brightness(me, brightness);
}
```

GPIO LED doesn't fill `set_brightness` and gets the default behavior; PWM LED fills it and gets its own implementation. Equivalent to a C++ virtual function with a default implementation.

---

## 11. Platform Abstraction Layer: The Foundation for Cross-MCU Porting

### 11.1 Three-Layer Architecture

| Layer          | File                        | Key Operations                                                   |
| -------------- | --------------------------- | ---------------------------------------------------------------- |
| Application    | `app.c`                     | `led_on(g_led_error)` — only calls base class interface          |
| Driver         | `led_gpio.c` / `led_pwm.c`  | `container_of` → `platform_gpio_write` — only calls platform API |
| Platform       | `platform_pin.c`            | `_g_ops->write(pin, value)` — ops dispatch                       |
| Implementation | `pin_board.c` (one per MCU) | STM32: `HAL_GPIO_WritePin` / NXP: `GPIO_PinWrite` / PC: `printf` |

### 11.2 Platform Layer's ops + register Mechanism

```c
// platform_pin.h
struct platform_pin_ops {
    int (*mode)(int32_t pin, int32_t mode);
    int (*write)(int32_t pin, bool value);
    bool (*read)(int32_t pin);
};

int platform_pin_register(const struct platform_pin_ops *ops);  // Registration at startup
void platform_pin_write(int32_t pin, bool value);               // Upper layer call
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

### 11.3 Cross-MCU Porting: Only Swap the Implementation Layer

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

**The same `led_gpio.c` compiles and runs on PC, STM32, and NXP with zero source modifications** — this is the power of the platform abstraction layer.

---

## 12. Linux Kernel Style: gpio_chip

### 12.1 ops Embedded in Object + Multi-Instance chip

The Linux kernel allows multiple vendors' GPIO controllers (on-chip + external I/O expanders) to coexist on the same SoC, with each chip instance carrying its own ops table:

```c
// gpio_chip.h
struct gpio_chip {
    const char *label;          // "stm32-gpioa"
    int base;                   // Starting GPIO number
    int ngpio;                  // How many pins this chip manages
    int (*request)(struct gpio_chip *gc, unsigned offset);
    void (*free)(struct gpio_chip *gc, unsigned offset);
    int (*direction_output)(struct gpio_chip *gc, unsigned offset, int value);
    int (*get)(struct gpio_chip *gc, unsigned offset);
    void (*set)(struct gpio_chip *gc, unsigned offset, int value);
    void *driver_data;         // Vendor-private context
};

struct gpio_desc {
    struct gpio_chip *gc;      // Reverse-lookup to chip
    unsigned int offset;       // Pin number offset
};
```

### 12.2 desc → chip → ops dispatch

```c
// gpiolib.c
void gpiod_set_value(struct gpio_desc *desc, int value) {
    desc->gc->set(desc->gc, desc->offset, value);
    //         ↑ dispatches to the corresponding vendor's implementation
}
```

The same `gpiod_set_value` call, depending on which vendor's `gpio_chip` `desc->gc` points to, lands in a different implementation. This is the foundation of the Linux kernel's "one leds-gpio driver runs across N SoC vendors."

---

## 13. initcall Auto-Registration: The Ultimate Implementation of the Open-Closed Principle

### 13.1 The Problem: Every New Driver Requires Changing main

```c
// main.c — Violates the Open-Closed Principle
int main(void) {
    led_init();       // Every new driver
    motor_init();     // requires adding a line here
    sensor_init();    // and adding an #include
    uart_init();
    // ...endless
}
```

### 13.2 The Solution: Compiler + Linker + Startup Code Cooperation

```c
// initcall.h
typedef int (*initcall_t)(void);

#define MODULE_INIT(fn) \
    static initcall_t __initcall_##fn \
    __attribute__((used, section("my_initcall"))) = fn

// Declare linker-generated section boundary symbols
extern initcall_t __start_my_initcall[];
extern initcall_t __stop_my_initcall[];
```

```c
// drv_led.c
static int drv_led_init(void) {
    // ... LED driver initialization
    return 0;
}
MODULE_INIT(drv_led_init);  // One-line registration, main unchanged
```

```c
// initcall.c
void do_initcalls(void) {
    initcall_t *fn;
    for (fn = __start_my_initcall; fn < __stop_my_initcall; fn++)
        (*fn)();
}
```

### 13.3 The Full Secret of the Magic

1. `__attribute__((section("my_initcall")))`: Places the function pointer into a custom section
2. The linker automatically merges all `.o` files' same-named sections, generating `__start_` / `__stop_` boundary symbols
3. `__attribute__((used))`: Prevents the compiler from optimizing away the seemingly "dead code" static variable
4. `do_initcalls()` iterates over the section range, calling each one

**Adding a new driver = write a new file + one line of `MODULE_INIT`**, and `main.c` stays untouched — this is the full secret behind that one line of magic in the Linux kernel's `module_init`.

---

## 14. OOP-in-C Panorama

| Section                 | C Approach                    | C++ Equivalent                     | Core Change                                                                 |
| ----------------------- | ----------------------------- | ---------------------------------- | --------------------------------------------------------------------------- |
| Encapsulation           | `struct` + `me` pointer       | `class` + `this`                   | Data and code separated, multiple instances share the same functions        |
| Hand-rolled class       | Function prefix + init/deinit | namespace + constructor/destructor | Naming convention prevents symbol conflicts, lifecycle management           |
| Data ownership          | static / static const         | private / const                    | Three types of data each in their proper place, information hiding complete |
| Inheritance             | struct nesting                | class inheritance                  | Base as first field, constructor chaining                                   |
| Operation table         | ops struct                    | vtable embryo                      | Function pointers packed, access by name                                    |
| Virtual pointer         | ops field inside base         | vptr                               | Object carries its own operation table, calls only need base pointer        |
| Polymorphism            | Unified base class interface  | virtual dispatch                   | Same line of code, different behavior                                       |
| Upcasting               | `&sub.base` assignment        | implicit upcast                    | Subclass hidden, base handle exposed, BSP encapsulation                     |
| container_of            | Macro to recover outer struct | `dynamic_cast`                     | Base position unconstrained, zero runtime overhead                          |
| Pure virtual / Optional | assert / NULL default         | `= 0` / default impl               | Required: debug-time assertion; Optional: quiet default                     |
| Platform abstraction    | HAL ops + register            | Abstract base class + factory      | Cross-MCU porting only swaps implementation layer                           |
| Linux style             | gpio_chip with embedded ops   | Polymorphism + multi-instance      | Multiple vendor chips coexist on the same SoC                               |
| initcall                | `__attribute__((section))`    | Open-Closed Principle              | Compile-time + linker auto-registration, main unchanged                     |

---

## 15. Why Embedded Must Use C for OOP

### 15.1 C++'s Real-World Difficulties in Embedded

- **Code bloat**: Virtual function tables, RTTI, and exception handling increase Flash by 10-30%
- **Unstable ABI**: Different compilers' name mangling are incompatible; binary modules cannot be cross-linked
- **Lack of determinism**: Exception handling stack unwinding, placement new failure paths are hard to predict
- **Toolchain limitations**: Many MCUs only support C89/C99 compilers

### 15.2 Engineering Advantages of C OOP

- **Zero abstraction overhead**: struct nesting = zero offset, container_of = one sub instruction, ops table = indirect jump
- **Stable ABI**: Struct layout is determined by the C ABI; different compilers and different languages can interoperate
- **Explicit control**: No implicit construction/destruction, no implicit this — every line of code is visible
- **Auditable**: All "object-oriented magic" is just a few lines of macros + structs — crystal clear during code review

### 15.3 One-Sentence Summary

> Object-oriented programming in C is not "making do without C++" — it is the optimal solution for embedded scenarios: achieving the most design freedom with the fewest language features, where every byte of overhead is auditable.

---

## References

- [zhaoming-embedded/oop-in-c](https://github.com/ZhaoChengBo/zhaoming-embedded) — Source repository for this post
- [Linux Kernel — gpio_chip](https://www.kernel.org/doc/html/latest/driver-api/gpio/)
- [GObject Type System](https://docs.gtk.org/gobject/)
- [Zephyr RTOS Device Driver Model](https://docs.zephyrproject.org/latest/contribute/device-drivers/index.html)
- [C11 Standard — 6.7.2.1 Structure and union specifiers](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)
- [Linux Kernel container_of](https://elixir.bootlin.com/linux/latest/source/include/linux/container_of.h)
