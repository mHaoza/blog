---
title: 我的前端实践方案
image: https://img.iice.fun/blog/2026/01/24/d403d2368ff08e1f95ef2b45b33c42fe.webp
date: 2026-01-24
---

这里记录我个人在开发中使用的前端开发方案。

## 🛠️ 核心工具链

### Node 版本管理

- **[Volta](https://volta.sh/)** - 跨平台 Node 版本管理器
  - 通过「[垫片](https://developer.mozilla.org/zh-CN/docs/Glossary/Shim)」机制保证 GUI 应用也能找到 Node
  - 完美兼容 VS Code 插件等场景
- **[fnm](https://github.com/Schniz/fnm)** - 曾使用的工具，在 Windows 上对 GUI 应用支持不佳，已弃用

### 包管理器

- **[pnpm](https://pnpm.io/)** - 快速、节省磁盘空间的包管理器

### 代码质量

- **[oxc](https://oxc.rs)** - 高性能的 JavaScript/TypeScript 代码质量工具集（Linter + Formatter）
- **[simple-git-hooks](https://github.com/toplenboren/simple-git-hooks)** - 轻量级 Git Hooks 管理工具，在提交前进行代码检查

## 🚀 前端技术栈

- 构建工具：**[Vite](https://vitejs.dev/)** - 下一代前端构建工具
- JavaScript 框架：**[Vue 3](https://vuejs.org/)** - 渐进式 JavaScript 框架
- 全栈框架：**[Nuxt 4](https://nuxt.com/)** - Vue.js 全栈框架
- 状态管理：**[Pinia](https://pinia.vuejs.org/)** - Vue 3 官方推荐的状态管理库，提供类型安全的 Store
- UI 框架：**[Nuxt UI](https://nuxtui.org/)** - 组件丰富，提供官方 AI MCP 支持
- 样式方案：**[Tailwind CSS](https://tailwindcss.com/)** - 原子化 CSS 框架
- 工具函数库：**[VueUse](https://vueuse.org/)** - 基于组合式 API 的实用工具集合，包含大量常用组合式函数
- 跨平台开发：**[Tauri](https://tauri.app/)** - 使用 Web 技术构建桌面应用

## 💻 开发环境

### 编辑器

**[Visual Studio Code](https://code.visualstudio.com/)** - 强大的现代代码编辑器

### VS Code 扩展插件

#### 🎨 主题与美化

- [Atom One Dark Theme](https://marketplace.visualstudio.com/items?itemName=akamud.vscode-theme-onedark) - 经典的 Atom 配色主题
- [One Dark Pro](https://marketplace.visualstudio.com/items?itemName=zhuangtongfa.Material-theme) - 增强版 One Dark 主题
- [Material Icon Theme](https://marketplace.visualstudio.com/items?itemName=PKief.material-icon-theme) - Material Design 风格文件图标

#### 🤖 AI 辅助

- [Github Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) - AI 内联代码建议
- [Github Copilot Chat](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) - AI 聊天助手

#### 📝 编辑增强

- [Auto Rename Tag](https://marketplace.visualstudio.com/items?itemName=formulahendry.auto-rename-tag) - 自动重命名配对标签
- [Bookmarks](https://marketplace.visualstudio.com/items?itemName=alefragnani.Bookmarks) - 代码书签管理
- [Code Translate](https://marketplace.visualstudio.com/items?itemName=w88975.code-translate) - 划词翻译
- [EditorConfig](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig) - 统一编辑器配置

#### 🔍 代码质量

- [OXC](https://marketplace.visualstudio.com/items?itemName=oxc.oxc-vscode) - 高性能代码质量工具集成
- [Code Spell Checker](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) - 拼写检查
- [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint) - ESLint 支持（目前已使用OXC，兼容历史项目）
- [Prettier](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode) - 代码格式化（目前已使用OXC，兼容历史项目）

#### 🌿 Git 工具

- [GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens) - Git 功能增强
- [Git Graph](https://marketplace.visualstudio.com/items?itemName=mhutchie.git-graph) - 可视化 Git 历史

#### 🎨 CSS 与样式

- [Color Highlight](https://marketplace.visualstudio.com/items?itemName=naumovs.color-highlight) - 颜色值高亮显示
- [CSS Peek](https://marketplace.visualstudio.com/items?itemName=pranaygp.vscode-css-peek) - 快速查看 CSS 定义
- [Tailwind CSS IntelliSense](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss) - Tailwind CSS 智能提示
- [PostCSS Language Support](https://marketplace.visualstudio.com/items?itemName=csstools.postcss) - PostCSS 语法支持
- [VS Color Picker](https://marketplace.visualstudio.com/items?itemName=anseki.vscode-color-picker) - 颜色选择器

#### 🖼️ 图标与图像

- [Icônes](https://marketplace.visualstudio.com/items?itemName=vscode-icons-team.vscode-icons) - 快速查看 Iconify 图标库
- [Iconify IntelliSense](https://marketplace.visualstudio.com/items?itemName=antfu.iconify) - Iconify 图标智能提示
- [Image Preview](https://marketplace.visualstudio.com/items?itemName=kisstkondoros.vscode-gutter-preview) - 行内图片预览
- [Paste Image](https://marketplace.visualstudio.com/items?itemName=mushan.vscode-paste-image) - 直接粘贴图片并保存
- [SVG](https://marketplace.visualstudio.com/items?itemName=jock.svg) - SVG 编辑与预览

#### 📄 Markdown 增强

- [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one) - Markdown 编辑全家桶
- [Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced) - 强大的预览与导出功能
- [Markdown Preview Github Styling](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-preview-github-styles) - GitHub 风格预览

#### 🌐 语言与文件支持

- [DotENV](https://marketplace.visualstudio.com/items?itemName=mikestead.dotenv) - .env 文件语法高亮
- [Even Better TOML](https://marketplace.visualstudio.com/items?itemName=bungcip.better-toml) - TOML 文件支持
- [i18n Ally](https://marketplace.visualstudio.com/items?itemName=lokalise.i18n-ally) - 国际化开发助手
- [rust-analyzer](https://marketplace.visualstudio.com/items?itemName=matklad.rust-analyzer) - Rust 语言支持

#### 🔧 开发工具

- [Live Server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer) - 本地开发服务器
- [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) - 在编辑器内测试 API
- [JSON to TS](https://marketplace.visualstudio.com/items?itemName=MariusAlchimavicius.json-to-ts) - JSON 转 TypeScript 类型
- [Sort JSON objects](https://marketplace.visualstudio.com/items?itemName=richie5um2.vscode-sort-json) - JSON 对象排序
- [Draw.io Integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio) - 在 VS Code 中使用 draw.io 绘图
- [project-tree](https://marketplace.visualstudio.com/items?itemName=alefragnani.project-tree) - 生成项目目录树

#### 🚀 框架专用

- [Vue (Official)](https://marketplace.visualstudio.com/items?itemName=Vue.volar) - Vue 3 官方语言支持（原 Volar）
- [Tauri](https://marketplace.visualstudio.com/items?itemName=tauri-apps.tauri-vscode) - Tauri 开发支持

#### 🔗 远程开发

- [Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) - 通过 SSH 连接远程开发

## 🧠 AI 增强工具

### MCP 服务器（Model Context Protocol）

MCP 服务器为 AI 助手提供上下文感知能力，增强代码补全和建议质量。

#### 方式 1：从扩展市场安装(可选全局或工作区安装)

在 VS Code 扩展市场搜索时添加前缀 `@mcp`：

- **[Context7](https://github.com/upstash/context7)** - 提供丰富的库文档上下文
- **[Nuxt](https://nuxt.com/docs/4.x/guide/ai/mcp)** - Nuxt 框架官方 MCP 支持

#### 方式 2：自定义 MCP 服务器

在项目根目录创建 `.vscode/mcp.json` 文件配置自定义服务器：

```json
{
  "servers": {
    "nuxt-ui": {
      "type": "http",
      "url": "https://ui.nuxt.com/mcp"
    }
  }
}
```

**推荐 MCP 服务器：**

- **[Nuxt UI MCP](https://ui.nuxt.com/docs/getting-started/ai/mcp#visual-studio-code)** - Nuxt UI 组件库上下文

### AI Skills

通过 Skills 扩展 GitHub Copilot 的专项能力，可使用以下命令快速安装：

```sh
npx add-skill hyf0/vue-skills
```

或手动复制到 GitHub Copilot 的 `~/.copilot/skills` 目录。

**Skills：**

- **[vue-skills](https://github.com/hyf0/vue-skills)** - Vue 3 组合式 API、响应式系统优化
- **[threejs-skills](https://github.com/CloudAI-X/threejs-skills)** - Three.js 场景构建与性能优化

---

## 📌 更新日志

- **2026-01-24** - 初始版本，记录当前使用的前端技术栈
