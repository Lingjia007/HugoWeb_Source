---
title: "LVGL 動態載入與渲染樹管理：智慧售貨機終端的應用樹架構實踐"
date: 2026-07-02
description: "基於立創·衡山派D133EBS開發板 + LVGL v9 + RT-Thread 的智慧售貨機終端，深入剖析應用樹（layout_node）如何管理 LVGL 渲染樹，涵蓋組合模式雙樹映射、Wrap 橋接機制、策略驅動的 Flex/Grid 佈局、控件工廠動態創建、事件驅動熱更新、靜態骨架到動態元件的替換流程等核心技術詳解"
image: "smart-vending-terminal.png"
categories:
  - "嵌入式"
  - "GUI開發"
tags:
  - "LVGL"
  - "RT-Thread"
  - "ArtInChip"
  - "售貨機"
  - "嵌入式UI"
  - "設計模式"
---

## 前言

LVGL 作為嵌入式領域最活躍的圖形庫，其核心是一個樹形的**渲染樹**（`lv_obj_t` 物件樹），所有可見控件都是這棵樹的節點。但在實際業務中，直接操作渲染樹面臨諸多問題：**靜態佈局無法動態更新**、**控件生命週期難以統一管理**、**元件間耦合緊密**。

本文以一個基於 **立創·衡山派D133EBS開發板 + LVGL v9 + RT-Thread** 的智慧售貨機終端為實踐載體，深入剖析如何在 LVGL 渲染樹之上構建一層**應用樹**（`layout_node` 框架），通過組合模式、工廠模式、策略模式和發布-訂閱模式的協同，實現 UI 的動態載入、熱更新和生命週期管理。

{{< bilibili BV1e3Kh6ZEad >}}

---

## 一、雙樹架構：渲染樹 vs 應用樹

### 1.1 問題背景

專案由 AiUIBuilder v2.0.2 生成靜態 UI 骨架（`ui_builder/screen.c`），在 800×480 螢幕上創建了橫幅圖片、商品卡片、購物車行、導航按鈕等控件。但這些控件是**硬編碼的靜態物件**：

```c
// screen.c (AiUIBuilder 生成)
scr->image_banner_1 = lv_img_create(scr->container_banner);
scr->container_main_1 = lv_obj_create(scr->container_main);
// ...固定數量、固定資料、無法執行時增刪
```

![AiUIBuilder 靜態 UI 骨架](uibuilder.png)

業務需求要求**執行時動態增刪控件**（商品數量可變、購物車項動態添加、橫幅可配置替換），直接操作渲染樹會導致：

- 物件生命週期分散在各回呼中，容易洩漏
- 無法批次銷毀子樹
- 佈局參數硬編碼，無法統一調整
- 元件間直接呼叫，緊耦合

#### 1.1.1 LVGL 原生的四大侷限

LVGL 是一個優秀的輕量級嵌入式圖形庫，但其原生設計在應對複雜、動態、多層級的 UI 應用時會顯現以下侷限：

| 侷限維度                       | 具體表現                                                                                                  | 原生做法痛點                                                                               |
| ------------------------------ | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **渲染狀態 vs 業務狀態**       | `lv_obj_t` 保存的是渲染狀態（坐標、尺寸、顏色），但不知道業務含義（如購物車數量是「3件商品」還是「3元總價」） | `lv_label_set_text(cart_label, "3")` 把業務邏輯和 UI 渲染死綁在一起，一旦物件銷毀就野指標  |
| **靜態配置 vs 動態生命週期**   | LVGL 鼓勵啟動時靜態配置，但現代嵌入式 UI 高度動態（商品列表從網路下發，數量隨時變）                       | 手動迴圈 `lv_obj_create`，手動計算坐標，大量 `lv_obj_del` 導致記憶體碎片，頻繁創建銷毀卡頓   |
| **硬編碼佈局 vs 策略化佈局**   | Flex/Grid 僅是基礎排版，複雜業務規則（缺貨變灰、橫屏每行從3變5）難以配置                                  | 業務程式碼裡寫滿 `if (stock==0) lv_obj_add_state(card, LV_STATE_DISABLED)`，散落各處極難維護 |
| **單頁面思維 vs 頁面棧狀態機** | 沒有路由/頁面棧概念，切換頁面就是 `lv_obj_clean` 重建，無法保存狀態（返回列表頁滑動位置丟失）             | 每次重建從頭渲染，用戶交互狀態全部丟失                                                     |

#### 1.1.2 一個通俗的比喻

把開發嵌入式 UI 比作蓋房子：

- **LVGL 是磚頭、水泥和塗料**。直接 `lv_obj_create` 就是一塊一塊壘磚頭。
- **簡單介面（狗窩）**：幾個按鈕幾個文字，直接壘磚頭最快最合適，不需要架構。
- **複雜介面（三十層商業大廈）**：智慧終端多頁面、網路資料動態刷新、多種交互狀態，還堅持一塊一塊壘磚頭，最後一定會塌方。你需要**建築圖紙和施工框架**（應用樹），圖紙定義了哪裡是衛生間、哪裡是承重牆（業務結構），具體怎麼砌磚由施工隊（LVGL 渲染引擎）按圖紙執行。

#### 1.1.3 應用樹填補的空白

LVGL 本身不提供完整應用框架來管理複雜 UI 狀態和業務邏輯。應用樹（`layout_node` + `layout_strategy`）填補這一空白：

- **結構化**：用樹形組織 UI，清晰表達介面層級
- **策略化**：將佈局演算法抽象為可替換的策略
- **元件化**：通過工廠和物件池高效管理元件
- **非同步化**：背景載入資源，保證流暢度
- **解耦化**：用事件匯流排實現業務邏輯與 UI 的鬆耦合通信

從而將 LVGL 從一個單純的「畫圖工具」提升為能夠支撐大型、複雜、可維護嵌入式 UI 應用的開發框架。

### 1.2 解決方案：應用樹

在 LVGL 渲染樹之上構建一棵**應用樹**，每個應用樹節點持有對應的 `lv_obj_t*`，並附加生命週期和佈局管理能力：

```
應用樹 (layout_node)
  container (strategy=Flex-Row)
  ├── widget_unit (banner_item, priv=...)
  ├── widget_unit (banner_item, priv=...)
  └── widget_unit (banner_item, priv=...)

渲染樹 (lv_obj_t)
  lv_obj (LV_FLEX_FLOW_ROW)
  ├── lv_img (banner_1.png)
  ├── lv_img (banner_2.png)
  └── lv_img (banner_3.png)
```

應用樹節點通過 `obj` 指標與渲染樹節點一一對應，但應用樹額外管理了：

| 管理維度   | 渲染樹 (lv_obj_t)                  | 應用樹 (layout_node)                 |
| ---------- | ---------------------------------- | ------------------------------------ |
| 生命週期   | `lv_obj_del()` 手動散落各處        | `destroy()` 虛函式，深度優先自動銷毀 |
| 子節點管理 | LVGL 內部鏈結串列，無業務語意      | `children[]` 動態陣列，容量倍增      |
| 佈局參數   | 分散在各 `lv_obj_set_style_*` 呼叫 | `layout_strategy_t` 統一封裝         |
| 資料繫結   | 手動逐欄位賦值                     | `bind()` / `update()` 虛函式         |
| 私有資料   | 無標準機制                         | `widget_unit.priv` 指向記憶體池物件  |

---

## 二、組合模式：layout_node 核心設計

### 2.1 三層型別體系

應用樹的核心是一個**組合模式 (Composite)** 的三層型別體系：

```c
/* 抽象基類 - 虛函式表 */
struct layout_node {
    lv_obj_t *obj;                          // 對應的渲染樹節點
    const char *type_name;                  // 型別名（"container"/"banner_item"/...）
    layout_node_type_t node_type;           // CONTAINER 或 WIDGET
    layout_container_t *parent;             // 父容器引用

    void (*destroy)(layout_node_t *self);   // 虛函式：銷毀
    void (*update)(layout_node_t *self,     // 虛函式：資料更新
                   const cJSON *data);
};

/* 容器節點 - 可含子節點 */
struct layout_container {
    layout_node_t base;                     // 繼承基類
    layout_strategy_t strategy;             // 佈局策略
    layout_node_t **children;               // 子節點動態陣列
    int child_count;
    int child_capacity;
    // 方法指標: add_child, remove_child, clear, get_child
};

/* 葉子節點 - 最小可渲染單元 */
struct widget_unit {
    layout_node_t base;                     // 繼承基類
    void *priv;                             // 私有資料（從記憶體池分配）
    void (*bind)(widget_unit_t *self,       // 資料繫結函式
                 const cJSON *data);
};
```

### 2.2 面向組合元件塊的顆粒度

應用樹節點的顆粒度是**業務級的組合元件塊**，極少對應 LVGL 的原生基礎物件（如單個 `lv_label`、`lv_btn`）。

#### 什麼是組合元件塊？

以售貨機的「商品卡片」為例，在原生 LVGL 中它由以下 5 個物件組成：

- 1 個 `lv_obj`（最外層 container）
- 1 個 `lv_img`（商品圖片）
- 2 個 `lv_label`（名稱、價格）
- 1 個 `lv_btn`（購買按鈕）

應用樹的做法：

```c
// 應用樹節點（組合元件）
widget_unit_t *card = widget_factory_create("commodity_card", parent);
card->base.obj = container;  // 指向組合元件最外層 container
card->priv = &card_data;     // 業務資料（商品ID、價格、庫存）
```

一個 `widget_unit_t` 節點，`obj` 指標指向最外層 container，元件工廠內部一口氣把這 5 個 `lv_obj_t` 都創建出來並組合好。

#### 塊級銷毀與安全回收

當頁面切換或列表滑動導致某個商品卡片移出螢幕時：

| 原生做法                                                        | 應用樹做法                                                                            |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| 手動找到 img, label1, label2, btn 分別 `lv_obj_del()`，極易漏刪 | 直接銷毀 `layout_node`，框架執行 `lv_obj_del(node->obj)`，LVGL 自動遞迴刪除所有子物件 |
| 業務程式碼可能還在引用已刪除物件，導致野指標崩潰                  | 應用樹清空 `node->obj`，業務層不會再誤觸底層 UI                                       |

#### 掛起/啟動機制：業務狀態保留與肉體重建

在記憶體受限的 MCU 上，長列表有 100 個商品時不能創建 100 個 `lv_obj` 塊（記憶體會爆）。應用樹配合非同步載入器實現：

- **保留 100 個 `layout_node`**（僅存幾個位元組的業務資料，記憶體佔用極小）
- **僅創建螢幕可見的 5 個 `lv_obj` 塊**

當用戶滑動螢幕時：

```
滑出螢幕的塊：
  應用樹執行 lv_obj_del() 銷毀 LVGL 肉體 → 釋放顯存
  但 layout_node 的業務資料（滑動偏移、選中狀態）保留

滑入螢幕的塊：
  應用樹根據已有 layout_node → 呼叫工廠重新 lv_obj_create 造出肉體
  利用保留的業務資料恢復介面狀態
```

這種「面向組合元件塊」的管理方式，使得幾百 KB 記憶體的 MCU 能流暢執行無限捲動列表、多頁面巢狀的複雜 GUI。

### 2.4 深度優先銷毀

`destroy()` 虛函式實現了**深度優先**的子樹銷毀，保證所有子節點先於父節點被回收：

```c
static void container_destroy(layout_node_t *self_base) {
    layout_container_t *self = (layout_container_t *)self_base;

    // 1. 先銷毀所有子節點（深度優先）
    for (int i = 0; i < self->child_count; i++)
        layout_node_destroy(self->children[i]);

    // 2. 釋放子節點陣列
    rt_free(self->children);

    // 3. 銷毀渲染樹物件
    lv_obj_del(self->base.obj);

    // 4. 釋放自身
    rt_free(self);
}
```

通用銷毀調度器通過虛函式表實現多型：

```c
void layout_node_destroy(layout_node_t *node) {
    if (node && node->destroy)
        node->destroy(node);
}
```

這意味著呼叫者無需關心節點的具體型別，一個 `layout_node_destroy()` 即可銷毀整棵子樹。

### 2.5 子節點動態陣列

容器使用動態陣列管理子節點，初始容量 8，倍增擴容：

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

`add_child` 時自動擴容並應用佈局策略中的子節點尺寸：

```c
void layout_container_add_child(layout_container_t *self, layout_node_t *child) {
    children_ensure_capacity(self, self->child_count + 1);
    child->parent = self;
    self->children[self->child_count++] = child;
    layout_strategy_apply_child_size(child->obj, &self->strategy);
}
```

`clear()` 清空所有子節點後，若容量超過初始值的 4 倍則縮容，避免長期執行後記憶體膨脹：

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

## 三、Wrap 橋接：從靜態骨架到動態容器

### 3.1 兩種創建方式

`layout_container` 提供兩種創建方式，這是整個架構最關鍵的設計：

| 方式       | 函式                        | 創建 LVGL 物件         | 銷毀 LVGL 物件                             | 用途                      |
| ---------- | --------------------------- | ---------------------- | ------------------------------------------ | ------------------------- |
| **Create** | `layout_container_create()` | 新建 `lv_obj_create()` | `container_destroy()` 中 `lv_obj_del()`    | 全新動態容器              |
| **Wrap**   | `layout_container_wrap()`   | 複用已有物件           | `container_destroy_wrapped()` 中**不**刪除 | 橋接 AiUIBuilder 靜態物件 |

### 3.2 Wrap 機制詳解

`layout_container_wrap()` 接收一個**已存在的 `lv_obj_t*`**，將其包裝為應用樹容器，但**不改變物件所有權**：

```c
layout_container_t *layout_container_wrap(lv_obj_t *existing_obj,
                                          const layout_strategy_t *strategy) {
    layout_container_t *self = rt_malloc(sizeof(layout_container_t));
    memset(self, 0, sizeof(*self));

    // 複用已有物件，而非創建新物件
    self->base.obj = existing_obj;

    // 在已有物件上應用佈局策略
    layout_strategy_apply(existing_obj, &self->strategy);

    // 使用特殊的銷毀函式 - 不刪除 LVGL 物件
    self->base.destroy = container_destroy_wrapped;

    return self;
}
```

`container_destroy_wrapped()` 的關鍵區別：

```c
static void container_destroy_wrapped(layout_node_t *self_base) {
    layout_container_t *self = (layout_container_t *)self_base;

    // 仍然銷毀所有子節點（深度優先）
    for (int i = 0; i < self->child_count; i++)
        layout_node_destroy(self->children[i]);
    rt_free(self->children);

    // 關鍵：不刪除 LVGL 物件，它由外部（screen_t）管理
    self->base.obj = NULL;

    rt_free(self);
}
```

### 3.3 實際橋接流程

在 `custom_init()` 中，橋接流程如下：

```
1. AiUIBuilder 生成:  scr->container_banner (lv_obj_t)
                       ├── scr->image_banner_1 (靜態子控件)
                       ├── scr->image_banner_2
                       └── scr->image_banner_3

2. 刪除靜態子控件:    lv_obj_del(scr->image_banner_1);
                       lv_obj_del(scr->image_banner_2);
                       lv_obj_del(scr->image_banner_3);

3. Wrap 容器:         banner_comp->container = layout_container_wrap(
                           scr->container_banner, &flex_row_strategy);

4. 動態填充子節點:    for each image_path:
                         item = widget_factory_create("banner_item", container->base.obj);
                         container->add_child(container, &item->base);
                         item->bind(item, &path_data);
```

這樣，AiUIBuilder 創建的 `container_banner` 仍然存在於渲染樹中（由 `screen_t` 管理生命週期），但它的**子控件已全部替換為應用樹管理的動態節點**，支援執行時增刪、資料繫結和熱更新。

---

## 四、策略模式：layout_strategy 佈局引擎

### 4.1 策略配置結構

`layout_strategy_t` 將 LVGL 的分散佈局 API 統一封裝為一個配置結構：

```c
typedef struct {
    layout_type_t type;       // FLEX_ROW / FLEX_COLUMN / GRID
    int gap_x, gap_y;        // 間距
    int pad_x, pad_y;        // 內邊距
    int cols;                 // Grid 列數
    int cell_w, cell_h;      // 子節點固定尺寸 (0=auto)
    bool scrollable;          // 是否可捲動
    lv_dir_t scroll_dir;      // 捲動方向
} layout_strategy_t;
```

### 4.2 策略應用

`layout_strategy_apply()` 將策略一次性映射到 LVGL Flex API：

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
        // Grid = row-wrap，子節點通過 FLEX_IN_NEW_TRACK 換行
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

### 4.3 子節點尺寸應用

策略中的 `cell_w` 和 `cell_h` 定義了子節點的固定尺寸。當子節點被添加到容器時，`layout_strategy_apply_child_size()` 自動應用這些尺寸：

```c
void layout_strategy_apply_child_size(lv_obj_t *child_obj,
                                       const layout_strategy_t *s) {
    if (s->cell_w > 0)
        lv_obj_set_width(child_obj, s->cell_w);
    if (s->cell_h > 0)
        lv_obj_set_height(child_obj, s->cell_h);
}
```

這個函式在 `layout_container_add_child()` 中被呼叫：

```c
void layout_container_add_child(layout_container_t *self, layout_node_t *child) {
    children_ensure_capacity(self, self->child_count + 1);
    child->parent = self;
    self->children[self->child_count++] = child;
    // 關鍵：添加子節點時自動應用策略尺寸
    layout_strategy_apply_child_size(child->obj, &self->strategy);
}
```

**設計意義**：佈局參數從分散的 API 呼叫（各元件檔案裡的 `lv_obj_set_size`）收攏到策略結構體，元件創建時只需關注「我是什麼型別」，尺寸由父容器的策略統一控制。

### 4.4 四大元件的佈局策略

在 `custom_init()` 中，四大元件各自使用不同的策略創建：

```c
// 橫幅輪播 - Flex-Row，水平捲動
s_banner_comp = banner_component_create(
    scr->container_banner,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_ROW, .cell_w = 494, .cell_h = 78,
        .scrollable = true, .scroll_dir = LV_DIR_HOR,
    }, ...);

// 商品展示 - Grid 3列，垂直捲動
s_commodity_comp = commodity_component_create(
    scr->container_main,
    &(layout_strategy_t){
        .type = LAYOUT_GRID, .cols = 3,
        .gap_x = 12, .gap_y = 12, .cell_w = 240, .cell_h = 100,
        .scrollable = true, .scroll_dir = LV_DIR_VER,
    }, ...);

// 購物車 - Flex-Column，垂直捲動
s_cart_comp = cart_component_create(
    scr->container_cart_list_scroll,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_COLUMN, .gap_y = 10,
        .cell_w = 750, .cell_h = 70,
        .scrollable = true, .scroll_dir = LV_DIR_VER,
    });

// 導航欄 - Flex-Row，不可捲動
s_navi_comp = navi_component_create(
    scr->container_navi_bg,
    &(layout_strategy_t){
        .type = LAYOUT_FLEX_ROW, .gap_x = 8,
        .cell_w = 120, .cell_h = 35,
        .scrollable = false,
    }, ...);
```

佈局參數從程式碼中分散的 `lv_obj_set_style_*` 呼叫，收攏為**一個結構體字面量**，一目了然。

---

## 五、工廠模式：widget_factory 動態創建

### 5.1 註冊表設計

`widget_factory` 實現了按型別名字串註冊創建函式的**註冊表模式**：

```c
typedef widget_unit_t *(*widget_create_fn)(lv_obj_t *parent);

static struct {
    const char *type_name;
    widget_create_fn create;
} s_entries[16];
static int s_entry_count = 0;
```

初始化階段註冊四種控件：

```c
void custom_init() {
    commodity_card_register();   // "commodity_card" -> commodity_card_create()
    banner_item_register();      // "banner_item"    -> banner_item_create()
    cart_item_register();        // "cart_item"      -> cart_item_create()
    navi_item_register();        // "navi_item"      -> navi_item_create()
}
```

### 5.2 執行時動態創建

元件工廠通過型別名動態創建控件，並自動加入應用樹：

```c
// component_factory.c - 創建 banner 元件
for (int i = 0; i < initial_data->count; i++) {
    widget_unit_t *item =
        widget_factory_create("banner_item", comp->container->base.obj);
    if (item) {
        comp->container->add_child(comp->container, &item->base);
        item->bind(item, &path_data);  // 資料繫結
    }
}
```

每種控件的 `create()` 函式負責：

1. 從 RT-Thread 記憶體池分配 `widget_unit_t` 和私有資料結構
2. 創建 LVGL 渲染物件並設定樣式
3. 設定 `bind()` 和 `destroy()` 虛函式
4. 回傳初始化完成的 `widget_unit_t*`

---

## 六、元件工廠：應用樹的組裝單元

### 6.1 元件 = 容器 + 子控件 + 控制器

`component_factory` 在應用樹之上封裝**完整的業務元件**，每個元件由三部分組成：

```c
typedef struct {
    layout_container_t *container;     // 應用樹容器
    banner_carousel_t *carousel;       // 控制器（不持有容器）
} banner_component_t;
```

以 banner 元件為例，`banner_component_create()` 的組裝流程：

```
1. layout_container_wrap(parent, strategy)  → 創建應用樹容器
2. widget_factory_create("banner_item", ...) × N  → 批次創建子控件
3. container->add_child(...) × N  → 加入應用樹
4. item->bind(...) × N  → 繫結資料
5. banner_carousel_create(container, ...)  → 掛載控制器
```

### 6.2 控制器-容器分離

`banner_carousel` 是一個**純粹的控制器**，它不創建也不擁有容器和子控件，只附加行為：

```c
struct banner_carousel {
    layout_container_t *container;    // 引用，非擁有
    lv_timer_t *auto_scroll_timer;   // 自動捲動計時器
    lv_obj_t **dots;                  // 指示器圓點
    int32_t last_scroll_x;           // 手勢檢測狀態
    bool is_animating;
};
```

這種分離使得：

- 容器可以獨立於控制器被銷毀和重建
- 控制器可以在執行時切換到不同的容器實例
- 資料熱更新時，控制器暫停 → 容器 clear → 重建子控件 → 控制器恢復

#### 6.2.1 自動輪播計時器

`banner_carousel` 創建一個 LVGL 計時器實現 5 秒間隔的自動捲動：

```c
carousel->auto_scroll_timer = lv_timer_create(
    auto_scroll_cb, 5000, carousel);

static void auto_scroll_cb(lv_timer_t *timer) {
    banner_carousel_t *carousel = timer->user_data;
    if (carousel->is_animating) return;  // 動畫中不觸發

    // 計算下一個索引（循環）
    int next_idx = (carousel->current_idx + 1) % carousel->container->child_count;

    // 捲動到目標位置
    lv_obj_scroll_to_x(carousel->container->base.obj,
                        next_idx * carousel->strategy.cell_w,
                        LV_ANIM_ON);

    carousel->current_idx = next_idx;
    update_dots_highlight(carousel);  // 同步指示器
}
```

#### 6.2.2 手勢檢測與用戶交互暫停

當用戶手指滑動橫幅時，自動輪播暫停 10 秒，避免干擾：

```c
static void gesture_detect_cb(lv_event_t *e) {
    banner_carousel_t *carousel = lv_event_get_user_data(e);
    int32_t scroll_x = lv_obj_get_scroll_x(carousel->container->base.obj);

    if (abs(scroll_x - carousel->last_scroll_x) > 10) {
        // 檢測到滑動：暫停自動捲動 10 秒
        lv_timer_pause(carousel->auto_scroll_timer);
        lv_timer_reset(carousel->resume_timer);  // 10 秒後恢復

        carousel->last_scroll_x = scroll_x;
    }
}
```

#### 6.2.3 指示器圓點同步

指示器是一組小圓點（`lv_obj`），數量與橫幅圖片一致，當前頁高亮：

```c
static void create_dots(banner_carousel_t *carousel, lv_obj_t *parent) {
    int count = carousel->container->child_count;
    carousel->dots = rt_malloc(sizeof(lv_obj_t *) * count);

    for (int i = 0; i < count; i++) {
        lv_obj_t *dot = lv_obj_create(parent);
        lv_obj_set_size(dot, 8, 8);
        lv_obj_set_style_bg_color(dot, lv_color_hex(0xCCCCCC), 0);
        // 第一個預設高亮
        if (i == 0)
            lv_obj_set_style_bg_color(dot, lv_color_hex(0xFFFFFF), 0);
        carousel->dots[i] = dot;
    }
}

static void update_dots_highlight(banner_carousel_t *carousel) {
    for (int i = 0; i < carousel->container->child_count; i++) {
        lv_obj_set_style_bg_color(
            carousel->dots[i],
            i == carousel->current_idx ? lv_color_hex(0xFFFFFF) : lv_color_hex(0xCCCCCC),
            0);
    }
}
```

### 6.3 資料熱更新

`banner_carousel_update_data()` 展示了應用樹的熱更新流程：

```c
void banner_carousel_update_data(banner_carousel_t *carousel,
                                  const carousel_data_t *data) {
    // 1. 暫停自動捲動
    lv_timer_pause(carousel->auto_scroll_timer);

    // 2. 銷毀舊指示器
    destroy_dots(carousel);

    // 3. 清空容器子節點（深度優先銷毀所有舊 banner_item）
    carousel->container->clear(carousel->container);

    // 4. 從新資料創建子控件
    for (int i = 0; i < data->count; i++) {
        widget_unit_t *item = widget_factory_create("banner_item", ...);
        carousel->container->add_child(carousel->container, &item->base);
        item->bind(item, &path_data);
    }

    // 5. 重建指示器（延遲 50ms 等 LVGL 佈局完成）
    schedule_dots_init(carousel);

    // 6. 恢復自動捲動
    lv_timer_resume(carousel->auto_scroll_timer);
}
```

整個流程對呼叫者透明——只需傳入新資料，應用樹自動完成**銷毀舊節點 → 創建新節點 → 資料繫結 → 佈局重排**。

---

## 七、事件驅動：跨元件通信與配置熱更新

### 7.1 事件匯流排

`event_bus` 實現發布-訂閱解耦，定義了 8 種事件型別：

```c
typedef enum {
    EVT_CART_ITEM_ADDED,       // 商品加購
    EVT_CART_ITEM_REMOVED,     // 購物車移除
    EVT_CART_COUNT_CHANGED,    // 數量變化
    EVT_CART_PRICE_CHANGED,    // 價格變化
    EVT_BANNER_CONFIG_CHANGED, // Banner 配置變更
    EVT_COMMODITY_CONFIG_CHANGED, // 商品配置變更
    EVT_NAVI_CONFIG_CHANGED,   // 導航配置變更
    EVT_NAVI_ITEM_CLICKED,     // 導航點擊
} event_type_t;
```

### 7.2 配置熱更新的完整鏈路

Web 配置觸發的事件鏈路，展示了從 HTTP 請求到 UI 重渲染的完整流程：

```
HTTP POST /api/config
  ├── HTTP 執行緒: 解析 JSON → 寫入 s_pending_json → 設定 s_pending_update=1
  ├── HTTP 執行緒: 儲存 JSON 到 /sdcard/config/banner.json（持久化）
  └── LVGL 計時器 (200ms): 偵測 s_pending_update
      ├── 發布 EVT_BANNER_CONFIG_CHANGED (data=json_str)
      └── on_banner_config_changed() 訂閱者:
          ├── carousel_data_from_json(json_str, &data)
          └── banner_component_update(comp, &data)
              └── banner_carousel_update_data()
                  ├── 暫停計時器
                  ├── container->clear()     ← 銷毀舊應用樹子節點
                  ├── widget_factory_create() ← 創建新應用樹子節點
                  ├── container->add_child()  ← 加入應用樹
                  ├── item->bind()            ← 資料繫結
                  ├── schedule_dots_init()    ← 重建指示器
                  └── 恢復計時器
```

**關鍵**：HTTP 執行緒絕不直接呼叫 LVGL API，僅設定 volatile 標誌；所有 UI 操作在 LVGL 計時器的 LVGL 執行緒上下文中執行，保證執行緒安全。

### 7.3 跨元件聯動

商品卡片加購不直接呼叫購物車 API，而是通過事件匯流排：

```
commodity_card 點擊加購
    → event_bus_publish(EVT_CART_ITEM_ADDED, {name, price})
    → cart_module 訂閱處理
        → cart_add_commodity()
        → event_bus_publish(EVT_CART_PRICE_CHANGED, {total_price})
        → qr_scanner 訂閱處理
            → qr_scanner_set_payment_info()  // 支付金額始終與購物車同步
```

三個模組零直接依賴，完全通過事件匯流排通信。

### 7.4 RT-Thread 執行緒安全：引用與隔離

LVGL 有一個鐵律：**所有對 `lv_obj_t` 的操作必須在同一個執行緒（LVGL 執行緒）中進行**。如果業務執行緒（網路、感測器）直接呼叫 `lv_label_set_text()`，系統大概率崩潰。

#### 7.4.1 執行緒物理隔離

```
LVGL 渲染執行緒                業務工作執行緒
lv_timer_handler()           網路通信/資料處理
lv_tick_inc()                硬體讀寫
只負責渲染                    不觸碰 lv_obj_t
```

業務執行緒絕對拿不到 `lv_obj_t` 指標，只能操作純業務資料（C 結構體）。

#### 7.4.2 RT-Thread 訊息佇列橋樑

業務執行緒透過 RT-Thread 訊息佇列通知 UI 執行緒刷新：

```c
// 定義跨執行緒訊息
typedef struct {
    event_type_t type;
    layout_node_t *target_node;  // 引用應用樹節點（不碰 lv_obj）
    void *data;
} app_msg_t;

// RT-Thread 訊息佇列初始化
struct rt_messagequeue ui_mq;
rt_mq_init(&ui_mq, "ui_mq",
           rt_malloc(sizeof(app_msg_t) * 16),
           sizeof(app_msg_t), 16, RT_IPC_FLAG_FIFO);

// 業務執行緒發送更新請求（隔離且安全）
void network_thread_handle_new_price(uint8_t price) {
    app_msg_t msg = {
        .type = EVT_CART_PRICE_CHANGED,
        .target_node = g_cart_node,  // 引用節點，不碰 lv_obj
        .data = &price
    };
    rt_mq_send(&ui_mq, &msg, sizeof(app_msg_t));
    // 業務執行緒工作完成，絕不阻塞 UI 執行緒
}

// LVGL 執行緒接收並驅動渲染
void lvgl_thread_entry(void *param) {
    app_msg_t recv_msg;
    while (1) {
        if (rt_mq_recv(&ui_mq, &recv_msg, sizeof(app_msg_t), RT_WAITING_FOREVER) == RT_EOK) {
            // 在 LVGL 執行緒安全地更新 UI
            switch (recv_msg.type) {
                case EVT_CART_PRICE_CHANGED:
                    cart_node_set_price(recv_msg.target_node, *(uint8_t*)recv_msg.data);
                    break;
            }
        }
        lv_timer_handler();  // 渲染
    }
}
```

#### 7.4.3 引用與隔離的精髓

| 機制         | 作用                                                             |
| ------------ | ---------------------------------------------------------------- |
| **執行緒隔離** | 業務跑獨立執行緒，看不到 `lv_obj_t`；UI 跑獨立執行緒，不關心業務協議 |
| **訊息佇列** | 業務執行緒引用應用樹節點指標，把資料丟進佇列後轉身就走             |
| **安全渲染** | UI 執行緒拿到指標後，在 LVGL 執行緒上下文安全操作底層 `lv_obj`       |

在這套架構下，即使網路執行緒瘋狂發訊息，LVGL 渲染執行緒仍能以自己節奏從佇列取訊息並平滑刷新，絕不死機或卡頓。

---

## 八、從靜態到動態：custom_init() 的替換流程

`custom_init()` 是系統組裝入口，完整展示了從 AiUIBuilder 靜態骨架到動態應用樹的替換過程：

### 8.1 SD卡掛載與資源路徑管理

動態元件的圖片資源存放在 SD 卡，啟動時必須先掛載：

```c
static int ensure_sdcard_mounted(void) {
    if (dfs_mount("sd0", "/sdcard", "elm", 0, 0) == 0)
        return 0;  // 已掛載

    // 重試最多 5 次
    for (int i = 0; i < 5; i++) {
        rt_thread_mdelay(100);
        if (dfs_mount("sd0", "/sdcard", "elm", 0, 0) == 0)
            return 0;
    }

    rt_kprintf("SD card mount failed, use default paths\n");
    return -1;
}
```

掛載成功後，資源路徑標準化：

| 資源型別    | SD卡路徑                     | 用途          |
| ----------- | ---------------------------- | ------------- |
| Banner 圖片 | `/sdcard/banner/xxx.png`     | 輪播橫幅      |
| 商品圖片    | `/sdcard/commodity/xxx.png`  | 商品卡片      |
| 配置檔案    | `/sdcard/config/banner.json` | Web配置持久化 |

所有路徑在 `web_config.c` 中統一管理，避免硬編碼散落各處。

### 8.2 完整初始化流程

```
Step 1: 初始化基礎設施
  event_bus_init() + async_loader_init()

Step 2: 掛載 SD 卡（資源路徑準備）
  ensure_sdcard_mounted()

Step 3: 初始化物件池
  banner_item_module_init()    // rt_mp_create, 容量=10
  commodity_module_init()      // rt_mp_create, 容量=30
  cart_module_init()           // rt_mp_create, 容量=20
  navi_item_module_init()      // rt_mp_create, 容量=10

Step 3: 註冊控件工廠
  commodity_card_register()    // "commodity_card" → create_fn
  banner_item_register()       // "banner_item"    → create_fn
  cart_item_register()         // "cart_item"      → create_fn
  navi_item_register()         // "navi_item"      → create_fn

Step 4: 獲取 AiUIBuilder 生成的螢幕物件
  screen_t *scr = screen_get(&ui_manager);

Step 5: 刪除靜態子控件，替換為動態元件
  // Banner: 刪除 3 個靜態圖片
  lv_obj_del(scr->image_banner_1);
  lv_obj_del(scr->image_banner_2);
  lv_obj_del(scr->image_banner_3);
  // Wrap 容器 + 動態填充
  s_banner_comp = banner_component_create(scr->container_banner, ...);

  // Commodity: 刪除 6 個靜態容器
  lv_obj_del(scr->container_main_1~6);
  // Wrap 容器 + 動態填充
  s_commodity_comp = commodity_component_create(scr->container_main, ...);

  // Cart: 刪除 8 個靜態控件
  lv_obj_del(scr->image_cart_item_bg_1~3);
  lv_obj_del(scr->button_cart_item_add_1);
  // ...
  // Wrap 容器（初始為空，事件驅動添加）
  s_cart_comp = cart_component_create(scr->container_cart_list_scroll, ...);

  // Navi: 刪除 2 個靜態容器
  lv_obj_del(scr->container_navi_1~2);
  // Wrap 容器 + 動態填充
  s_navi_comp = navi_component_create(scr->container_navi_bg, ...);

Step 6: 事件訂閱 + 啟動業務模組
  event_bus_subscribe(EVT_BANNER_CONFIG_CHANGED, ...);
  event_bus_subscribe(EVT_COMMODITY_CONFIG_CHANGED, ...);
  web_config_init();
  uart3_payment_init();
  qr_scanner_init();
```

**核心思路**：保留 AiUIBuilder 創建的**容器物件**（如 `container_banner`），刪除其**靜態子控件**，然後通過 Wrap 機制將容器納入應用樹管理，再通過控件工廠動態填充可資料驅動的子節點。

### 8.3 購物車開合動畫

購物車列表預設隱藏，點擊結算按鈕時展開，透過 LVGL 動畫 API 實現：

```c
// 展開動畫：高度從 0 → 400，透明度從 0 → 255
static void cart_list_open_anim(lv_obj_t *cart_container) {
    lv_anim_t a;
    lv_anim_init(&a);
    lv_anim_set_var(&a, cart_container);
    lv_anim_set_exec_cb(&a, lv_obj_set_height);
    lv_anim_set_values(&a, 0, 400);
    lv_anim_set_time(&a, 300);
    lv_anim_set_path_cb(&a, lv_anim_path_ease_out);
    lv_anim_start(&a);

    // 同步透明度動畫
    lv_anim_set_exec_cb(&a, lv_obj_set_style_opa);
    lv_anim_set_values(&a, LV_OPA_TRANSP, LV_OPA_COVER);
    lv_anim_start(&a);
}

// 收起動畫：反向過程
static void cart_list_close_anim(lv_obj_t *cart_container) {
    lv_anim_t a;
    lv_anim_init(&a);
    lv_anim_set_var(&a, cart_container);
    lv_anim_set_exec_cb(&a, lv_obj_set_height);
    lv_anim_set_values(&a, 400, 0);
    lv_anim_set_time(&a, 300);
    lv_anim_set_path_cb(&a, lv_anim_path_ease_in);
    lv_anim_start(&a);

    lv_anim_set_exec_cb(&a, lv_obj_set_style_opa);
    lv_anim_set_values(&a, LV_OPA_COVER, LV_OPA_TRANSP);
    lv_anim_start(&a);
}
```

動畫觸發由結算按鈕點擊事件驅動，配合應用樹的購物車容器：

```c
static void settlement_btn_click_cb(lv_event_t *e) {
    if (g_cart_list_visible) {
        cart_list_close_anim(s_cart_comp->container->base.obj);
        g_cart_list_visible = false;
    } else {
        cart_list_open_anim(s_cart_comp->container->base.obj);
        g_cart_list_visible = true;
    }
}
```

**設計要點**：動畫操作的是應用樹容器對應的 `lv_obj_t`，但控制邏輯在業務層（`g_cart_list_visible` 標誌），體現了資料與渲染的分離。

---

## 九、記憶體管理策略

### 9.1 物件池 vs 堆積分配

| 物件                       | 分配方式      | 原因                              |
| -------------------------- | ------------- | --------------------------------- |
| `layout_container_t`       | `rt_malloc`   | 生命週期長，數量少（4個元件容器） |
| `layout_node_t **children` | `rt_malloc`   | 需倍增擴容，不適用固定大小池      |
| `widget_unit_t` + `priv`   | `rt_mp_alloc` | 高頻創建/銷毀，需避免碎片         |
| `navi_click_ctx_t`         | `rt_malloc`   | 與事件回呼生命週期繫結            |

### 9.2 物件池容量對齊

每個控件模組獨立管理記憶體池，容量與業務上限對齊：

- `banner_item`: 容量 10（最多 10 張橫幅）
- `commodity_card`: 容量 30（最多 30 件商品）
- `cart_item`: 容量 20（最多 20 個購物車項）
- `navi_item`: 容量 10（最多 10 個導航標籤）

---

## 十、架構總結

### 10.1 設計模式協同

```
component_factory (組裝層: 容器 + 子控件 + 控制器)
widget_factory (工廠模式) | layout_strategy (策略模式) | event_bus (Pub-Sub)
layout_node (組合模式)
  container ─┬─ widget_unit (葉子)
             ├─ widget_unit
             └─ container ─┬─ widget_unit
                           └─ ...
LVGL 渲染樹 (lv_obj_t)
```

### 10.2 核心價值

1. **Wrap 橋接**：`layout_container_wrap()` 是連接靜態骨架與動態應用樹的橋樑，保留視覺化工具的快速原型能力，同時獲得執行時靈活性。

2. **深度優先銷毀**：`destroy()` 虛函式遞迴銷毀子樹，消除了手動 `lv_obj_del()` 散落各處的生命週期管理問題。

3. **資料驅動重建**：`clear()` → `widget_factory_create()` → `add_child()` → `bind()` 的標準化流程，使熱更新邏輯高度統一。

4. **策略封裝**：佈局參數從分散的 API 呼叫收攏為 `layout_strategy_t` 結構體，元件創建時一目了然。

5. **事件解耦**：元件間零直接依賴，購物車不知道商品卡片，支付模組不知道購物車，Web 配置不知道任何 UI API。

---

## 十一、串口支付流程

### 11.1 幀協議

設備與 PC 主機透過 UART3 通信，採用自訂幀協議：

```
AA 55 CMD LEN_H LEN_L DATA... 0D 0A
```

| 方向      | CMD  | 名稱              | 資料                           |
| --------- | ---- | ----------------- | ------------------------------ |
| PC → 設備 | 0x01 | CMD_QR_URL        | 二維碼 URL (UTF-8)             |
| PC → 設備 | 0x02 | CMD_PAY_STATUS    | 支付狀態碼 (1位元組)           |
| PC → 設備 | 0x03 | CMD_ORDER_NO      | 訂單號 (UTF-8)                 |
| PC → 設備 | 0x04 | CMD_HEARTBEAT_ACK | 心跳應答                       |
| 設備 → PC | 0x81 | CMD_REQ_QR        | name\0amount                   |
| 設備 → PC | 0x82 | CMD_REQ_QUERY     | 空                             |
| 設備 → PC | 0x83 | CMD_REQ_CLOSE     | 空                             |
| 設備 → PC | 0x84 | CMD_HEARTBEAT     | 空                             |
| 設備 → PC | 0x85 | CMD_BARCODE_PAY   | auth_code\0[amount\0][subject] |

### 11.2 雙支付模式

**模式一：用戶掃碼**

1. 用戶點擊結算按鈕 → 解析購物車總價 → 顯示支付模式選擇彈窗
2. 用戶選擇「用戶掃碼」→ 發送 `CMD_REQ_QR` 到 PC
3. PC 生成支付寶沙箱訂單，回傳 `CMD_QR_URL`
4. 設備顯示 QR 碼覆蓋層（240×240，LVGL `lv_qrcode` 控件）
5. PC 輪詢支付狀態後回傳 `CMD_PAY_STATUS`
6. 成功：顯示「支付成功!」，2 秒後自動隱藏，清空購物車，恢復輪播

**模式二：設備掃碼（條碼支付）**

1. 用戶選擇「設備攝像頭掃碼」
2. 啟動 DVP 攝像頭 + quirc 掃描執行緒
3. 設定 UI 層 alpha=128 使視頻層透視顯示
4. quirc 掃描到 18~28 位純數字（支付授權碼）後，透過 `CMD_BARCODE_PAY` 發送至 PC
5. PC 呼叫支付寶條碼支付 API，回傳支付狀態
6. 成功後流程同上

### 11.3 執行緒安全設計

UART3 RX 執行緒嚴格不呼叫任何 LVGL API。幀解析完成後僅入隊到 SPSC 環形緩衝區。LVGL 計時器 (100ms) 在 LVGL 執行緒上下文出隊處理，確保所有 UI 操作的執行緒安全。

---

## 十二、二維碼掃描模組

### 12.1 硬體配置

- DVP 攝像頭 (OV5640)，強制 QVGA 320×240 輸出，降低記憶體佔用（約 460KB）
- ArtInChip 視頻輸入框架 (`mpp_vin`, `drv_dvp`)
- Framebuffer 視頻層 (`mpp_fb`) 用於攝像頭預覽
- quirc QR 碼解碼庫

### 12.2 掃描流程

1. `qr_scanner_start()`：初始化 DVP（含 5 次重試容錯）、配置感測器格式、創建 quirc 實例 (160×120，2 倍下取樣)
2. 掃描執行緒迴圈：`dq_buf` → 丟棄前幾幀 → 每 5 幀提取 Y 通道並下取樣 → `quirc_decode`
3. `handle_qr_result()`：偵測是否為支付授權碼（18~28 位純數字），同碼 5 秒冷卻，呼叫 `uart3_payment_send_barcode()` 發送

### 12.3 攝像頭視圖模式

透過 ArtInChip framebuffer 的視頻層直接顯示 DVP 擷取幀：

- `qr_scanner_start_camera_view()`：開啟 framebuffer，設定 UI 層 alpha=128
- DVP 緩衝區零拷貝推送到視頻層，實現高效能畫中畫效果
- 停止時嚴格順序：清標誌 → 停掃描執行緒 → 停 DVP 流 → 等執行緒退出 → 禁用視頻層 + 恢復 UI alpha → 釋放資源

---

## 十三、Web 配置系統

![Web 配置系統](web_config.png)

> [!NOTE]
> 想要查看原始碼？[web_config.html](https://gitee.com/ling-sir007/luban-lite/blob/master/packages/artinchip/lvgl-ui/aic_demo/vending_demo/config/web_config.html)

### 13.1 架構

設備啟動 WiFi AP（SSID: `AIC-Banner-Config`，密碼: `12345678`，信道 6），IP 位址 192.168.1.1，執行 DHCP 伺服器，在 80 埠啟動 Mongoose HTTP 伺服器。

### 13.2 API 介面

| 方法 | 路徑                               | 功能                            |
| ---- | ---------------------------------- | ------------------------------- |
| GET  | `/`                                | 嵌入式 HTML 配置頁面            |
| GET  | `/api/config`                      | 取得 Banner 配置 JSON           |
| GET  | `/api/commodity`                   | 取得商品配置 JSON               |
| GET  | `/api/navi`                        | 取得導航配置 JSON               |
| POST | `/api/config`                      | 更新 Banner 配置                |
| POST | `/api/commodity`                   | 更新商品配置                    |
| POST | `/api/navi`                        | 更新導航配置                    |
| POST | `/api/reset`                       | 重設為預設配置                  |
| POST | `/api/upload/{filename}`           | 上傳 Banner 圖片（校驗 494×78） |
| POST | `/api/upload/commodity/{filename}` | 上傳商品圖片（校驗 80×80）      |

### 13.3 安全措施

- 檔案名白名單校驗：僅允許字母、數字、`.`、`_`、`-`，長度不超過 64
- Content-Length 早期拒絕：超限直接回傳 413
- 接收緩衝區限制：`recv_mbuf.len > 307200 + 4096` 時強制關閉連線
- 圖片尺寸校驗：解析 PNG/JPEG/BMP 檔案頭驗證寬高，不匹配則刪除並回傳錯誤
- 價格格式校驗：支援多種貨幣前前綴，驗證必須包含數字和可選小數部分

### 13.4 配置熱更新

1. HTTP 執行緒接收 POST → 解析 JSON → 寫入 `s_pending_json` 緩衝區 → 設定標誌
2. 同時將 JSON 儲存到 SD 卡檔案實現持久化
3. LVGL 計時器 200ms 輪詢偵測標誌 → 發布事件 → 各元件執行熱更新

### 13.5 商品與導航分類系統

商品卡片的 `tag` 和 `navi_tag` 是兩個獨立維度，分別服務於不同目的：

| 欄位       | 用途                           | 取值範圍                     | 來源             |
| ---------- | ------------------------------ | ---------------------------- | ---------------- |
| `tag`      | 商品卡片左上角**顯示圖示**     | `hot`/`new`/`recommend`/`none` | Web 配置手動選擇 |
| `navi_tag` | 導航欄**分類篩選**歸屬         | 各手動建立的 navi 項 tag     | Web 配置分類下拉 |

**導航項管理規則**：

1. 預設存在「全部」項（tag=`all`），不可刪除、不可編輯
2. 使用者手動新增導航項（最多 10 個），tag 由 label 自動產生（如 label="飲料" → tag="飲料"）
3. 導航項變更（增/刪/改）時，立即重新整理商品卡片的分類下拉選項

**商品卡片分類歸屬**：

- 每個商品卡片在 Web 配置頁面中顯示「分類」下拉框，選項為目前所有 navi 項（排除「全部」），外加「無」選項
- 選擇後寫入 `navi_tag` 欄位，用於執行時篩選
- 點擊導航欄某項時，`commodity_apply_filter()` 按 `navi_tag` 過濾商品列表

```c
/* commodity_card_priv_t 中的關鍵欄位 */
typedef struct {
    // ...
    commodity_tag_t tag;       /* 顯示圖示：熱賣/新品/推薦/無 */
    char navi_tag[32];         /* 導航分類標籤，用於篩選 */
} commodity_card_priv_t;
```

**執行時篩選邏輯**（`commodity_apply_filter`）：

```c
/* 按 navi_tag 過濾商品 */
cJSON *filtered = cJSON_CreateArray();
int count = cJSON_GetArraySize(items);
for (int i = 0; i < count; i++) {
    cJSON *item = cJSON_GetArrayItem(items, i);
    cJSON *navi_tag = cJSON_GetObjectItem(item, "navi_tag");
    if (cJSON_IsString(navi_tag) &&
        strcmp(navi_tag->valuestring, filter_tag) == 0) {
        cJSON_AddItemToArray(filtered, cJSON_Duplicate(item, 1));
    }
}
```

**資料繫結安全性**：當商品卡片欄位為空時，必須清除舊的顯示值，避免殘影：

```c
/* 圖片為空時清空舊值 */
cJSON *image = cJSON_GetObjectItem(data, "image");
if (cJSON_IsString(image) && image->valuestring[0] != '\0') {
    lv_img_set_src(p->img, image->valuestring);
    strncpy(p->image, image->valuestring, sizeof(p->image) - 1);
} else {
    lv_img_set_src(p->img, NULL);   /* 清除殘影 */
    p->image[0] = '\0';
}
```

### 13.6 原始碼目錄組織

原始碼從扁平的 `my_custom_init/` 重構為按功能分類的 `vending_demo/` 目錄樹：

```
vending_demo/
├── core/          # event_bus.c/h, async_loader.c/h
├── layout/        # layout_node.c/h, layout_strategy.c/h, widget_factory.c/h
├── banner/        # banner_item.c/h, banner_carousel.c/h, banner_types.h
├── commodity/     # commodity_card.c/h
├── cart/          # cart_item.c/h
├── navi/          # navi_item.c/h
├── config/        # web_config.c/h, web_config.html → web_config_html.c
└── payment/       # uart3_payment.c/h, qr_scanner.c/h
```

SConscript 使用 `os.walk()` 遞迴掃描 `vending_demo/` 及其子目錄，收集 `.c` 檔案和標頭檔路徑：

```python
vending_path = cwd + '/vending_demo'
if os.path.exists(vending_path):
    for root, dirs, files in os.walk(vending_path):
        rela_path = root.replace(cwd + '/', '')
        src = src + Glob(rela_path + '/*.c')
        if check_h_hpp_exist(root):
            inc = inc + [root]
    html_src = os.path.join(cwd, 'vending_demo', 'config', 'web_config.html')
    html_c = os.path.join(cwd, 'vending_demo', 'config', 'web_config_html.c')
```

`web_config_html.c` 由 `web_config.html` 在建構時自動產生，包含嵌入式 HTML 字串常數，供 Mongoose HTTP 伺服器直接回傳，無需檔案系統讀取。

---

## 十四、非同步載入機制

基於 RT-Thread 執行緒 + 雙訊息佇列實現：

```
主執行緒 (LVGL)              工作執行緒
    |                                                 |
    | submit(task)                         |
    | ----[task_mq]---->              |
    |                                                 | work_fn(arg) → result
    |                                [result_mq]----> done_cb(result)
    |                                                 |
    | poll() <-- 排空 result_mq   |
```

- 工作執行緒棧：4096 位元組，優先級 20
- 任務/結果佇列容量：8 條
- `async_loader_poll()` 在 LVGL 計時器中呼叫，保證 `done_cb` 在 LVGL 執行緒安全執行

---

## 十五、技術亮點總結

1. **Wrap 橋接**：`layout_container_wrap()` 是連接靜態骨架與動態應用樹的橋樑，保留視覺化工具的快速原型能力，同時獲得執行時靈活性。

2. **深度優先銷毀**：`destroy()` 虛函式遞迴銷毀子樹，消除了手動 `lv_obj_del()` 散落各處的生命週期管理問題。

3. **資料驅動重建**：`clear()` → `widget_factory_create()` → `add_child()` → `bind()` 的標準化流程，使熱更新邏輯高度統一。

4. **策略封裝**：佈局參數從分散的 API 呼叫收攏為 `layout_strategy_t` 結構體，元件創建時一目瞭然。

5. **事件解耦**：元件間零直接依賴，購物車不知道商品卡片，支付模組不知道購物車，Web 配置不知道任何 UI API。

6. **嵌入式 Web 配置**：RT-Thread 上執行 Mongoose HTTP 伺服器 + WiFi AP，手機連線熱點即可修改配置，無需韌體升級，配置自動持久化到 SD 卡。

7. **雙支付模式**：支援「用戶掃碼」和「設備掃碼」兩種模式，透過 UART3 與 PC 主機通信，實現支付寶沙箱環境下的完整支付閉環。

8. **無鎖執行緒間通信**：UART3 使用 SPSC 環形佇列，Web 配置使用 volatile 標誌 + 緩衝區，均避免重量級鎖開銷。

9. **記憶體池管理**：所有高頻創建/銷毀的控件使用 RT-Thread 記憶體池，容量與業務上限對齊，避免記憶體碎片。

10. **攝像頭視圖硬體加速**：DVP 緩衝區零拷貝推送到視頻層，UI 層半透明實現畫中畫效果。

11. **tag/navi_tag 雙維度分類**：tag 控制商品卡片顯示圖示，navi_tag 控制導航欄篩選歸屬，兩個維度獨立管理、互不干擾，分類變更即時重新整理商品選項。

12. **遞迴目錄建構**：SConscript 透過 `os.walk()` 遞迴掃描 `vending_demo/` 子目錄，支援按功能歸檔的目錄結構，`web_config_html.c` 建構時自動產生。
