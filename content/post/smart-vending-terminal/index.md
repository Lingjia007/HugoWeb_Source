---
title: "LVGL 动态加载与渲染树管理：智能售货机终端的应用树架构实践"
date: 2026-07-02
description: "基于 ArtInChip + LVGL v8 + RT-Thread 的智能售货机终端，深入剖析应用树（layout_node）如何管理 LVGL 渲染树，涵盖组合模式双树映射、Wrap 桥接机制、策略驱动的 Flex/Grid 布局、控件工厂动态创建、事件驱动热更新、静态骨架到动态组件的替换流程等核心技术详解"
categories:
  - "嵌入式"
tags:
  - "LVGL"
  - "RT-Thread"
  - "ArtInChip"
  - "售货机"
  - "嵌入式UI"
  - "设计模式"
---

## 前言

LVGL 作为嵌入式领域最活跃的图形库，其核心是一个树形的**渲染树**（`lv_obj_t` 对象树），所有可见控件都是这棵树的节点。但在实际业务中，直接操作渲染树面临诸多问题：**静态布局无法动态更新**、**控件生命周期难以统一管理**、**组件间耦合紧密**。

本文以一个基于 **ArtInChip + LVGL v8 + RT-Thread** 的智能售货机终端为实践载体，深入剖析如何在 LVGL 渲染树之上构建一层**应用树**（`layout_node` 框架），通过组合模式、工厂模式、策略模式和发布-订阅模式的协同，实现 UI 的动态加载、热更新和生命周期管理。

---

## 一、双树架构：渲染树 vs 应用树

### 1.1 问题背景

项目由 AiUIBuilder v2.0.2 生成静态 UI 骨架（`ui_builder/screen.c`），在 800×480 屏幕上创建了横幅图片、商品卡片、购物车行、导航按钮等控件。但这些控件是**硬编码的静态对象**：

```c
// screen.c (AiUIBuilder 生成)
scr->image_banner_1 = lv_img_create(scr->container_banner);
scr->container_main_1 = lv_obj_create(scr->container_main);
// ...固定数量、固定数据、无法运行时增删
```

业务需求要求**运行时动态增删控件**（商品数量可变、购物车项动态添加、横幅可配置替换），直接操作渲染树会导致：

- 对象生命周期分散在各回调中，容易泄漏
- 无法批量销毁子树
- 布局参数硬编码，无法统一调整
- 组件间直接调用，紧耦合

### 1.2 解决方案：应用树

在 LVGL 渲染树之上构建一棵**应用树**，每个应用树节点持有对应的 `lv_obj_t*`，并附加生命周期和布局管理能力：

```
应用树 (layout_node)
  container (strategy=Flex-Row)
  ├── widget_unit (banner_item, priv=...)
  ├── widget_unit (banner_item, priv=...)
  └── widget_unit (banner_item, priv=...)

渲染树 (lv_obj_t)
  lv_obj (LV_FLEX_FLOW_ROW)
  ├── lv_img (banner_1.png)
  ├── lv_img (banner_2.png)
  └── lv_img (banner_3.png)
```

应用树节点通过 `obj` 指针与渲染树节点一一对应，但应用树额外管理了：

| 管理维度   | 渲染树 (lv_obj_t)                  | 应用树 (layout_node)                 |
| ---------- | ---------------------------------- | ------------------------------------ |
| 生命周期   | `lv_obj_del()` 手动散落各处        | `destroy()` 虚函数，深度优先自动销毁 |
| 子节点管理 | LVGL 内部链表，无业务语义          | `children[]` 动态数组，容量倍增      |
| 布局参数   | 分散在各 `lv_obj_set_style_*` 调用 | `layout_strategy_t` 统一封装         |
| 数据绑定   | 手动逐字段赋值                     | `bind()` / `update()` 虚函数         |
| 私有数据   | 无标准机制                         | `widget_unit.priv` 指向内存池对象    |

---

## 二、组合模式：layout_node 核心设计

### 2.1 三层类型体系

应用树的核心是一个**组合模式 (Composite)** 的三层类型体系：

```c
/* 抽象基类 - 虚函数表 */
struct layout_node {
    lv_obj_t *obj;                          // 对应的渲染树节点
    const char *type_name;                  // 类型名（"container"/"banner_item"/...）
    layout_node_type_t node_type;           // CONTAINER 或 WIDGET
    layout_container_t *parent;             // 父容器引用

    void (*destroy)(layout_node_t *self);   // 虚函数：销毁
    void (*update)(layout_node_t *self,     // 虚函数：数据更新
                   const cJSON *data);
};

/* 容器节点 - 可含子节点 */
struct layout_container {
    layout_node_t base;                     // 继承基类
    layout_strategy_t strategy;             // 布局策略
    layout_node_t **children;               // 子节点动态数组
    int child_count;
    int child_capacity;
    // 方法指针: add_child, remove_child, clear, get_child
};

/* 叶子节点 - 最小可渲染单元 */
struct widget_unit {
    layout_node_t base;                     // 继承基类
    void *priv;                             // 私有数据（从内存池分配）
    void (*bind)(widget_unit_t *self,       // 数据绑定函数
                 const cJSON *data);
};
```

### 2.2 深度优先销毁

`destroy()` 虚函数实现了**深度优先**的子树销毁，保证所有子节点先于父节点被回收：

```c
static void container_destroy(layout_node_t *self_base) {
    layout_container_t *self = (layout_container_t *)self_base;

    // 1. 先销毁所有子节点（深度优先）
    for (int i = 0; i < self->child_count; i++)
        layout_node_destroy(self->children[i]);

    // 2. 释放子节点数组
    rt_free(self->children);

    // 3. 销毁渲染树对象
    lv_obj_del(self->base.obj);

    // 4. 释放自身
    rt_free(self);
}
```

通用销毁调度器通过虚函数表实现多态：

```c
void layout_node_destroy(layout_node_t *node) {
    if (node && node->destroy)
        node->destroy(node);
}
```

这意味着调用者无需关心节点的具体类型，一个 `layout_node_destroy()` 即可销毁整棵子树。

### 2.3 子节点动态数组

容器使用动态数组管理子节点，初始容量 8，倍增扩容：

```c
#define LAYOUT_CHILDREN_INIT_CAP 8

static int children_ensure_capacity(layout_container_t *self, int needed) {
    if (needed <= self->child_capacity) return 0;

    int new_cap = self->child_capacity;
    while (new_cap < needed)
        new_cap *= 2;

    layout_node_t **new_arr = rt_malloc(sizeof(layout_node_t *) * new_cap);
    memcpy(new_arr, self->children, sizeof(layout_node_t *) * self->child_count);
    rt_free(self->children);
    self->children = new_arr;
    self->child_capacity = new_cap;
}
```

`add_child` 时自动扩容并应用布局策略中的子节点尺寸：

```c
void layout_container_add_child(layout_container_t *self, layout_node_t *child) {
    children_ensure_capacity(self, self->child_count + 1);
    child->parent = self;
    self->children[self->child_count++] = child;
    layout_strategy_apply_child_size(child->obj, &self->strategy);
}
```

`clear()` 清空所有子节点后，若容量超过初始值的 4 倍则缩容，避免长期运行后内存膨胀：

```c
void layout_container_clear(layout_container_t *self) {
    for (int i = 0; i < self->child_count; i++)
        layout_node_destroy(self->children[i]);
    self->child_count = 0;

    if (self->child_capacity > LAYOUT_CHILDREN_INIT_CAP * 4) {
        rt_free(self->children);
        self->children = rt_malloc(sizeof(layout_node_t *) * LAYOUT_CHILDREN_INIT_CAP);
        self->child_capacity = LAYOUT_CHILDREN_INIT_CAP;
    }
}
```

---

## 三、Wrap 桥接：从静态骨架到动态容器

### 3.1 两种创建方式

`layout_container` 提供两种创建方式，这是整个架构最关键的设计：

| 方式       | 函数                        | 创建 LVGL 对象         | 销毁 LVGL 对象                             | 用途                      |
| ---------- | --------------------------- | ---------------------- | ------------------------------------------ | ------------------------- |
| **Create** | `layout_container_create()` | 新建 `lv_obj_create()` | `container_destroy()` 中 `lv_obj_del()`    | 全新动态容器              |
| **Wrap**   | `layout_container_wrap()`   | 复用已有对象           | `container_destroy_wrapped()` 中**不**删除 | 桥接 AiUIBuilder 静态对象 |

### 3.2 Wrap 机制详解

`layout_container_wrap()` 接收一个**已存在的 `lv_obj_t*`**，将其包装为应用树容器，但**不改变对象所有权**：

```c
layout_container_t *layout_container_wrap(lv_obj_t *existing_obj,
                                          const layout_strategy_t *strategy) {
    layout_container_t *self = rt_malloc(sizeof(layout_container_t));
    memset(self, 0, sizeof(*self));

    // 复用已有对象，而非创建新对象
    self->base.obj = existing_obj;

    // 在已有对象上应用布局策略
    layout_strategy_apply(existing_obj, &self->strategy);

    // 使用特殊的销毁函数 - 不删除 LVGL 对象
    self->base.destroy = container_destroy_wrapped;

    return self;
}
```

`container_destroy_wrapped()` 的关键区别：

```c
static void container_destroy_wrapped(layout_node_t *self_base) {
    layout_container_t *self = (layout_container_t *)self_base;

    // 仍然销毁所有子节点（深度优先）
    for (int i = 0; i < self->child_count; i++)
        layout_node_destroy(self->children[i]);
    rt_free(self->children);

    // 关键：不删除 LVGL 对象，它由外部（screen_t）管理
    self->base.obj = NULL;

    rt_free(self);
}
```

### 3.3 实际桥接流程

在 `custom_init()` 中，桥接流程如下：

```
1. AiUIBuilder 生成:  scr->container_banner (lv_obj_t)
                       ├── scr->image_banner_1 (静态子控件)
                       ├── scr->image_banner_2
                       └── scr->image_banner_3

2. 删除静态子控件:    lv_obj_del(scr->image_banner_1);
                       lv_obj_del(scr->image_banner_2);
                       lv_obj_del(scr->image_banner_3);

3. Wrap 容器:         banner_comp->container = layout_container_wrap(
                           scr->container_banner, &flex_row_strategy);

4. 动态填充子节点:    for each image_path:
                         item = widget_factory_create("banner_item", container->base.obj);
                         container->add_child(container, &item->base);
                         item->bind(item, &path_data);
```

这样，AiUIBuilder 创建的 `container_banner` 仍然存在于渲染树中（由 `screen_t` 管理生命周期），但它的**子控件已全部替换为应用树管理的动态节点**，支持运行时增删、数据绑定和热更新。

---

## 四、策略模式：layout_strategy 布局引擎

### 4.1 策略配置结构

`layout_strategy_t` 将 LVGL 的分散布局 API 统一封装为一个配置结构：

```c
typedef struct {
    layout_type_t type;       // FLEX_ROW / FLEX_COLUMN / GRID
    int gap_x, gap_y;        // 间距
    int pad_x, pad_y;        // 内边距
    int cols;                 // Grid 列数
    int cell_w, cell_h;      // 子节点固定尺寸 (0=auto)
    bool scrollable;          // 是否可滚动
    lv_dir_t scroll_dir;      // 滚动方向
} layout_strategy_t;
```

### 4.2 策略应用

`layout_strategy_apply()` 将策略一次性映射到 LVGL Flex API：

```c
void layout_strategy_apply(lv_obj_t *obj, const layout_strategy_t *s) {
    switch (s->type) {
    case LAYOUT_FLEX_ROW:
        lv_obj_set_flex_flow(obj, LV_FLEX_FLOW_ROW);
        break;
    case LAYOUT_FLEX_COLUMN:
        lv_obj_set_flex_flow(obj, LV_FLEX_FLOW_COLUMN);
        break;
    case LAYOUT_GRID:
        // Grid = row-wrap，子节点通过 FLEX_IN_NEW_TRACK 换行
        lv_obj_set_flex_flow(obj, LV_FLEX_FLOW_ROW_WRAP);
        break;
    }

    lv_obj_set_flex_align(obj, LV_FLEX_ALIGN_START,
                              LV_FLEX_ALIGN_START,
                              LV_FLEX_ALIGN_START);

    lv_obj_set_style_pad_row(obj, s->gap_y, ...);
    lv_obj_set_style_pad_column(obj, s->gap_x, ...);

    if (s->scrollable) {
        lv_obj_add_flag(obj, LV_OBJ_FLAG_SCROLLABLE);
        lv_obj_set_scroll_dir(obj, s->scroll_dir);
    }
}
```

### 4.3 四大组件的布局策略

在 `custom_init()` 中，四大组件各自使用不同的策略创建：

```c
// 横幅轮播 - Flex-Row，水平滚动
s_banner_comp = banner_component_create(
    scr->container_banner,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_ROW, .cell_w = 494, .cell_h = 78,
        .scrollable = true, .scroll_dir = LV_DIR_HOR,
    }, ...);

// 商品展示 - Grid 3列，垂直滚动
s_commodity_comp = commodity_component_create(
    scr->container_main,
    &(layout_strategy_t){
        .type = LAYOUT_GRID, .cols = 3,
        .gap_x = 12, .gap_y = 12, .cell_w = 240, .cell_h = 100,
        .scrollable = true, .scroll_dir = LV_DIR_VER,
    }, ...);

// 购物车 - Flex-Column，垂直滚动
s_cart_comp = cart_component_create(
    scr->container_cart_list_scroll,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_COLUMN, .gap_y = 10,
        .cell_w = 750, .cell_h = 70,
        .scrollable = true, .scroll_dir = LV_DIR_VER,
    });

// 导航栏 - Flex-Row，不可滚动
s_navi_comp = navi_component_create(
    scr->container_navi_bg,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_ROW, .gap_x = 8,
        .cell_w = 120, .cell_h = 35,
        .scrollable = false,
    }, ...);
```

布局参数从代码中分散的 `lv_obj_set_style_*` 调用，收拢为**一个结构体字面量**，一目了然。

---

## 五、工厂模式：widget_factory 动态创建

### 5.1 注册表设计

`widget_factory` 实现了按类型名字符串注册创建函数的**注册表模式**：

```c
typedef widget_unit_t *(*widget_create_fn)(lv_obj_t *parent);

static struct {
    const char *type_name;
    widget_create_fn create;
} s_entries[16];
static int s_entry_count = 0;
```

初始化阶段注册四种控件：

```c
void custom_init() {
    commodity_card_register();   // "commodity_card" -> commodity_card_create()
    banner_item_register();      // "banner_item"    -> banner_item_create()
    cart_item_register();        // "cart_item"      -> cart_item_create()
    navi_item_register();        // "navi_item"      -> navi_item_create()
}
```

### 5.2 运行时动态创建

组件工厂通过类型名动态创建控件，并自动加入应用树：

```c
// component_factory.c - 创建 banner 组件
for (int i = 0; i < initial_data->count; i++) {
    widget_unit_t *item =
        widget_factory_create("banner_item", comp->container->base.obj);
    if (item) {
        comp->container->add_child(comp->container, &item->base);
        item->bind(item, &path_data);  // 数据绑定
    }
}
```

每种控件的 `create()` 函数负责：

1. 从 RT-Thread 内存池分配 `widget_unit_t` 和私有数据结构
2. 创建 LVGL 渲染对象并设置样式
3. 设置 `bind()` 和 `destroy()` 虚函数
4. 返回初始化完成的 `widget_unit_t*`

---

## 六、组件工厂：应用树的组装单元

### 6.1 组件 = 容器 + 子控件 + 控制器

`component_factory` 在应用树之上封装**完整的业务组件**，每个组件由三部分组成：

```c
typedef struct {
    layout_container_t *container;     // 应用树容器
    banner_carousel_t *carousel;       // 控制器（不持有容器）
} banner_component_t;
```

以 banner 组件为例，`banner_component_create()` 的组装流程：

```
1. layout_container_wrap(parent, strategy)  → 创建应用树容器
2. widget_factory_create("banner_item", ...) × N  → 批量创建子控件
3. container->add_child(...) × N  → 加入应用树
4. item->bind(...) × N  → 绑定数据
5. banner_carousel_create(container, ...)  → 挂载控制器
```

### 6.2 控制器-容器分离

`banner_carousel` 是一个**纯粹的控制器**，它不创建也不拥有容器和子控件，只附加行为：

```c
struct banner_carousel {
    layout_container_t *container;    // 引用，非拥有
    lv_timer_t *auto_scroll_timer;   // 自动滚动定时器
    lv_obj_t **dots;                  // 指示器圆点
    int32_t last_scroll_x;           // 手势检测状态
    bool is_animating;
};
```

这种分离使得：

- 容器可以独立于控制器被销毁和重建
- 控制器可以在运行时切换到不同的容器实例
- 数据热更新时，控制器暂停 → 容器 clear → 重建子控件 → 控制器恢复

### 6.3 数据热更新

`banner_carousel_update_data()` 展示了应用树的热更新流程：

```c
void banner_carousel_update_data(banner_carousel_t *carousel,
                                  const carousel_data_t *data) {
    // 1. 暂停自动滚动
    lv_timer_pause(carousel->auto_scroll_timer);

    // 2. 销毁旧指示器
    destroy_dots(carousel);

    // 3. 清空容器子节点（深度优先销毁所有旧 banner_item）
    carousel->container->clear(carousel->container);

    // 4. 从新数据创建子控件
    for (int i = 0; i < data->count; i++) {
        widget_unit_t *item = widget_factory_create("banner_item", ...);
        carousel->container->add_child(carousel->container, &item->base);
        item->bind(item, &path_data);
    }

    // 5. 重建指示器（延迟 50ms 等 LVGL 布局完成）
    schedule_dots_init(carousel);

    // 6. 恢复自动滚动
    lv_timer_resume(carousel->auto_scroll_timer);
}
```

整个流程对调用者透明——只需传入新数据，应用树自动完成**销毁旧节点 → 创建新节点 → 数据绑定 → 布局重排**。

---

## 七、事件驱动：跨组件通信与配置热更新

### 7.1 事件总线

`event_bus` 实现发布-订阅解耦，定义了 8 种事件类型：

```c
typedef enum {
    EVT_CART_ITEM_ADDED,       // 商品加购
    EVT_CART_ITEM_REMOVED,     // 购物车移除
    EVT_CART_COUNT_CHANGED,    // 数量变化
    EVT_CART_PRICE_CHANGED,    // 价格变化
    EVT_BANNER_CONFIG_CHANGED, // Banner 配置变更
    EVT_COMMODITY_CONFIG_CHANGED, // 商品配置变更
    EVT_NAVI_CONFIG_CHANGED,   // 导航配置变更
    EVT_NAVI_ITEM_CLICKED,     // 导航点击
} event_type_t;
```

### 7.2 配置热更新的完整链路

Web 配置触发的事件链路，展示了从 HTTP 请求到 UI 重渲染的完整流程：

```
HTTP POST /api/config
  ├── HTTP 线程: 解析 JSON → 写入 s_pending_json → 设置 s_pending_update=1
  ├── HTTP 线程: 保存 JSON 到 /sdcard/config/banner.json（持久化）
  └── LVGL 定时器 (200ms): 检测 s_pending_update
      ├── 发布 EVT_BANNER_CONFIG_CHANGED (data=json_str)
      └── on_banner_config_changed() 订阅者:
          ├── carousel_data_from_json(json_str, &data)
          └── banner_component_update(comp, &data)
              └── banner_carousel_update_data()
                  ├── 暂停定时器
                  ├── container->clear()     ← 销毁旧应用树子节点
                  ├── widget_factory_create() ← 创建新应用树子节点
                  ├── container->add_child()  ← 加入应用树
                  ├── item->bind()            ← 数据绑定
                  ├── schedule_dots_init()    ← 重建指示器
                  └── 恢复定时器
```

**关键**：HTTP 线程绝不直接调用 LVGL API，仅设置 volatile 标志；所有 UI 操作在 LVGL 定时器的 LVGL 线程上下文中执行，保证线程安全。

### 7.3 跨组件联动

商品卡片加购不直接调用购物车 API，而是通过事件总线：

```
commodity_card 点击加购
    → event_bus_publish(EVT_CART_ITEM_ADDED, {name, price})
    → cart_module 订阅处理
        → cart_add_commodity()
        → event_bus_publish(EVT_CART_PRICE_CHANGED, {total_price})
        → qr_scanner 订阅处理
            → qr_scanner_set_payment_info()  // 支付金额始终与购物车同步
```

三个模块零直接依赖，完全通过事件总线通信。

---

## 八、从静态到动态：custom_init() 的替换流程

`custom_init()` 是系统组装入口，完整展示了从 AiUIBuilder 静态骨架到动态应用树的替换过程：

```
Step 1: 初始化基础设施
  event_bus_init() + async_loader_init()

Step 2: 初始化对象池
  banner_item_module_init()    // rt_mp_create, 容量=10
  commodity_module_init()      // rt_mp_create, 容量=30
  cart_module_init()           // rt_mp_create, 容量=20
  navi_item_module_init()      // rt_mp_create, 容量=10

Step 3: 注册控件工厂
  commodity_card_register()    // "commodity_card" → create_fn
  banner_item_register()       // "banner_item"    → create_fn
  cart_item_register()         // "cart_item"      → create_fn
  navi_item_register()         // "navi_item"      → create_fn

Step 4: 获取 AiUIBuilder 生成的屏幕对象
  screen_t *scr = screen_get(&ui_manager);

Step 5: 删除静态子控件，替换为动态组件
  // Banner: 删除 3 个静态图片
  lv_obj_del(scr->image_banner_1);
  lv_obj_del(scr->image_banner_2);
  lv_obj_del(scr->image_banner_3);
  // Wrap 容器 + 动态填充
  s_banner_comp = banner_component_create(scr->container_banner, ...);

  // Commodity: 删除 6 个静态容器
  lv_obj_del(scr->container_main_1~6);
  // Wrap 容器 + 动态填充
  s_commodity_comp = commodity_component_create(scr->container_main, ...);

  // Cart: 删除 8 个静态控件
  lv_obj_del(scr->image_cart_item_bg_1~3);
  lv_obj_del(scr->button_cart_item_add_1);
  // ...
  // Wrap 容器（初始为空，事件驱动添加）
  s_cart_comp = cart_component_create(scr->container_cart_list_scroll, ...);

  // Navi: 删除 2 个静态容器
  lv_obj_del(scr->container_navi_1~2);
  // Wrap 容器 + 动态填充
  s_navi_comp = navi_component_create(scr->container_navi_bg, ...);

Step 6: 事件订阅 + 启动业务模块
  event_bus_subscribe(EVT_BANNER_CONFIG_CHANGED, ...);
  event_bus_subscribe(EVT_COMMODITY_CONFIG_CHANGED, ...);
  web_config_init();
  uart3_payment_init();
  qr_scanner_init();
```

**核心思路**：保留 AiUIBuilder 创建的**容器对象**（如 `container_banner`），删除其**静态子控件**，然后通过 Wrap 机制将容器纳入应用树管理，再通过控件工厂动态填充可数据驱动的子节点。

---

## 九、内存管理策略

### 9.1 对象池 vs 堆分配

| 对象                       | 分配方式      | 原因                              |
| -------------------------- | ------------- | --------------------------------- |
| `layout_container_t`       | `rt_malloc`   | 生命周期长，数量少（4个组件容器） |
| `layout_node_t **children` | `rt_malloc`   | 需倍增扩容，不适用固定大小池      |
| `widget_unit_t` + `priv`   | `rt_mp_alloc` | 高频创建/销毁，需避免碎片         |
| `navi_click_ctx_t`         | `rt_malloc`   | 与事件回调生命周期绑定            |

### 9.2 对象池容量对齐

每个控件模块独立管理内存池，容量与业务上限对齐：

- `banner_item`: 容量 10（最多 10 张横幅）
- `commodity_card`: 容量 30（最多 30 件商品）
- `cart_item`: 容量 20（最多 20 个购物车项）
- `navi_item`: 容量 10（最多 10 个导航标签）

---

## 十、架构总结

### 10.1 设计模式协同

```
component_factory (组装层: 容器 + 子控件 + 控制器)
widget_factory (工厂模式) | layout_strategy (策略模式) | event_bus (Pub-Sub)
layout_node (组合模式)
  container ─┬─ widget_unit (叶子)
             ├─ widget_unit
             └─ container ─┬─ widget_unit
                           └─ ...
LVGL 渲染树 (lv_obj_t)
```

### 10.2 核心价值

1. **Wrap 桥接**：`layout_container_wrap()` 是连接静态骨架与动态应用树的桥梁，保留可视化工具的快速原型能力，同时获得运行时灵活性。

2. **深度优先销毁**：`destroy()` 虚函数递归销毁子树，消除了手动 `lv_obj_del()` 散落各处的生命周期管理问题。

3. **数据驱动重建**：`clear()` → `widget_factory_create()` → `add_child()` → `bind()` 的标准化流程，使热更新逻辑高度统一。

4. **策略封装**：布局参数从分散的 API 调用收拢为 `layout_strategy_t` 结构体，组件创建时一目了然。

5. **事件解耦**：组件间零直接依赖，购物车不知道商品卡片，支付模块不知道购物车，Web 配置不知道任何 UI API。

---

## 十一、串口支付流程

### 11.1 帧协议

设备与 PC 主机通过 UART3 通信，采用自定义帧协议：

```
AA 55 CMD LEN_H LEN_L DATA... 0D 0A
```

| 方向      | CMD  | 名称              | 数据                           |
| --------- | ---- | ----------------- | ------------------------------ |
| PC → 设备 | 0x01 | CMD_QR_URL        | 二维码 URL (UTF-8)             |
| PC → 设备 | 0x02 | CMD_PAY_STATUS    | 支付状态码 (1字节)             |
| PC → 设备 | 0x03 | CMD_ORDER_NO      | 订单号 (UTF-8)                 |
| PC → 设备 | 0x04 | CMD_HEARTBEAT_ACK | 心跳应答                       |
| 设备 → PC | 0x81 | CMD_REQ_QR        | name\0amount                   |
| 设备 → PC | 0x82 | CMD_REQ_QUERY     | 空                             |
| 设备 → PC | 0x83 | CMD_REQ_CLOSE     | 空                             |
| 设备 → PC | 0x84 | CMD_HEARTBEAT     | 空                             |
| 设备 → PC | 0x85 | CMD_BARCODE_PAY   | auth_code\0[amount\0][subject] |

### 11.2 双支付模式

**模式一：用户扫码**

1. 用户点击结算按钮 → 解析购物车总价 → 显示支付模式选择弹窗
2. 用户选择"用户扫码" → 发送 `CMD_REQ_QR` 到 PC
3. PC 生成支付宝沙箱订单，返回 `CMD_QR_URL`
4. 设备显示 QR 码覆盖层（240×240，LVGL `lv_qrcode` 控件）
5. PC 轮询支付状态后返回 `CMD_PAY_STATUS`
6. 成功：显示"支付成功!"，2 秒后自动隐藏，清空购物车，恢复轮播

**模式二：设备扫码（条码支付）**

1. 用户选择"设备摄像头扫码"
2. 启动 DVP 摄像头 + quirc 扫描线程
3. 设置 UI 层 alpha=128 使视频层透视显示
4. quirc 扫描到 18~28 位纯数字（支付授权码）后，通过 `CMD_BARCODE_PAY` 发送至 PC
5. PC 调用支付宝条码支付 API，返回支付状态
6. 成功后流程同上

### 11.3 线程安全设计

UART3 RX 线程严格不调用任何 LVGL API。帧解析完成后仅入队到 SPSC 环形缓冲区。LVGL 定时器 (100ms) 在 LVGL 线程上下文出队处理，确保所有 UI 操作的线程安全。

---

## 十二、二维码扫描模块

### 12.1 硬件配置

- DVP 摄像头 (OV5640)，强制 QVGA 320×240 输出，降低内存占用（约 460KB）
- ArtInChip 视频输入框架 (`mpp_vin`, `drv_dvp`)
- Framebuffer 视频层 (`mpp_fb`) 用于摄像头预览
- quirc QR 码解码库

### 12.2 扫描流程

1. `qr_scanner_start()`：初始化 DVP（含 5 次重试容错）、配置传感器格式、创建 quirc 实例 (160×120，2 倍下采样)
2. 扫描线程循环：`dq_buf` → 丢弃前几帧 → 每 5 帧提取 Y 通道并下采样 → `quirc_decode`
3. `handle_qr_result()`：检测是否为支付授权码（18~28 位纯数字），同码 5 秒冷却，调用 `uart3_payment_send_barcode()` 发送

### 12.3 摄像头视图模式

通过 ArtInChip framebuffer 的视频层直接显示 DVP 采集帧：

- `qr_scanner_start_camera_view()`：打开 framebuffer，设置 UI 层 alpha=128
- DVP 缓冲区零拷贝推送到视频层，实现高性能画中画效果
- 停止时严格顺序：清标志 → 停扫描线程 → 停 DVP 流 → 等线程退出 → 禁用视频层 + 恢复 UI alpha → 释放资源

---

## 十三、Web 配置系统

### 13.1 架构

设备启动 WiFi AP（SSID: `AIC-Banner-Config`，密码: `12345678`，信道 6），IP 地址 192.168.1.1，运行 DHCP 服务器，在 80 端口启动 Mongoose HTTP 服务器。

### 13.2 API 接口

| 方法 | 路径                               | 功能                            |
| ---- | ---------------------------------- | ------------------------------- |
| GET  | `/`                                | 嵌入式 HTML 配置页面            |
| GET  | `/api/config`                      | 获取 Banner 配置 JSON           |
| GET  | `/api/commodity`                   | 获取商品配置 JSON               |
| GET  | `/api/navi`                        | 获取导航配置 JSON               |
| POST | `/api/config`                      | 更新 Banner 配置                |
| POST | `/api/commodity`                   | 更新商品配置                    |
| POST | `/api/navi`                        | 更新导航配置                    |
| POST | `/api/reset`                       | 重置为默认配置                  |
| POST | `/api/upload/{filename}`           | 上传 Banner 图片（校验 494×78） |
| POST | `/api/upload/commodity/{filename}` | 上传商品图片（校验 80×80）      |

### 13.3 安全措施

- 文件名白名单校验：仅允许字母、数字、`.`、`_`、`-`，长度不超过 64
- Content-Length 早期拒绝：超限直接返回 413
- 接收缓冲区限制：`recv_mbuf.len > 307200 + 4096` 时强制关闭连接
- 图片尺寸校验：解析 PNG/JPEG/BMP 文件头验证宽高，不匹配则删除并返回错误
- 价格格式校验：支持多种货币前缀，验证必须包含数字和可选小数部分

### 13.4 配置热更新

1. HTTP 线程接收 POST → 解析 JSON → 写入 `s_pending_json` 缓冲区 → 设置标志
2. 同时将 JSON 保存到 SD 卡文件实现持久化
3. LVGL 定时器 200ms 轮询检测标志 → 发布事件 → 各组件执行热更新

---

## 十四、异步加载机制

基于 RT-Thread 线程 + 双消息队列实现：

```
主线程 (LVGL)              工作线程
    |                                                 |
    | submit(task)                         |
    | ----[task_mq]---->              |
    |                                                 | work_fn(arg) → result
    |                                [result_mq]----> done_cb(result)
    |                                                 |
    | poll() <-- 排空 result_mq   |
```

- 工作线程栈：4096 字节，优先级 20
- 任务/结果队列容量：8 条
- `async_loader_poll()` 在 LVGL 定时器中调用，保证 `done_cb` 在 LVGL 线程安全执行

---

## 十五、技术亮点总结

1. **Wrap 桥接**：`layout_container_wrap()` 是连接静态骨架与动态应用树的桥梁，保留可视化工具的快速原型能力，同时获得运行时灵活性。

2. **深度优先销毁**：`destroy()` 虚函数递归销毁子树，消除了手动 `lv_obj_del()` 散落各处的生命周期管理问题。

3. **数据驱动重建**：`clear()` → `widget_factory_create()` → `add_child()` → `bind()` 的标准化流程，使热更新逻辑高度统一。

4. **策略封装**：布局参数从分散的 API 调用收拢为 `layout_strategy_t` 结构体，组件创建时一目了然。

5. **事件解耦**：组件间零直接依赖，购物车不知道商品卡片，支付模块不知道购物车，Web 配置不知道任何 UI API。

6. **嵌入式 Web 配置**：RT-Thread 上运行 Mongoose HTTP 服务器 + WiFi AP，手机连接热点即可修改配置，无需固件升级，配置自动持久化到 SD 卡。

7. **双支付模式**：支持"用户扫码"和"设备扫码"两种模式，通过 UART3 与 PC 主机通信，实现支付宝沙箱环境下的完整支付闭环。

8. **无锁线程间通信**：UART3 使用 SPSC 环形队列，Web 配置使用 volatile 标志 + 缓冲区，均避免重量级锁开销。

9. **内存池管理**：所有高频创建/销毁的控件使用 RT-Thread 内存池，容量与业务上限对齐，避免内存碎片。

10. **摄像头视图硬件加速**：DVP 缓冲区零拷贝推送到视频层，UI 层半透明实现画中画效果。
