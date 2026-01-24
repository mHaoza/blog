---
title: 博客 Docker 部署方案
tags: [Docker, Nuxt]
image: https://img.iice.fun/blog/2026/01/23/2ce3b36b3e2916d3a9645f9c4d1a1407.webp
date: 2026-01-15
---

记录一下给 Nuxt 博客的部署方案。

完整配置见 [GitHub](https://github.com/mHaoza/docker-service/tree/main/blog)

## 核心思路

**构建和运行分离**：

- **Builder 服务**：拉代码、装依赖、打包，完事就退出
- **App 服务**：跑应用，监听构建完成后自动重启

## Docker Compose 配置

```yaml
services:
  builder:
    build:
      dockerfile: Dockerfile.builder
    volumes:
      - workspace:/workspace
      - shared:/shared
    restart: "no"
    profiles: [build] # 手动触发，不自动启动

  app:
    build:
      dockerfile: Dockerfile.app
    volumes:
      - workspace:/workspace
      - shared:/shared
    ports:
      - "80:3000"
    restart: unless-stopped # 开机自启

volumes:
  workspace: # 存代码和产物
  shared: # 进程间通信
```

关键：

- `profiles: [build]` 让构建服务手动触发
- 用 named volume 不用 bind mount，避免权限问题
- 两个容器通过 shared volume 里的标记文件通信

## 构建服务

`build.sh`，拉代码、装依赖、打包，最后写标记文件：

```bash
#!/bin/sh
REPO_URL="https://gitee.com/mHaoza/sprout.git"
PROJECT_DIR="/workspace/sprout"
MARKER_FILE="/shared/.build-complete"

rm -f "$MARKER_FILE"

if [ -d "$PROJECT_DIR" ]; then
    cd "$PROJECT_DIR"
    git reset --hard origin/main    # 强制以远程为准
    git pull origin main
else
    git clone "$REPO_URL" "$PROJECT_DIR"
    cd "$PROJECT_DIR"
fi

pnpm install --frozen-lockfile --registry=https://registry.npmmirror.com
pnpm build

echo "$(date -Iseconds)" > "$MARKER_FILE"    # 通知 app 重启
```

## 运行服务

用 `inotify-tools` 监听标记文件，检测到新构建就重启：

```bash
#!/bin/sh
MARKER_FILE="/shared/.build-complete"

start_app() {
    cd /workspace/sprout
    NODE_ENV=production node .output/server/index.mjs &
    APP_PID=$!
}

start_app

while true; do
    inotifywait -e create,modify -t 10 /shared 2>/dev/null || true

    if [ -f "$MARKER_FILE" ]; then
        kill "$APP_PID" && wait "$APP_PID" 2>/dev/null || true
        start_app
        rm -f "$MARKER_FILE"
    fi

    sleep 2
done
```

## 使用

```bash
# 首次启动
docker compose up -d app

# 触发构建
docker compose run --rm builder

# 看日志
docker compose logs -f app
```

构建完成后 app 会自动重启。
