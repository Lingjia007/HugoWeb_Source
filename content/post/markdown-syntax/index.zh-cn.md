+++
author = "Hugo Authors"
title = "Markdown语法示例"
date = "2025-03-11"
description = "示例文章展示了HTML元素的基本Markdown语法和格式."
tags = [
    "markdown",
    "css",
    "html",
    "主题",
]
categories = [
    "主题",
    "语法",
]
series = ["Themes Guide"]
aliases = ["migrate-from-jekyl"]
image = "pawel-czerwinski-8uZPynIu-rQ-unsplash.jpg"
+++

本文提供了一个可以在 Hugo 内容文件中使用的基本 Markdown 语法示例，还展示了基本 HTML 元素是否在 Hugo 主题中用 CSS 装饰。

<!--more-->

## 标题

下面的 HTML `<h1>`—`<h6>` 元素表示六个级别的节标题 `<h1>` 是最高的节级别，而 `<h6>` 是最低的节级别。

# H1

## H2

### H3

#### H4

##### H5

###### H6

## 段落

一个伤害我的灵魂。当水来的时候，它是冷的还是冷的，或者是冷的或冷的，因为你想要受苦的娃娃因为或多或少的牛奶而丢失了，你想用身体上沉重的负担来安慰自己。你认为下一件事是你的吗？当下一代想要血液或一切时，他们会发生什么？安静。因为我在别人眼前都是这样做的，他们将被迫这样做，以至于他们无法做到，如果我打败他们，他们就会逃跑。我讨厌我掀起波澜，让你遭受版本的痛苦，但我希望你遭受希望的痛苦。

意大利？厨房里有很多东西或对他们的仇恨，所以我不知道他们喜欢吃什么，或者他们喜欢吃些什么。

## 块引用

块引用元素表示从另一个来源引用的内容，可以选择引用必须在“footer”或“cite”元素内，也可以选择内联更改，如注释和缩写。

#### 无归属块引用

> Tiam，ad mint，epu 和 noise 依次排列。
> **注意**您可以在块引用中使用\_Markdown 语法。

#### 归属块引用

> 不要通过分享记忆来交流，要通过交流来分享记忆。<br>
> — <cite>罗伯·皮克[^1]</cite>

[^1]: The above quote is excerpted from Rob Pike's [talk](https://www.youtube.com/watch?v=PAAkCSZUG1c) during Gopherfest, November 18, 2015.

## 表格

表格不是核心 Markdown 规范的一部分，但 Hugo 支持开箱即用。

| 姓名  | 年龄 |
| ----- | ---- |
| Bob   | 27   |
| Alice | 23   |

#### 表中的内联 Markdown

| Italics   | Bold     | Code   |
| --------- | -------- | ------ |
| _italics_ | **bold** | `code` |

| A                                    | B                                                    | C                                                                                                                            | D                              | E                                | F                                |
| ------------------------------------ | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------ | -------------------------------- | -------------------------------- |
| 当他痛苦的时候，他被认为是有帮助的。 | 愚者比愚者更聪明，愚者比愚昧者更令人憎恨，无人知晓。 | 当他当选时，他对自己说：“我不会去尘土，我也不会去尘土。”我们生活在一种担忧的状态中，但在一种尘土的状态下，害怕买到干净的价格 | 他想去那里，因为他不指挥汽车。 | 喝一件你需要穿的衣服，而不是穿。 | 但你错了，你错了。生活值得学习。 |

## 代码块

#### 带有反引号的代码块

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

#### 四个空格缩进的代码块

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

#### 带有 Hugo 内部高亮短代码的代码块

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

<style>
    .highlight {
        /* 你可以根据需要调整这个高度 */
        max-height: 400px;
        overflow: hidden;
    }

    .code-show {
        max-height: none !important;
    }

    .code-more-box {
        width: 100%;
        padding-top: 78px;
        background-image: -webkit-gradient(linear, left top, left bottom, from(rgba(255, 255, 255, 0)), to(#fff));
        position: absolute;
        left: 0;
        right: 0;
        bottom: 0;
        z-index: 1;
    }

    .code-more-btn {
        display: block;
        margin: auto;
        width: 44px;
        height: 22px;
        background: #f0f0f5;
        border-top-left-radius: 8px;
        border-top-right-radius: 8px;
        padding-top: 6px;
        cursor: pointer;
    }

    .code-more-img {
        cursor: pointer !important;
        display: block;
        margin: auto;
        width: 22px;
        height: 16px;
    }
</style>

<script>
  function initCodeMoreBox() {
    let codeBlocks = document.querySelectorAll(".highlight");
    if (!codeBlocks) {
      return;
    }
    codeBlocks.forEach(codeBlock => {
      // 校验是否overflow
      if (codeBlock.scrollHeight <= codeBlock.clientHeight) {
        return;
      }
      // 元素初始化
      // codeMoreBox
      let codeMoreBox = document.createElement('div');
      codeMoreBox.classList.add('code-more-box');
      // codeMoreBtn
      let codeMoreBtn = document.createElement('span');
      codeMoreBtn.classList.add('code-more-btn');
      codeMoreBtn.addEventListener('click', () => {
        codeBlock.classList.add('code-show');
        codeMoreBox.style.display = 'none';
        // 触发resize事件，重新计算目录位置
        window.dispatchEvent(new Event('resize'))
      })
      // img
      let img = document.createElement('img');
      img.classList.add('code-more-img');
      img.src = {{ (resources.Get "icons/codeMore.png").Permalink }}
      // 元素添加
      codeMoreBtn.appendChild(img);
      codeMoreBox.appendChild(codeMoreBtn);
      codeBlock.appendChild(codeMoreBox)
    })
  }
  
  initCodeMoreBox();
</script>

{{< /highlight >}}

#### 带变更显示的代码块

```diff
[dependencies.bevy]
git = "https://github.com/bevyengine/bevy"
rev = "11f52b8c72fc3a568e8bb4a4cd1f3eb025ac2e13"
- features = ["dynamic"]
+ features = ["jpeg", "dynamic"]
```

## 列表类型

#### 有序列表

1. 第一项
2. 第二项
3. 第三项

#### 无序列表

- 列表项
- 其他项
- 另一项

#### 嵌套列表

- 水果
  - 苹果
  - 橙子
  - 香蕉
- 乳品
  - 牛奶
  - 奶酪

## 其他元素：缩写、下标、上标、键盘输入、高亮标记

<abbr title="Graphics Interchange Format">GIF</abbr> is a bitmap image format.

H<sub>2</sub>O

X<sup>n</sup> + Y<sup>n</sup> = Z<sup>n</sup>

Press <kbd>CTRL</kbd> + <kbd>ALT</kbd> + <kbd>Delete</kbd> to end the session.

Most <mark>salamanders</mark> are nocturnal, and hunt for insects, worms, and other small creatures.

## Markdown 中的数学公式

假设我们有一个求和公式如下：

$$
S = \sum_{i=1}^{n} X_i
$$

其中：

- $S$ 表示总和，
- $n$ 是项数，
- $X_i$ 表示第$i$项的值。

## 带有超链接的图片

请切换黑色主题

[![Google](https://www.google.com/images/branding/googlelogo/1x/googlelogo_light_color_272x92dp.png)](https://google.com)
