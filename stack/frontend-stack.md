---
title: 我的前端实践方案
image: https://img.iice.fun/blog/2026/01/24/d403d2368ff08e1f95ef2b45b33c42fe.webp
date: 2026-01-24
---

这里记录我个人在开发中使用的前端开发方案。

## 1. 🛠️ 核心工具链

### 1.1 Node 版本管理

- **[Volta](https://volta.sh/)** - 跨平台 Node 版本管理器
  - 通过「[垫片](https://developer.mozilla.org/zh-CN/docs/Glossary/Shim)」机制保证 GUI 应用也能找到 Node
  - 完美兼容 VS Code 插件等场景
- **[fnm](https://github.com/Schniz/fnm)** - 曾使用的工具，在 Windows 上对 GUI 应用支持不佳，已弃用

### 1.2 包管理器

- **[pnpm](https://pnpm.io/)** - 快速、节省磁盘空间的包管理器

### 1.3 代码质量

- **[oxc](https://oxc.rs)** - 高性能的 JavaScript/TypeScript 代码质量工具集（Linter + Formatter）
- **[simple-git-hooks](https://github.com/toplenboren/simple-git-hooks)** - 轻量级 Git Hooks 管理工具，在提交前进行代码检查

## 2. 🚀 前端技术栈

- 构建工具：**[Vite](https://vitejs.dev/)** - 下一代前端构建工具
- JavaScript 框架：**[Vue 3](https://vuejs.org/)** - 渐进式 JavaScript 框架
- 全栈框架：**[Nuxt 4](https://nuxt.com/)** - Vue.js 全栈框架
- 状态管理：**[Pinia](https://pinia.vuejs.org/)** - Vue 3 官方推荐的状态管理库，提供类型安全的 Store
- UI 框架：**[Nuxt UI](https://nuxtui.org/)** - 组件丰富，提供官方 AI MCP 支持
- 样式方案：**[Tailwind CSS](https://tailwindcss.com/)** - 原子化 CSS 框架
- 工具函数库：**[VueUse](https://vueuse.org/)** - 基于组合式 API 的实用工具集合，包含大量常用组合式函数
- 跨平台开发：**[Tauri](https://tauri.app/)** - 使用 Web 技术构建桌面应用

## 3. 💻 开发环境

### 3.1 编辑器

**[Visual Studio Code](https://code.visualstudio.com/)** - 强大的现代代码编辑器

## 4. 📌 更新日志

- **2026-01-24** - 初始版本，记录当前使用的前端技术栈
- **2026-08-21** - 移除了vscode拓展，详见[VS Code 拓展合集](https://www.iice.fun/more/vscode-extends)
