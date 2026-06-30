---
title: 關於
menu:
  main:
    weight: -90
    params:
      icon: user
---

本站基於 [Hugo](https://gohugo.io/) 靜態站點生成器構建，使用 [hugo-theme-stack](https://github.com/CaiJimmy/hugo-theme-stack) 主題，透過 GitHub Actions 工作流實現**雙平台自動部署**——推送到 `main` 分支即觸發構建與發布，無需手動操作。

## 部署架構

| 部署目標     | 存取地址                               | 發布分支  | 說明                                              |
| ------------ | -------------------------------------- | --------- | ------------------------------------------------- |
| GitHub Pages | https://lingjia007.github.io/Hugo_Web/ | `master`  | 主站點，使用源倉庫 `hugo.yaml` 中的 baseURL       |
| Netlify      | https://lingsir007.netlify.app/        | `netlify` | 鏡像站點，工作流會自動改寫 baseURL 以適配獨立域名 |

源代碼倉庫與發布倉庫分離：構建產物被推送到外部倉庫 `Lingjia007/Hugo_Web` 的對應分支，由 GitHub Pages / Netlify 自動拉取發布。

## 工作流詳解

工作流定義於 [.github/workflows/deploy.yml](https://github.com/Lingjia007/HugoWeb_Source/blob/main/.github/workflows/deploy.yml)，包含兩個並行 Job。

### Job 1：`deploy-to-gh-pages`

推送到 GitHub Pages 的流程：

1. **Checkout** — 拉取源倉庫（含子模組，`fetch-depth: 0` 保留完整 Git 歷史用於 `lastmod` 時間戳）
2. **Git Configuration** — 關閉 `quotePath`、`autocrlf`，開啟 `safecrlf`，避免中文路徑與 CRLF 問題
3. **Set up Hugo** — 透過 `peaceiris/actions-hugo@v2` 安裝最新版 Hugo
4. **Build** — `hugo -F --cleanDestinationDir` 構建並清空舊產物
5. **Copy static and assets** — 將 `static/` 與 `assets/` 複製到 `public/en`、`public/zh-hk` 多語言目錄
6. **Deploy** — 透過 `peaceiris/actions-gh-pages@v3` 將 `./public` 推送到 `Lingjia007/Hugo_Web` 的 `master` 分支，提交訊息沿用 HEAD commit message

### Job 2：`deploy-to-netlify`

推送到 Netlify 分支的流程，差異在於構建前需先改寫配置以適配獨立域名：

1. **Checkout main** — 顯式指定 `ref: main`
2. **Git Configuration** — 同上
3. **Switch to netlify branch** — `git checkout -B netlify` 新建並切換
4. **Modify baseURL** — `sed` 將 `hugo.yaml` 首行 baseURL 從 GitHub Pages 地址替換為 Netlify 地址：

   ```bash
   sed -i '1s#https://lingjia007.github.io/Hugo_Web/#https://lingsir007.netlify.app/#' hugo.yaml
   ```

5. **Modify Waifu Tips Path** — 看板娘提示檔案中去除 `Hugo_Web/` 路徑前綴，避免子路徑錯誤：

   ```bash
   sed -i 's#Hugo_Web/##g' static/js/my-waifu-tips.json
   ```

6. **Commit & Push** — 以 GitHub Actions 身份提交配置變更，`--force` 強制推送到 `netlify` 分支
7. **Set up Hugo + Build** — 同 Job 1
8. **Deploy** — 推送到 `Lingjia007/Hugo_Web` 的 `netlify` 分支

## 觸發與密鑰

- **觸發條件**：`push` 到 `main` 分支
- **認證方式**：使用倉庫 Secret `TOKEN`（Personal Access Token），授予對外部倉庫 `Lingjia007/Hugo_Web` 的寫入權限
- **權限宣告**：`deploy-to-netlify` Job 宣告 `permissions: contents: write`，確保 `git push` 操作被允許

## 本地預覽

提交前可使用以下命令在本地預覽：

```bash
hugo server -D
```

如需模擬 GitHub Pages 的子路徑部署效果：

```bash
hugo server -D --baseURL https://lingjia007.github.io/Hugo_Web/ --appendPort=false
```
