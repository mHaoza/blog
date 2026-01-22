---
title: 解决 VSCode OXC 插件初始化失败：从 fnm 切换到 Volta
tags: [VS Code, OXC]
image: https://img.iice.fun/blog/2026/01/22/cd9867ff7a18b1252b7a8c9eeeb473b1.webp
date: 2026-01-19
---

## 背景

最近在给一个基于 Nuxt 4 的项目集成 [OXC](https://oxc.rs)（Rust 写的超快 JS/TS 工具链），结果 VS Code 插件一启动就报「Server initialization failed」，整条工具链直接罢工。

---

## 现象

VS Code 输出面板（OXC）日志片段：

```log
2026-01-19 10:50:14.130 [info] Using server binary at: d:\projects\fon\node_modules\.bin\oxlint
2026-01-19 10:50:14.153 [info] 'node' 不是内部或外部命令，也不是可运行的程序

2026-01-19 10:50:14.155 [info] [Error - 10:50:14] Server initialization failed.
2026-01-19 10:50:14.155 [info]   Message: Pending response rejected since connection got disposed
  Code: -32097
2026-01-19 10:50:14.155 [info] [Info  - 10:50:14] Connection to server got closed. Server will restart.
2026-01-19 10:50:14.155 [info] true
2026-01-19 10:50:14.155 [info] [Error - 10:50:14] oxc client: couldn't create connection to server.
2026-01-19 10:50:14.156 [info]   Message: Pending response rejected since connection got disposed
  Code: -32097
2026-01-19 10:50:14.156 [info] [Error - 10:50:14] Server process exited with code 1.
```

关键报错：  
**'node' 不是内部或外部命令，也不是可运行的程序**  
→ 插件子进程找不到 `node` 可执行文件。

---

## 根因

OXC VS Code 插件在 Windows 上**直接 spawn 子进程**，并不会继承 fnm 的 shell 环境（`fnm env` 只在交互式 shell 生效）。于是子进程找不到 `node`，初始化直接失败。

相关 issue 参考：

- [oxc-project/oxc#17355](https://github.com/oxc-project/oxc/issues/17355)
- [oxc-project/oxc#15040](https://github.com/oxc-project/oxc/pull/15040)

---

## 当前解决方案
使用其他 Node 版本管理器替代 fnm。（此处使用 Volta）
1. 卸载 fnm
2. 安装 [Volta](https://volta.sh/)（另一个跨平台 Node 版本管理器）


## 小结

- Windows 上 GUI 应用（含 VS Code 插件）**只能读到系统 PATH**，fnm 的 shell hook 机制天生无法覆盖。
- Volta 通过「垫片」把可执行文件写进 PATH，对 GUI 和 CLI 一视同仁，兼容性更好。
