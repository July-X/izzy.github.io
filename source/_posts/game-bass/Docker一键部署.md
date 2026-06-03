---
title: "Docker 一键部署：从 0 到运行的游戏后端"
date: 2025-06-04 11:30:00
tags:
  - 游戏后端
  - Docker
  - DevOps
  - Go
categories:
  - 开发日志
top_img: false
---

<!-- more -->

`game-bass` 的目标是 5 分钟部署。实现方式是 Docker 多阶段构建 + docker-compose 编排，一条命令启动完整的后端服务。

## 多阶段构建：20MB 的最终镜像

```dockerfile
FROM golang:1.26-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server ./cmd/server

FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
WORKDIR /app
COPY --from=builder /app/server .
COPY --from=builder /app/scripts ./scripts
EXPOSE 8080
CMD ["./server"]
```

{% mermaid %}
graph LR
    A["golang:1.26-alpine"] --> B["编译阶段"]
    B --> C["go build -ldflags=-s -w"]
    C --> D["alpine:3.19"]
    D --> E["最终镜像 ~20MB"]
{% endmermaid %}

关键点：
- **`CGO_ENABLED=0`**：禁用 CGO，编译纯静态二进制，不依赖 glibc
- **`-ldflags="-s -w"`**：去掉符号表和调试信息，二进制从 ~15MB 压缩到 ~8MB
- **`alpine:3.19`**：最终镜像基于 Alpine，只有 ~5MB 基础层
- **`ca-certificates`**：安装 CA 证书，支持 HTTPS 请求
- **`tzdata`**：时区数据，支持服务端时间格式化

## docker-compose 编排

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --requirepass baas-dev
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "baas-dev", "ping"]
      interval: 2s
      timeout: 2s
      retries: 5

  server:
    build:
      context: ..
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - REDIS_ADDR=redis:6379
      - REDIS_PASSWORD=baas-dev
      - JWT_SECRET=game-bass-dev-secret
      - SERVER_ADDR=:8080
    volumes:
      - ./dev.license.dat:/license.dat
    depends_on:
      redis:
        condition: service_healthy
```

三个关键设计：

**1. 健康检查驱动的启动顺序**

```yaml
depends_on:
  redis:
    condition: service_healthy
```

不是简单的"Redis 容器启动了"，而是"Redis 通过健康检查了"。`redis-cli ping` 返回 PONG 才算健康，避免 server 连接还没就绪的 Redis。

**2. 环境变量配置**

所有配置通过环境变量注入，不需要配置文件。容器内读取 `REDIS_ADDR`、`JWT_SECRET` 等变量，开发和生产环境只需改环境变量值。

**3. 许可证文件挂载**

```yaml
volumes:
  - ./dev.license.dat:/license.dat
```

开发许可证通过 volume 挂载，不打包进镜像。生产环境换成正式许可证文件即可。

## 一条命令启动

```bash
cd deploy/
docker compose up -d
```

输出：
```
[+] Running 3/3
 ✔ Network deploy_default    Created
 ✔ Container deploy-redis-1  Healthy
 ✔ Container deploy-server-1 Started
```

Redis 先启动并通过健康检查，server 才启动。总启动时间约 5~10 秒。

## 开发 vs 生产

| 配置项 | 开发环境 | 生产环境 |
|--------|----------|----------|
| Redis 密码 | `baas-dev` | 强密码 |
| JWT Secret | `game-bass-dev-secret` | 随机 64 字节 |
| 许可证 | `dev.license.dat` | 正式许可证 |
| 端口 | `8080` | 反向代理后 `443` |
| 日志级别 | `debug` | `info` |

生产部署时复制 `docker-compose.yml`，修改环境变量值即可。镜像不变，配置分离。

## Godot SDK 连接

客户端通过 Godot SDK 连接：

```gdscript
var game_baas = preload("res://addons/game-bass/game_baas.gd").new()
game_baas.connect_to_server("http://localhost:8080")

# 匿名登录
var result = await game_baas.auth_anonymous()

# 提交排行榜
await game_baas.leaderboard_submit("daily", 1500)
```

SDK 通过 HTTP 调用 REST API，WebSocket 用于实时联机。所有端口统一 8080，不需要额外配置。

## 运维：日志和监控

```bash
# 查看日志
docker compose logs -f server

# 查看 Redis 数据
docker compose exec redis redis-cli -a baas-dev

# 重启服务
docker compose restart server

# 更新镜像
docker compose build server
docker compose up -d server
```

没有 Prometheus、Grafana、ELK。对于独立团队来说，`docker compose logs` 足够了。需要监控时再加，不过度工程化。

> 5 分钟部署不是一个口号，是一个工程约束。从 `git clone` 到 `docker compose up`，如果超过 5 分钟，说明部署流程需要优化。