---
title: About
menu:
    main: 
        weight: -90
        params:
            icon: user
---

This site is built with [Hugo](https://gohugo.io/) static site generator, using the [hugo-theme-stack](https://github.com/CaiJimmy/hugo-theme-stack) theme. It is deployed automatically to **dual platforms** through GitHub Actions — pushing to the `main` branch triggers the build and publish workflow, no manual operation required.

## Deployment Architecture

| Target | URL | Branch | Notes |
| ------ | --- | ------ | ----- |
| GitHub Pages | https://lingjia007.github.io/Hugo_Web/ | `master` | Primary site, uses baseURL from source repo's `hugo.yaml` |
| Netlify | https://lingsir007.netlify.app/ | `netlify` | Mirror site, workflow auto-rewrites baseURL for the standalone domain |

The source repository and publish repository are separated: build artifacts are pushed to the external repository `Lingjia007/Hugo_Web` on the corresponding branch, then pulled and published automatically by GitHub Pages / Netlify.

## Workflow Details

The workflow is defined in [.github/workflows/deploy.yml](https://github.com/Lingjia007/HugoWeb_Source/blob/main/.github/workflows/deploy.yml) and contains two parallel jobs.

### Job 1: `deploy-to-gh-pages`

Pipeline to GitHub Pages:

1. **Checkout** — Clone source repo (with submodules, `fetch-depth: 0` to preserve full Git history for `lastmod` timestamps)
2. **Git Configuration** — Disable `quotePath`, `autocrlf`; enable `safecrlf` to avoid CJK path and CRLF issues
3. **Set up Hugo** — Install latest Hugo via `peaceiris/actions-hugo@v2`
4. **Build** — `hugo -F --cleanDestinationDir` to build and clean stale artifacts
5. **Copy static and assets** — Copy `static/` and `assets/` into `public/en`, `public/zh-hk` multilingual directories
6. **Deploy** — Push `./public` to the `master` branch of `Lingjia007/Hugo_Web` via `peaceiris/actions-gh-pages@v3`, inheriting the HEAD commit message

### Job 2: `deploy-to-netlify`

Pipeline to the Netlify branch. The difference is that the config must be rewritten before build to adapt the standalone domain:

1. **Checkout main** — Explicitly specify `ref: main`
2. **Git Configuration** — Same as above
3. **Switch to netlify branch** — `git checkout -B netlify` to create and switch
4. **Modify baseURL** — `sed` replaces the first line of baseURL in `hugo.yaml` from the GitHub Pages URL to the Netlify URL:

   ```bash
   sed -i '1s#https://lingjia007.github.io/Hugo_Web/#https://lingsir007.netlify.app/#' hugo.yaml
   ```

5. **Modify Waifu Tips Path** — Strip the `Hugo_Web/` path prefix from the waifu tips file to avoid subpath errors:

   ```bash
   sed -i 's#Hugo_Web/##g' static/js/my-waifu-tips.json
   ```

6. **Commit & Push** — Commit config changes as GitHub Actions, force-push to the `netlify` branch
7. **Set up Hugo + Build** — Same as Job 1
8. **Deploy** — Push to the `netlify` branch of `Lingjia007/Hugo_Web`

## Triggers & Secrets

- **Trigger**: `push` to the `main` branch
- **Authentication**: Uses the repository Secret `TOKEN` (Personal Access Token), granting write access to the external repository `Lingjia007/Hugo_Web`
- **Permissions**: The `deploy-to-netlify` job declares `permissions: contents: write` to allow `git push` operations

## Local Preview

Preview locally before committing:

```bash
hugo server -D
```

To simulate the GitHub Pages subpath deployment:

```bash
hugo server -D --baseURL https://lingjia007.github.io/Hugo_Web/ --appendPort=false
```
