---
title: "LVGL Dynamic Loading and Render Tree Management: Application Tree Architecture Practices for Smart Vending Terminal"
date: 2026-07-02
description: "Based on ArtInChip + LVGL v9 + RT-Thread smart vending terminal, an in-depth analysis of how the application tree (layout_node) manages the LVGL render tree, covering Composite pattern dual-tree mapping, Wrap bridging mechanism, strategy-driven Flex/Grid layout, widget factory dynamic creation, event-driven hot updates, and the replacement flow from static skeleton to dynamic components"
image: "smart-vending-terminal.png"
categories:
  - "Embedded"
tags:
  - "LVGL"
  - "RT-Thread"
  - "ArtInChip"
  - "Vending Machine"
  - "Embedded UI"
  - "Design Patterns"
---

## Preface

As the most active graphics library in the embedded space, LVGL's core is a tree-structured **render tree** (`lv_obj_t` object tree), where all visible widgets are nodes of this tree. However, in real-world applications, directly manipulating the render tree faces several problems: **static layouts cannot be dynamically updated**, **widget lifecycles are difficult to manage uniformly**, and **components are tightly coupled**.

This article uses a smart vending terminal based on **ArtInChip + LVGL v9 + RT-Thread** as a practical vehicle, providing an in-depth analysis of how to build an **application tree** (`layout_node` framework) on top of the LVGL render tree. Through the coordination of the Composite, Factory, Strategy, and Publish-Subscribe patterns, it enables dynamic loading, hot updates, and lifecycle management of the UI.

---

## 1. Dual-Tree Architecture: Render Tree vs Application Tree

### 1.1 Problem Background

The project uses AiUIBuilder v2.0.2 to generate a static UI skeleton (`ui_builder/screen.c`), creating widgets such as banner images, commodity cards, cart rows, and navigation buttons on an 800×480 screen. However, these widgets are **hard-coded static objects**:

```c
// screen.c (AiUIBuilder generated)
scr->image_banner_1 = lv_img_create(scr->container_banner);
scr->container_main_1 = lv_obj_create(scr->container_main);
// ...fixed quantity, fixed data, no runtime add/remove
```

Business requirements demand **runtime dynamic addition and removal of widgets** (variable product counts, dynamically added cart items, configurable banner replacements). Directly manipulating the render tree leads to:

- Object lifecycles scattered across callbacks, prone to leaks
- Inability to batch-destroy subtrees
- Layout parameters hard-coded, no unified adjustment
- Direct calls between components, tight coupling

### 1.2 Solution: Application Tree

Build an **application tree** on top of the LVGL render tree, where each application tree node holds a corresponding `lv_obj_t*` and adds lifecycle and layout management capabilities:

```
Application Tree (layout_node)
  container (strategy=Flex-Row)
  ├── widget_unit (banner_item, priv=...)
  ├── widget_unit (banner_item, priv=...)
  └── widget_unit (banner_item, priv=...)

Render Tree (lv_obj_t)
  lv_obj (LV_FLEX_FLOW_ROW)
  ├── lv_img (banner_1.png)
  ├── lv_img (banner_2.png)
  └── lv_img (banner_3.png)
```

Application tree nodes correspond one-to-one with render tree nodes via the `obj` pointer, but the application tree additionally manages:

| Management Dimension | Render Tree (lv_obj_t)                           | Application Tree (layout_node)                                  |
| -------------------- | ------------------------------------------------ | --------------------------------------------------------------- |
| Lifecycle            | `lv_obj_del()` manually scattered everywhere     | `destroy()` virtual function, depth-first automatic destruction |
| Child Management     | LVGL internal linked list, no business semantics | `children[]` dynamic array, capacity doubling                   |
| Layout Parameters    | Scattered across `lv_obj_set_style_*` calls      | `layout_strategy_t` unified encapsulation                       |
| Data Binding         | Manual per-field assignment                      | `bind()` / `update()` virtual functions                         |
| Private Data         | No standard mechanism                            | `widget_unit.priv` points to memory pool object                 |

---

## 2. Composite Pattern: layout_node Core Design

### 2.1 Three-Layer Type System

The core of the application tree is a **Composite pattern** three-layer type system:

```c
/* Abstract base class - virtual function table */
struct layout_node {
    lv_obj_t *obj;                          // Corresponding render tree node
    const char *type_name;                  // Type name ("container"/"banner_item"/...)
    layout_node_type_t node_type;           // CONTAINER or WIDGET
    layout_container_t *parent;             // Parent container reference

    void (*destroy)(layout_node_t *self);   // Virtual function: destroy
    void (*update)(layout_node_t *self,     // Virtual function: data update
                   const cJSON *data);
};

/* Container node - can contain children */
struct layout_container {
    layout_node_t base;                     // Inherit base class
    layout_strategy_t strategy;             // Layout strategy
    layout_node_t **children;               // Child node dynamic array
    int child_count;
    int child_capacity;
    // Method pointers: add_child, remove_child, clear, get_child
};

/* Leaf node - minimal renderable unit */
struct widget_unit {
    layout_node_t base;                     // Inherit base class
    void *priv;                             // Private data (allocated from memory pool)
    void (*bind)(widget_unit_t *self,       // Data binding function
                 const cJSON *data);
};
```

### 2.2 Depth-First Destruction

The `destroy()` virtual function implements **depth-first** subtree destruction, ensuring all child nodes are reclaimed before their parent:

```c
static void container_destroy(layout_node_t *self_base) {
    layout_container_t *self = (layout_container_t *)self_base;

    // 1. Destroy all children first (depth-first)
    for (int i = 0; i < self->child_count; i++)
        layout_node_destroy(self->children[i]);

    // 2. Free the children array
    rt_free(self->children);

    // 3. Destroy the render tree object
    lv_obj_del(self->base.obj);

    // 4. Free self
    rt_free(self);
}
```

The generic destruction dispatcher achieves polymorphism through the virtual function table:

```c
void layout_node_destroy(layout_node_t *node) {
    if (node && node->destroy)
        node->destroy(node);
}
```

This means the caller does not need to know the concrete type of the node; a single `layout_node_destroy()` call can destroy the entire subtree.

### 2.3 Child Node Dynamic Array

Containers use a dynamic array to manage child nodes, with an initial capacity of 8 and doubling expansion:

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

`add_child` automatically expands capacity and applies the child size from the layout strategy:

```c
void layout_container_add_child(layout_container_t *self, layout_node_t *child) {
    children_ensure_capacity(self, self->child_count + 1);
    child->parent = self;
    self->children[self->child_count++] = child;
    layout_strategy_apply_child_size(child->obj, &self->strategy);
}
```

After `clear()` removes all child nodes, if the capacity exceeds 4 times the initial value, it shrinks to avoid memory bloat during long-running sessions:

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

## 3. Wrap Bridging: From Static Skeleton to Dynamic Container

### 3.1 Two Creation Modes

`layout_container` provides two creation modes, which is the most critical design of the entire architecture:

| Mode       | Function                    | Creates LVGL Object           | Destroys LVGL Object                              | Usage                              |
| ---------- | --------------------------- | ----------------------------- | ------------------------------------------------- | ---------------------------------- |
| **Create** | `layout_container_create()` | Creates new `lv_obj_create()` | `container_destroy()` calls `lv_obj_del()`        | Brand-new dynamic container        |
| **Wrap**   | `layout_container_wrap()`   | Reuses existing object        | `container_destroy_wrapped()` does **not** delete | Bridges AiUIBuilder static objects |

### 3.2 Wrap Mechanism Explained

`layout_container_wrap()` receives an **existing `lv_obj_t*`**, wraps it as an application tree container, but **does not change object ownership**:

```c
layout_container_t *layout_container_wrap(lv_obj_t *existing_obj,
                                          const layout_strategy_t *strategy) {
    layout_container_t *self = rt_malloc(sizeof(layout_container_t));
    memset(self, 0, sizeof(*self));

    // Reuse existing object, rather than creating a new one
    self->base.obj = existing_obj;

    // Apply layout strategy on the existing object
    layout_strategy_apply(existing_obj, &self->strategy);

    // Use special destroy function - does not delete LVGL object
    self->base.destroy = container_destroy_wrapped;

    return self;
}
```

The key difference in `container_destroy_wrapped()`:

```c
static void container_destroy_wrapped(layout_node_t *self_base) {
    layout_container_t *self = (layout_container_t *)self_base;

    // Still destroy all children (depth-first)
    for (int i = 0; i < self->child_count; i++)
        layout_node_destroy(self->children[i]);
    rt_free(self->children);

    // Key: do NOT delete the LVGL object, it is managed externally (by screen_t)
    self->base.obj = NULL;

    rt_free(self);
}
```

### 3.3 Actual Bridging Flow

In `custom_init()`, the bridging flow is as follows:

```
1. AiUIBuilder generates:  scr->container_banner (lv_obj_t)
                           ├── scr->image_banner_1 (static child widget)
                           ├── scr->image_banner_2
                           └── scr->image_banner_3

2. Delete static children: lv_obj_del(scr->image_banner_1);
                           lv_obj_del(scr->image_banner_2);
                           lv_obj_del(scr->image_banner_3);

3. Wrap container:         banner_comp->container = layout_container_wrap(
                               scr->container_banner, &flex_row_strategy);

4. Dynamically fill children: for each image_path:
                               item = widget_factory_create("banner_item", container->base.obj);
                               container->add_child(container, &item->base);
                               item->bind(item, &path_data);
```

This way, the `container_banner` created by AiUIBuilder still exists in the render tree (with its lifecycle managed by `screen_t`), but its **child widgets have been entirely replaced with dynamically managed application tree nodes**, supporting runtime add/remove, data binding, and hot updates.

---

## 4. Strategy Pattern: layout_strategy Layout Engine

### 4.1 Strategy Configuration Structure

`layout_strategy_t` unifies LVGL's scattered layout APIs into a single configuration structure:

```c
typedef struct {
    layout_type_t type;       // FLEX_ROW / FLEX_COLUMN / GRID
    int gap_x, gap_y;        // Gaps
    int pad_x, pad_y;        // Padding
    int cols;                 // Grid column count
    int cell_w, cell_h;      // Child fixed size (0=auto)
    bool scrollable;          // Whether scrollable
    lv_dir_t scroll_dir;      // Scroll direction
} layout_strategy_t;
```

### 4.2 Strategy Application

`layout_strategy_apply()` maps the strategy to LVGL Flex API in a single pass:

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
        // Grid = row-wrap, children wrap to new track via FLEX_IN_NEW_TRACK
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

### 4.3 Layout Strategies for the Four Major Components

In `custom_init()`, the four major components each use different strategies:

```c
// Banner carousel - Flex-Row, horizontal scroll
s_banner_comp = banner_component_create(
    scr->container_banner,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_ROW, .cell_w = 494, .cell_h = 78,
        .scrollable = true, .scroll_dir = LV_DIR_HOR,
    }, ...);

// Product display - Grid 3 columns, vertical scroll
s_commodity_comp = commodity_component_create(
    scr->container_main,
    &(layout_strategy_t){
        .type = LAYOUT_GRID, .cols = 3,
        .gap_x = 12, .gap_y = 12, .cell_w = 240, .cell_h = 100,
        .scrollable = true, .scroll_dir = LV_DIR_VER,
    }, ...);

// Shopping cart - Flex-Column, vertical scroll
s_cart_comp = cart_component_create(
    scr->container_cart_list_scroll,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_COLUMN, .gap_y = 10,
        .cell_w = 750, .cell_h = 70,
        .scrollable = true, .scroll_dir = LV_DIR_VER,
    });

// Navigation bar - Flex-Row, non-scrollable
s_navi_comp = navi_component_create(
    scr->container_navi_bg,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_ROW, .gap_x = 8,
        .cell_w = 120, .cell_h = 35,
        .scrollable = false,
    }, ...);
```

Layout parameters that were previously scattered across `lv_obj_set_style_*` calls are now consolidated into **a single struct literal**, clear at a glance.

---

## 5. Factory Pattern: widget_factory Dynamic Creation

### 5.1 Registry Design

`widget_factory` implements a **registry pattern** that registers creation functions by type name string:

```c
typedef widget_unit_t *(*widget_create_fn)(lv_obj_t *parent);

static struct {
    const char *type_name;
    widget_create_fn create;
} s_entries[16];
static int s_entry_count = 0;
```

During initialization, four widget types are registered:

```c
void custom_init() {
    commodity_card_register();   // "commodity_card" -> commodity_card_create()
    banner_item_register();      // "banner_item"    -> banner_item_create()
    cart_item_register();        // "cart_item"      -> cart_item_create()
    navi_item_register();        // "navi_item"      -> navi_item_create()
}
```

### 5.2 Runtime Dynamic Creation

Component factories dynamically create widgets by type name and automatically add them to the application tree:

```c
// component_factory.c - create banner component
for (int i = 0; i < initial_data->count; i++) {
    widget_unit_t *item =
        widget_factory_create("banner_item", comp->container->base.obj);
    if (item) {
        comp->container->add_child(comp->container, &item->base);
        item->bind(item, &path_data);  // Data binding
    }
}
```

Each widget's `create()` function is responsible for:

1. Allocating `widget_unit_t` and private data structure from RT-Thread memory pool
2. Creating LVGL render objects and setting styles
3. Setting `bind()` and `destroy()` virtual functions
4. Returning the fully initialized `widget_unit_t*`

---

## 6. Component Factory: Assembly Units of the Application Tree

### 6.1 Component = Container + Child Widgets + Controller

`component_factory` encapsulates **complete business components** on top of the application tree, each component consisting of three parts:

```c
typedef struct {
    layout_container_t *container;     // Application tree container
    banner_carousel_t *carousel;       // Controller (does not own the container)
} banner_component_t;
```

Taking the banner component as an example, the assembly flow of `banner_component_create()`:

```
1. layout_container_wrap(parent, strategy)  → Create application tree container
2. widget_factory_create("banner_item", ...) × N  → Batch create child widgets
3. container->add_child(...) × N  → Add to application tree
4. item->bind(...) × N  → Bind data
5. banner_carousel_create(container, ...)  → Mount controller
```

### 6.2 Controller-Container Separation

`banner_carousel` is a **pure controller** — it does not create or own containers and child widgets, it only attaches behavior:

```c
struct banner_carousel {
    layout_container_t *container;    // Reference, not ownership
    lv_timer_t *auto_scroll_timer;   // Auto-scroll timer
    lv_obj_t **dots;                  // Indicator dots
    int32_t last_scroll_x;           // Gesture detection state
    bool is_animating;
};
```

This separation enables:

- The container can be destroyed and rebuilt independently of the controller
- The controller can switch to a different container instance at runtime
- During data hot-updates: controller pauses → container clears → child widgets rebuilt → controller resumes

### 6.3 Data Hot-Update

`banner_carousel_update_data()` demonstrates the application tree's hot-update flow:

```c
void banner_carousel_update_data(banner_carousel_t *carousel,
                                  const carousel_data_t *data) {
    // 1. Pause auto-scroll
    lv_timer_pause(carousel->auto_scroll_timer);

    // 2. Destroy old indicators
    destroy_dots(carousel);

    // 3. Clear container children (depth-first destroy all old banner_items)
    carousel->container->clear(carousel->container);

    // 4. Create child widgets from new data
    for (int i = 0; i < data->count; i++) {
        widget_unit_t *item = widget_factory_create("banner_item", ...);
        carousel->container->add_child(carousel->container, &item->base);
        item->bind(item, &path_data);
    }

    // 5. Rebuild indicators (delayed 50ms to wait for LVGL layout completion)
    schedule_dots_init(carousel);

    // 6. Resume auto-scroll
    lv_timer_resume(carousel->auto_scroll_timer);
}
```

The entire flow is transparent to the caller — just pass in the new data, and the application tree automatically completes **destroy old nodes → create new nodes → data binding → layout reflow**.

---

## 7. Event-Driven: Cross-Component Communication and Config Hot-Updates

### 7.1 Event Bus

`event_bus` implements publish-subscribe decoupling, defining 8 event types:

```c
typedef enum {
    EVT_CART_ITEM_ADDED,       // Item added to cart
    EVT_CART_ITEM_REMOVED,     // Cart item removed
    EVT_CART_COUNT_CHANGED,    // Quantity changed
    EVT_CART_PRICE_CHANGED,    // Price changed
    EVT_BANNER_CONFIG_CHANGED, // Banner config changed
    EVT_COMMODITY_CONFIG_CHANGED, // Commodity config changed
    EVT_NAVI_CONFIG_CHANGED,   // Navigation config changed
    EVT_NAVI_ITEM_CLICKED,     // Navigation clicked
} event_type_t;
```

### 7.2 Complete Config Hot-Update Chain

The event chain triggered by Web configuration demonstrates the complete flow from HTTP request to UI re-rendering:

```
HTTP POST /api/config
  ├── HTTP thread: Parse JSON → write to s_pending_json → set s_pending_update=1
  ├── HTTP thread: Save JSON to /sdcard/config/banner.json (persistence)
  └── LVGL timer (200ms): Detect s_pending_update
      ├── Publish EVT_BANNER_CONFIG_CHANGED (data=json_str)
      └── on_banner_config_changed() subscriber:
          ├── carousel_data_from_json(json_str, &data)
          └── banner_component_update(comp, &data)
              └── banner_carousel_update_data()
                  ├── Pause timer
                  ├── container->clear()     ← Destroy old application tree children
                  ├── widget_factory_create() ← Create new application tree children
                  ├── container->add_child()  ← Add to application tree
                  ├── item->bind()            ← Data binding
                  ├── schedule_dots_init()    ← Rebuild indicators
                  └── Resume timer
```

**Key point**: The HTTP thread never directly calls LVGL APIs; it only sets a volatile flag. All UI operations execute within the LVGL timer's LVGL thread context, ensuring thread safety.

### 7.3 Cross-Component Coordination

Adding a commodity card to cart does not directly call the cart API, but goes through the event bus:

```
commodity_card click add-to-cart
    → event_bus_publish(EVT_CART_ITEM_ADDED, {name, price})
    → cart_module subscriber handles
        → cart_add_commodity()
        → event_bus_publish(EVT_CART_PRICE_CHANGED, {total_price})
        → qr_scanner subscriber handles
            → qr_scanner_set_payment_info()  // Payment amount always stays in sync with cart
```

Three modules with zero direct dependencies, communicating entirely through the event bus.

---

## 8. From Static to Dynamic: The custom_init() Replacement Flow

`custom_init()` is the system assembly entry point, fully demonstrating the replacement process from AiUIBuilder static skeleton to dynamic application tree:

```
Step 1: Initialize infrastructure
  event_bus_init() + async_loader_init()

Step 2: Initialize object pools
  banner_item_module_init()    // rt_mp_create, capacity=10
  commodity_module_init()      // rt_mp_create, capacity=30
  cart_module_init()           // rt_mp_create, capacity=20
  navi_item_module_init()      // rt_mp_create, capacity=10

Step 3: Register widget factories
  commodity_card_register()    // "commodity_card" → create_fn
  banner_item_register()       // "banner_item"    → create_fn
  cart_item_register()         // "cart_item"      → create_fn
  navi_item_register()         // "navi_item"      → create_fn

Step 4: Get AiUIBuilder-generated screen object
  screen_t *scr = screen_get(&ui_manager);

Step 5: Delete static child widgets, replace with dynamic components
  // Banner: Delete 3 static images
  lv_obj_del(scr->image_banner_1);
  lv_obj_del(scr->image_banner_2);
  lv_obj_del(scr->image_banner_3);
  // Wrap container + dynamic fill
  s_banner_comp = banner_component_create(scr->container_banner, ...);

  // Commodity: Delete 6 static containers
  lv_obj_del(scr->container_main_1~6);
  // Wrap container + dynamic fill
  s_commodity_comp = commodity_component_create(scr->container_main, ...);

  // Cart: Delete 8 static widgets
  lv_obj_del(scr->image_cart_item_bg_1~3);
  lv_obj_del(scr->button_cart_item_add_1);
  // ...
  // Wrap container (initially empty, event-driven addition)
  s_cart_comp = cart_component_create(scr->container_cart_list_scroll, ...);

  // Navi: Delete 2 static containers
  lv_obj_del(scr->container_navi_1~2);
  // Wrap container + dynamic fill
  s_navi_comp = navi_component_create(scr->container_navi_bg, ...);

Step 6: Event subscriptions + start business modules
  event_bus_subscribe(EVT_BANNER_CONFIG_CHANGED, ...);
  event_bus_subscribe(EVT_COMMODITY_CONFIG_CHANGED, ...);
  web_config_init();
  uart3_payment_init();
  qr_scanner_init();
```

**Core idea**: Preserve the **container objects** created by AiUIBuilder (e.g., `container_banner`), delete their **static child widgets**, then use the Wrap mechanism to bring the containers under application tree management, and finally use the widget factory to dynamically fill them with data-driven child nodes.

---

## 9. Memory Management Strategy

### 9.1 Object Pool vs Heap Allocation

| Object                     | Allocation Method | Reason                                                     |
| -------------------------- | ----------------- | ---------------------------------------------------------- |
| `layout_container_t`       | `rt_malloc`       | Long lifecycle, few instances (4 component containers)     |
| `layout_node_t **children` | `rt_malloc`       | Needs doubling expansion, not suitable for fixed-size pool |
| `widget_unit_t` + `priv`   | `rt_mp_alloc`     | High-frequency create/destroy, must avoid fragmentation    |
| `navi_click_ctx_t`         | `rt_malloc`       | Lifecycle bound to event callback                          |

### 9.2 Object Pool Capacity Alignment

Each widget module independently manages its memory pool, with capacity aligned to business limits:

- `banner_item`: capacity 10 (max 10 banners)
- `commodity_card`: capacity 30 (max 30 products)
- `cart_item`: capacity 20 (max 20 cart items)
- `navi_item`: capacity 10 (max 10 navigation tabs)

---

## 10. Architecture Summary

### 10.1 Design Pattern Coordination

```
component_factory (Assembly: container + child widgets + controller)
widget_factory (Factory) | layout_strategy (Strategy) | event_bus (Pub-Sub)
layout_node (Composite)
  container ─┬─ widget_unit (leaf)
             ├─ widget_unit
             └─ container ─┬─ widget_unit
                           └─ ...
LVGL Render Tree (lv_obj_t)
```

### 10.2 Core Values

1. **Wrap Bridging**: `layout_container_wrap()` is the bridge connecting the static skeleton with the dynamic application tree, preserving the rapid prototyping capability of the visual tool while gaining runtime flexibility.

2. **Depth-First Destruction**: The `destroy()` virtual function recursively destroys subtrees, eliminating the problem of manual `lv_obj_del()` calls scattered everywhere for lifecycle management.

3. **Data-Driven Rebuilding**: The standardized flow of `clear()` → `widget_factory_create()` → `add_child()` → `bind()` makes hot-update logic highly uniform.

4. **Strategy Encapsulation**: Layout parameters are consolidated from scattered API calls into a `layout_strategy_t` struct, making component creation clear at a glance.

5. **Event Decoupling**: Zero direct dependencies between components — the cart doesn't know about commodity cards, the payment module doesn't know about the cart, and Web config doesn't know any UI API.

---

## 11. UART Payment Flow

### 11.1 Frame Protocol

The device communicates with the PC host via UART3, using a custom frame protocol:

```
AA 55 CMD LEN_H LEN_L DATA... 0D 0A
```

| Direction   | CMD  | Name              | Data                           |
| ----------- | ---- | ----------------- | ------------------------------ |
| PC → Device | 0x01 | CMD_QR_URL        | QR code URL (UTF-8)            |
| PC → Device | 0x02 | CMD_PAY_STATUS    | Payment status code (1 byte)   |
| PC → Device | 0x03 | CMD_ORDER_NO      | Order number (UTF-8)           |
| PC → Device | 0x04 | CMD_HEARTBEAT_ACK | Heartbeat ACK                  |
| Device → PC | 0x81 | CMD_REQ_QR        | name\0amount                   |
| Device → PC | 0x82 | CMD_REQ_QUERY     | empty                          |
| Device → PC | 0x83 | CMD_REQ_CLOSE     | empty                          |
| Device → PC | 0x84 | CMD_HEARTBEAT     | empty                          |
| Device → PC | 0x85 | CMD_BARCODE_PAY   | auth_code\0[amount\0][subject] |

### 11.2 Dual Payment Mode

**Mode 1: User Scans QR Code**

1. User clicks checkout button → parse cart total price → show payment mode selection popup
2. User selects "User Scans QR" → send `CMD_REQ_QR` to PC
3. PC generates Alipay sandbox order, returns `CMD_QR_URL`
4. Device displays QR code overlay (240×240, LVGL `lv_qrcode` widget)
5. PC polls payment status and returns `CMD_PAY_STATUS`
6. On success: display "Payment Successful!", auto-hide after 2 seconds, clear cart, restore carousel

**Mode 2: Device Scans Barcode (Barcode Payment)**

1. User selects "Device Camera Scans QR"
2. Start DVP camera + quirc scanning thread
3. Set UI layer alpha=128 to make video layer show through
4. After quirc scans an 18~28 digit pure numeric string (payment authorization code), send it to PC via `CMD_BARCODE_PAY`
5. PC calls Alipay barcode payment API, returns payment status
6. On success, the flow is the same as above

### 11.3 Thread Safety Design

The UART3 RX thread strictly does not call any LVGL APIs. After frame parsing is complete, it only enqueues data to an SPSC ring buffer. An LVGL timer (100ms) dequeues and processes in the LVGL thread context, ensuring thread safety for all UI operations.

---

## 12. QR Code Scanner Module

### 12.1 Hardware Configuration

- DVP camera (OV5640), forced QVGA 320×240 output to reduce memory usage (~460KB)
- ArtInChip video input framework (`mpp_vin`, `drv_dvp`)
- Framebuffer video layer (`mpp_fb`) for camera preview
- quirc QR code decoding library

### 12.2 Scanning Flow

1. `qr_scanner_start()`: Initialize DVP (with 5-retry fault tolerance), configure sensor format, create quirc instance (160×120, 2x downsampled)
2. Scanning thread loop: `dq_buf` → discard first few frames → extract Y channel and downsample every 5 frames → `quirc_decode`
3. `handle_qr_result()`: Detect if it is a payment authorization code (18~28 digit pure numeric), same-code 5-second cooldown, call `uart3_payment_send_barcode()` to send

### 12.3 Camera View Mode

Display DVP captured frames directly through the ArtInChip framebuffer video layer:

- `qr_scanner_start_camera_view()`: Open framebuffer, set UI layer alpha=128
- DVP buffer zero-copy push to video layer, achieving high-performance picture-in-picture effect
- Strict stop sequence: clear flag → stop scanning thread → stop DVP stream → wait for thread exit → disable video layer + restore UI alpha → release resources

---

## 13. Web Configuration System

### 13.1 Architecture

The device starts a WiFi AP (SSID: `AIC-Banner-Config`, password: `12345678`, channel 6), IP address 192.168.1.1, runs a DHCP server, and starts a Mongoose HTTP server on port 80.

### 13.2 API Endpoints

| Method | Path                               | Function                                |
| ------ | ---------------------------------- | --------------------------------------- |
| GET    | `/`                                | Embedded HTML configuration page        |
| GET    | `/api/config`                      | Get Banner configuration JSON           |
| GET    | `/api/commodity`                   | Get commodity configuration JSON        |
| GET    | `/api/navi`                        | Get navigation configuration JSON       |
| POST   | `/api/config`                      | Update Banner configuration             |
| POST   | `/api/commodity`                   | Update commodity configuration          |
| POST   | `/api/navi`                        | Update navigation configuration         |
| POST   | `/api/reset`                       | Reset to default configuration          |
| POST   | `/api/upload/{filename}`           | Upload Banner image (validate 494×78)   |
| POST   | `/api/upload/commodity/{filename}` | Upload commodity image (validate 80×80) |

### 13.3 Security Measures

- Filename whitelist validation: only allows letters, digits, `.`, `_`, `-`, max length 64
- Content-Length early rejection: returns 413 immediately if exceeded
- Receive buffer limit: force close connection when `recv_mbuf.len > 307200 + 4096`
- Image dimension validation: parse PNG/JPEG/BMP file headers to verify width and height; delete and return error if mismatched
- Price format validation: supports various currency prefixes, must contain digits and optional decimal part

### 13.4 Configuration Hot-Update

1. HTTP thread receives POST → parse JSON → write to `s_pending_json` buffer → set flag
2. Simultaneously save JSON to SD card file for persistence
3. LVGL timer polls flag every 200ms → publish event → each component performs hot-update

---

## 14. Asynchronous Loading Mechanism

Implemented based on RT-Thread thread + dual message queues:

```
Main Thread (LVGL)            Worker Thread
    |                                                 |
    | submit(task)                         |
    | ----[task_mq]---->              |
    |                                                 | work_fn(arg) → result
    |                                [result_mq]----> done_cb(result)
    |                                                 |
    | poll() <-- drain result_mq  |
```

- Worker thread stack: 4096 bytes, priority 20
- Task/result queue capacity: 8 entries each
- `async_loader_poll()` is called in an LVGL timer, ensuring `done_cb` executes safely in the LVGL thread context

---

## 15. Technical Highlights

1. **Wrap Bridging**: `layout_container_wrap()` is the bridge connecting the static skeleton with the dynamic application tree, preserving the rapid prototyping capability of the visual tool while gaining runtime flexibility.

2. **Depth-First Destruction**: The `destroy()` virtual function recursively destroys subtrees, eliminating the problem of manual `lv_obj_del()` calls scattered everywhere for lifecycle management.

3. **Data-Driven Rebuilding**: The standardized flow of `clear()` → `widget_factory_create()` → `add_child()` → `bind()` makes hot-update logic highly uniform.

4. **Strategy Encapsulation**: Layout parameters are consolidated from scattered API calls into a `layout_strategy_t` struct, making component creation clear at a glance.

5. **Event Decoupling**: Zero direct dependencies between components — the cart doesn't know about commodity cards, the payment module doesn't know about the cart, and Web config doesn't know any UI API.

6. **Embedded Web Configuration**: Running a Mongoose HTTP server + WiFi AP on RT-Thread, a phone can connect to the hotspot to modify configuration without firmware upgrades; configuration is automatically persisted to the SD card.

7. **Dual Payment Mode**: Supports both "User Scans QR" and "Device Scans QR" modes, communicating with the PC host via UART3, achieving a complete payment loop in the Alipay sandbox environment.

8. **Lock-Free Inter-Thread Communication**: UART3 uses an SPSC ring queue, Web configuration uses volatile flags + buffers, both avoiding heavyweight lock overhead.

9. **Memory Pool Management**: All high-frequency create/destroy widgets use RT-Thread memory pools, with capacity aligned to business limits, avoiding memory fragmentation.

10. **Camera View Hardware Acceleration**: DVP buffer zero-copy push to video layer, UI layer semi-transparent to achieve picture-in-picture effect.
