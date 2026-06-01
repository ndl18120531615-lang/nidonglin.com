# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Firefly（流萤）是一个基于 Astro v6 的个人博客主题模板，Fork 自 [saicaca/fuwari](https://github.com/saicaca/fuwari)。项目用 TypeScript + Svelte 编写，Tailwind CSS v4 作为样式系统，Biome 做代码格式化。

## 常用命令

```bash
pnpm install              # 安装依赖（必须用 pnpm，preinstall 钩子会拦截其他包管理器）
pnpm dev                  # 启动开发服务器 → http://localhost:4321
pnpm build                # 完整构建：生成图标 → LQIP → astro build → pagefind 索引
pnpm preview              # 预览构建产物
pnpm check                # astro check（检查 Astro/TS 错误）
pnpm type-check           # tsc --noEmit 类型检查
pnpm format               # Biome 格式化 ./src
pnpm lint                 # Biome 检查并自动修复 ./src
pnpm new-post <filename>  # 创建新文章
pnpm icons                # 仅重新生成图标
pnpm lqips                # 仅重新生成 LQIP（低质量图像占位符）
```

## 架构概览

### 布局双层结构

- **`src/layouts/Layout.astro`** — 基础 HTML 外壳：`<head>`、SEO meta、分析脚本、主题初始化（内联 script 在页面渲染前设置亮/暗模式和壁纸模式，避免闪烁）、字体管理、樱花特效。
- **`src/layouts/MainGridLayout.astro`** — 网格布局层：Navbar、壁纸/横幅（含走马灯和 Ken Burns 动画）、CSS Grid 主内容区（左侧栏 + 主内容 + 右侧栏）、Footer、浮动控件、Live2D/Spine 看板娘。Swup 页面过渡的所有钩子逻辑都在此文件的 `<script>` 块中。

### 内容系统

使用 Astro Content Collections（`src/content.config.ts`），基于 glob loader：

| 集合 | 路径 | 用途 |
|------|------|------|
| `posts` | `src/content/posts/**/*.{md,mdx}` | 博客文章，包含 title/published/tags/category/pinned/draft/password 等 frontmatter |
| `spec` | `src/content/spec/**/*.{md,mdx}` | 静态页面：about、friends、guestbook |

### 配置系统 (`src/config/`)

所有配置集中在 `src/config/` 目录，通过 `index.ts` 统一 re-export。核心配置文件：

- `siteConfig.ts` — 站点标题、URL、主题色、导航栏、SEO、分页、分析、图像优化、页面开关等
- `sidebarConfig.ts` — 左侧栏/右侧栏/双侧栏布局、组件可见性、文章页临时双侧栏
- `backgroundWallpaper.ts` — 壁纸模式（banner/fullscreen/overlay/none）、走马灯、水波纹、渐变
- `profileConfig.ts` — 用户资料（头像、名称、社交链接）
- `commentConfig.ts` — 评论系统（Waline/Twikoo/Giscus/Artalk/Disqus）
- `fontConfig.ts` — 自定义字体选择
- 其他：`navBarConfig.ts`、`musicConfig.ts`、`friendsConfig.ts`、`galleryConfig.ts` 等

### 页面路由 (`src/pages/`)

路由即 Astro 文件约定。关键页面：
- `[...page].astro` — 首页 + 分页
- `posts/[...slug].astro` — 文章详情页
- `archive.astro`、`about.astro`、`friends.astro`、`guestbook.astro`、`bangumi.astro`、`search.astro`、`sponsor.astro`
- `gallery/index.astro`、`gallery/[album].astro` — 相册
- `api/allPostMeta.json.ts` — 提供文章元数据的 JSON API
- `rss.xml.ts` — RSS 订阅源
- `og/[...slug].png.ts` — OpenGraph 图片动态生成（Satori）

### 组件分类 (`src/components/`)

- **`common/`** — 通用组件：Pagination、Markdown 渲染、CoverImage、Icon (Svelte)、Dropdown 等
- **`layout/`** — 布局组件：Navbar、SideBar、Footer、PostCard、PostMeta、CategoryBar
- **`controls/`** — UI 控件：亮暗切换 (Svelte)、搜索 (Svelte)、壁纸切换 (Svelte)、回到顶部、浮动目录等
- **`widget/`** — 侧边栏小组件：Profile、Calendar、Categories、Tags、Music、SiteStats、Announcement
- **`features/`** — 功能组件：加密内容、音乐播放器、Live2D、Spine 模型、Fancybox、KaTeX、樱花特效
- **`analytics/`** — 统计分析：Google Analytics、Umami、Microsoft Clarity、51la
- **`comment/`** — 评论系统：Waline、Twikoo、Giscus、Artalk、Disqus

### Svelte 交互组件

部分组件使用 Svelte 5（`src/components/common/*.svelte`），这些是客户端交互组件：
- `Search.svelte`、`AdvancedSearch.svelte` — 搜索面板
- `LightDarkSwitch.svelte`、`WallpaperSwitch.svelte` — 显示设置
- `SharePoster.svelte` — 分享海报生成
- `Icon.svelte` — 图标加载器
- `DropdownItem.svelte`、`DropdownPanel.svelte` — 下拉菜单
- `LayoutSwitchButton.svelte` — 列表/网格布局切换

### Markdown 处理管道

在 `astro.config.mjs` 中配置了 unified 处理管道：

| 阶段 | 插件 | 功能 |
|------|------|------|
| Remark | `remarkMath` | 数学公式 |
| Remark | `remarkReadingTime` (`plugins/remark-reading-time.mjs`) | 阅读时间估算 |
| Remark | `remarkImageGrid` (`plugins/remark-image-grid.js`) | 图片网格布局 |
| Remark | `remarkExcerpt` (`plugins/remark-excerpt.js`) | 自动摘要提取 |
| Remark | `remarkDirective` + `parseDirectiveNode` | 自定义指令（如 GitHub 卡片） |
| Remark | `remarkSectionize` | 内容分段 |
| Remark | `remarkMermaid` / `remarkPlantuml` | 图表支持 |
| Rehype | `rehypeKatex` | KaTeX 数学渲染 |
| Rehype | `rehypeCallouts` | 提醒块（GitHub/Obsidian/VitePress 风格） |
| Rehype | `rehypeSlug` + `rehypeAutolinkHeadings` | 标题锚点链接 |
| Rehype | `rehypeFigure` | 图片包装为 figure |
| Rehype | `rehypeExternalLinks` | 外链处理（新窗口打开） |
| Rehype | `rehypeEmailProtection` | 邮箱混淆保护 |
| Rehype | `rehypeComponents` + `GithubCardComponent` | GitHub 仓库卡片 |
| Rehype | `rehypeMermaid` / `rehypePlantuml` | 图表渲染 |

### i18n 系统 (`src/i18n/`)

- `i18nKey.ts` — 定义所有翻译键的枚举
- `languages/{zh_CN,zh_TW,en,ja,ru}.ts` — 各语言的翻译映射
- `translation.ts` — `i18n(key)` 函数，根据 `siteConfig.lang` 返回对应翻译；缺失翻译时回退到简体中文再回退到英文
- 支持的语言代码在 `siteConfig.ts` 的 `SITE_LANG` 设置

### Swup 页面过渡

Swup 用于 SPA 式的页面过渡体验。在 `astro.config.mjs` 中配置 containers，这些 DOM 元素在页面切换时被替换：

```
#banner-overlay-container, #banner-dim-container, #swup-container,
#left-sidebar-dynamic, #right-sidebar-dynamic, #floating-toc-wrapper
```

**重要**：所有 Swup containers 必须在 DOM 中存在，否则 Swup 会回退到整页加载。`MainGridLayout.astro` 中有复杂逻辑确保所有模式下这些容器都存在（包括隐藏的占位容器）。

### 壁纸模式

四种壁纸模式，用户可在设置中切换：
- `banner` — 首页显示全宽横幅，非首页显示小横幅
- `fullscreen` — 始终全屏壁纸
- `overlay` — 全屏半透明壁纸叠加在内容上方
- `none` — 纯色背景

### 路径别名

```typescript
@/*            → ./src/*
@components/*  → ./src/components/*
@utils/*       → ./src/utils/*
@i18n/*        → ./src/i18n/*
@constants/*   → ./src/constants/*
@assets/*      → ./src/assets/*
@layouts/*     → ./src/layouts/*
```

## CSS 体系

- **Tailwind CSS v4** — 通过 `@tailwindcss/vite` 插件集成
- **Stylus** — 部分样式使用 `.styl` 文件（`src/styles/variables.styl`、`src/styles/markdown-extend.styl`）
- 全局样式入口：`src/styles/main.css`
- 主题色通过 CSS 变量 `--hue`（0-360 色相）驱动，在 `Layout.astro` 内联设置
- `expressive-code.css` 和 `markdown.css` 在 `MainGridLayout.astro` 中加载

## 构建流程

`pnpm build` 按顺序执行：
1. `scripts/generate-icons.js` — 根据配置中使用的图标，生成优化的图标集合文件以减少 bundle 体积
2. `scripts/generate-lqips.ts` — 为背景图片生成 LQIP（低质量图像占位符），存入 `src/constants/lqips.json`
3. `astro build` — Astro 构建，输出到 `dist/`
4. `pagefind --site dist` — 为静态站点生成全文搜索索引

`scripts/new-post.js` 会创建带有所有必需 frontmatter 的模板文章文件。

## 技术栈注意事项

- Astro v6 已弃用 `markdown.remarkPlugins`/`markdown.rehypePlugins`，改用 `unified({...})` from `@astrojs/markdown-remark`
- Svelte 5 使用 runes 语法（`$state`、`$derived`、`$effect`）
- 使用 `pnpm` 作为唯一包管理器，版本需 ≥ 9
- Node.js 版本需 ≥ 22
- Biome 同时用于格式化和 linting，无 ESLint/Prettier
- `trailingSlash: "always"` — 所有 URL 以 `/` 结尾
