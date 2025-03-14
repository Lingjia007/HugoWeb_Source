+++
author = "Hugo Authors"
title = "Markdown語法示例"
date = "2025-03-11"
description = "示範文章展示了HTML元素的基本Markdown語法和格式。"
tags = [
    "markdown",
    "css",
    "html",
    "主題",
]
categories = [
    "主題",
    "語法",
]
series = ["Themes Guide"]
aliases = ["migrate-from-jekyl"]
image = "pawel-czerwinski-8uZPynIu-rQ-unsplash.jpg"
+++

本文提供了一個可以在 Hugo 內容檔案中使用的基本 Markdown 語法示例，還展示了基本 HTML 元素是否在 Hugo 主題中用 CSS 裝飾。

<!--more-->

## 標題

下面的 HTML `<h1>`—`<h6>` 元素表示六個級別的節標題 `<h1>` 是最高的節級別，而 `<h6>` 是最低的節級別。

# H1

## H2

### H3

#### H4

##### H5

###### H6

## 段落

一個傷害我的靈魂。當水來的時候，它是冷的還是冷的，或者是冷的或冷的，因為你想要受苦的娃娃因為或多或少的牛奶而丟失了，你想用身體上沉重的負擔來安慰自己。你認為下一件事是你的嗎？當下一代想要血液或一切時，他們會發生什麼？安靜。因為我在別人眼前都是這樣做的，他們將被迫這樣做，以致於他們無法做到，如果我打敗他們，他們就會逃跑。我討厭我掀起波瀾，讓你遭受版本的痛苦，但我希望你遭受希望的痛苦。

義大利？廚房裡有很多東西或對他們的仇恨，所以我不知道他們喜歡吃什麼，或者他們喜歡吃些什麼。

## 塊引用

塊引用元素表示從另一個來源引用的內容，可以選擇引用必須在「footer」或「cite」元素內，也可以選擇內聯更改，如註釋和縮寫。

#### 無歸屬塊引用

> Tiam，ad mint，epu 和 noise 依次排列。
> **注意**您可以在塊引用中使用\_Markdown 語法。

#### 歸屬塊引用

> 不要通過分享記憶來交流，要通過交流來分享記憶。<br>
> — <cite>羅伯·皮克[^1]</cite>

[^1]: The above quote is excerpted from Rob Pike's [talk](https://www.youtube.com/watch?v=PAAkCSZUG1c) during Gopherfest, November 18, 2015.

## 表格

表格不是核心 Markdown 規範的一部分，但 Hugo 支持開箱即用。

| 姓名  | 年齡 |
| ----- | ---- |
| Bob   | 27   |
| Alice | 23   |

#### 表中的內聯 Markdown

| Italics   | Bold     | Code   |
| --------- | -------- | ------ |
| _italics_ | **bold** | `code` |

| A                                    | B                                                    | C                                                                                                                              | D                              | E                                | F                                |
| ------------------------------------ | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------ | -------------------------------- | -------------------------------- |
| 當他痛苦的時候，他被認為是有幫助的。 | 愚者比愚者更聰明，愚者比愚昧者更令人憎恨，無人知曉。 | 當他當選時，他對自己說：「我不會去塵土，我也不會去塵土。」我們生活在一種擔憂的狀態中，但在一種塵土的狀態下，害怕買到乾淨的價格 | 他想去那裡，因為他不指揮汽車。 | 喝一件你需要穿的衣服，而不是穿。 | 但你錯了，你錯了。生活值得學習。 |

## 程式碼塊

#### 帶有反引號的程式碼塊

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Example HTML5 Document</title>
  </head>
  <body>
    <p>Test</p>
  </body>
</html>
```

#### 四個空格縮進的程式碼塊

    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <title>Example HTML5 Document</title>
    </head>
    <body>
      <p>Test</p>
    </body>
    </html>

#### 帶有 Hugo 內部高亮短代碼的程式碼塊

{{< highlight html >}}

<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Example HTML5 Document</title>
</head>
<body>
  <p>Test</p>
</body>
</html>
{{< /highlight >}}

#### 差異程式碼塊

```diff
[dependencies.bevy]
git = "https://github.com/bevyengine/bevy"
rev = "11f52b8c72fc3a568e8bb4a4cd1f3eb025ac2e13"
- features = ["dynamic"]
+ features = ["jpeg", "dynamic"]
```

## 清單類型

#### 有序清單

1. 第一項
2. 第二項
3. 第三項

#### 無序清單

- 清單項目
- 另一個項目
- 再另一個項目

#### 嵌套清單

- 水果

  - 蘋果
  - 橙子
  - 香蕉

- 乳製品
  - 牛奶
  - 起司

## 其他元素 —— 縮寫、下標、上標、鍵盤輸入、標記

<abbr title="Graphics Interchange Format">GIF</abbr> is a bitmap image format.

H<sub>2</sub>O

X<sup>n</sup> + Y<sup>n</sup> = Z<sup>n</sup>

Press <kbd>CTRL</kbd> + <kbd>ALT</kbd> + <kbd>Delete</kbd> to end the session.

Most <mark>salamanders</mark> are nocturnal, and hunt for insects, worms, and other small creatures.

## 帶超鏈接的圖片

請切換黑色主題
[![Google](https://www.google.com/images/branding/googlelogo/1x/googlelogo_light_color_272x92dp.png)](https://google.com)
