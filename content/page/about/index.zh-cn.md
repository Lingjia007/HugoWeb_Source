---
title: 关于
menu:
  main:
    weight: -90
    params:
      icon: user
---

本站基于 [Hugo](https://gohugo.io/) 静态站点生成器构建，使用 [hugo-theme-stack](https://github.com/CaiJimmy/hugo-theme-stack) 主题，通过 GitHub Actions 工作流实现**双平台自动部署**——推送到 `main` 分支即触发构建与发布，无需手动操作。

## 部署架构

| 部署目标     | 访问地址                               | 发布分支  | 说明                                              |
| ------------ | -------------------------------------- | --------- | ------------------------------------------------- |
| GitHub Pages | https://lingjia007.github.io/Hugo_Web/ | `master`  | 主站点，使用源仓库 `hugo.yaml` 中的 baseURL       |
| Netlify      | https://lingsir007.netlify.app/        | `netlify` | 镜像站点，工作流会自动改写 baseURL 以适配独立域名 |

源代码仓库与发布仓库分离：构建产物被推送到外部仓库 `Lingjia007/Hugo_Web` 的对应分支，由 GitHub Pages / Netlify 自动拉取发布。

## 工作流详解

工作流定义于 [.github/workflows/deploy.yml](https://github.com/Lingjia007/HugoWeb_Source/blob/main/.github/workflows/deploy.yml)，包含两个并行 Job。

### Job 1：`deploy-to-gh-pages`

推送到 GitHub Pages 的流程：

1. **Checkout** — 拉取源仓库（含子模块，`fetch-depth: 0` 保留完整 Git 历史用于 `lastmod` 时间戳）
2. **Git Configuration** — 关闭 `quotePath`、`autocrlf`，开启 `safecrlf`，避免中文路径与 CRLF 问题
3. **Set up Hugo** — 通过 `peaceiris/actions-hugo@v2` 安装最新版 Hugo
4. **Build** — `hugo -F --cleanDestinationDir` 构建并清空旧产物
5. **Fix favicon path** — 将 `public/` 下所有 HTML 文件中的 `/icons/favicon.ico` 替换为 `/Hugo_Web/icons/favicon.ico`，适配 GitHub Pages 子路径：

   ```bash
   find public -name "*.html" -exec sed -i 's|href="/icons/favicon.ico"|href="/Hugo_Web/icons/favicon.ico"|g' {} +
   ```

6. **Copy static and assets** — 将 `static/` 与 `assets/` 复制到 `public/en`、`public/zh-hk` 多语言目录
7. **Deploy** — 通过 `peaceiris/actions-gh-pages@v3` 将 `./public` 推送到 `Lingjia007/Hugo_Web` 的 `master` 分支，提交信息沿用 HEAD commit message

### Job 2：`deploy-to-netlify`

推送到 Netlify 分支的流程，差异在于构建前需先改写配置以适配独立域名：

1. **Checkout main** — 显式指定 `ref: main`
2. **Git Configuration** — 同上
3. **Switch to netlify branch** — `git checkout -B netlify` 新建并切换
4. **Modify baseURL** — `sed` 将 `hugo.yaml` 首行 baseURL 从 GitHub Pages 地址替换为 Netlify 地址：

   ```bash
   sed -i '1s#https://lingjia007.github.io/Hugo_Web/#https://lingsir007.netlify.app/#' hugo.yaml
   ```

5. **Modify Waifu Tips Path** — 看板娘提示文件中去除 `Hugo_Web/` 路径前缀，避免子路径错误：

   ```bash
   sed -i 's#Hugo_Web/##g' static/js/my-waifu-tips.json
   ```

6. **Commit & Push** — 以 GitHub Actions 身份提交配置变更，`--force` 强制推送到 `netlify` 分支
7. **Set up Hugo + Build** — 同 Job 1
8. **Deploy** — 推送到 `Lingjia007/Hugo_Web` 的 `netlify` 分支

## 触发与密钥

- **触发条件**：`push` 到 `main` 分支
- **认证方式**：使用仓库 Secret `TOKEN`（Personal Access Token），授予对外部仓库 `Lingjia007/Hugo_Web` 的写入权限
- **权限声明**：`deploy-to-netlify` Job 声明 `permissions: contents: write`，确保 `git push` 操作被允许

## 本地预览

提交前可使用以下命令在本地预览：

```bash
hugo server -D
```

如需模拟 GitHub Pages 的子路径部署效果：

```bash
hugo server -D --baseURL https://lingjia007.github.io/Hugo_Web/ --appendPort=false
```
