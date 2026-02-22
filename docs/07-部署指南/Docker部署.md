# Docker 部署

本指南介绍如何使用 Docker 容器化部署 Univona Admin Server。

## Dockerfile

在项目根目录创建 `Dockerfile`：

```dockerfile
# ─── 构建阶段 ──────────────────────────────────────────────
FROM rust:1.92-bookworm AS builder

WORKDIR /usr/src/univona

# 先复制依赖清单，利用 Docker 缓存层
COPY Cargo.toml Cargo.lock ./

# 创建空的 src 目录用于预编译依赖
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release || true

# 复制完整源代码
COPY . .

# 重新构建（利用已缓存的依赖）
RUN touch src/main.rs && cargo build --release

# ─── 运行阶段 ──────────────────────────────────────────────
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

# 创建非 root 用户
RUN groupadd -r univona && useradd -r -g univona -d /opt/univona -s /sbin/nologin univona

WORKDIR /opt/univona

# 复制二进制文件
COPY --from=builder /usr/src/univona/target/release/univona-admin-server .

# 复制迁移文件
COPY --from=builder /usr/src/univona/migrations ./migrations

# 创建媒体存储目录
RUN mkdir -p /var/lib/univona/media && chown -R univona:univona /var/lib/univona

# 默认配置
COPY Univona.toml ./Univona.toml

USER univona

EXPOSE 3001

# 健康检查
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -sf http://localhost:3001/healthz || exit 1

ENTRYPOINT ["./univona-admin-server"]
```

### 带 Redis 缓存支持

```dockerfile
# 构建阶段中替换构建命令
RUN touch src/main.rs && cargo build --release --features redis-cache
```

## docker-compose.yml

创建 `docker-compose.yml`：

```yaml
version: "3.9"

services:
  # ─── Univona Admin Server ────────────────────────────────
  univona:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: univona-admin-server
    restart: unless-stopped
    ports:
      - "3001:3001"
    environment:
      # 数据库
      UNIVONA__DATABASE__URL: "postgres://univona:univona_password@postgres:5432/univona"
      UNIVONA__DATABASE__MAX_CONNECTIONS: "20"
      # 服务器
      UNIVONA__HOST: "0.0.0.0"
      UNIVONA__PORT: "3001"
      # 安全
      UNIVONA__SECURITY__CORS_ORIGINS: '["https://your-domain.com"]'
      JWT_SECRET: "${JWT_SECRET:?请设置 JWT_SECRET 环境变量}"
      UNIVONA_JWT_SECRET: "${UNIVONA_JWT_SECRET:?请设置 UNIVONA_JWT_SECRET 环境变量}"
      # 媒体
      MEDIA_STORAGE_DIR: "/var/lib/univona/media"
      # 日志
      UNIVONA__LOG__LEVEL: "info"
      RUST_LOG: "info"
      # Redis（可选）
      REDIS_URL: "redis://redis:6379"
    volumes:
      - univona_media:/var/lib/univona/media
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:3001/healthz"]
      interval: 30s
      timeout: 5s
      start_period: 15s
      retries: 3
    networks:
      - univona-net

  # ─── PostgreSQL ──────────────────────────────────────────
  postgres:
    image: postgres:16-bookworm
    container_name: univona-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: univona
      POSTGRES_PASSWORD: univona_password
      POSTGRES_DB: univona
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U univona -d univona"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - univona-net
    # 不对外暴露端口（仅内部网络访问）
    # 如需调试可取消注释
    # ports:
    #   - "5432:5432"

  # ─── Redis（可选） ───────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: univona-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - univona-net

volumes:
  postgres_data:
    driver: local
  univona_media:
    driver: local
  redis_data:
    driver: local

networks:
  univona-net:
    driver: bridge
```

## 环境变量配置

创建 `.env` 文件（**不要提交到版本控制**）：

```bash
# .env

# 必须设置的密钥
JWT_SECRET=your-strong-admin-jwt-secret-at-least-32-chars
UNIVONA_JWT_SECRET=your-strong-user-jwt-secret-at-least-32-chars

# 可选
ADMIN_API_KEY=your-admin-api-key
```

确保 `.env` 已添加到 `.gitignore`：

```bash
echo ".env" >> .gitignore
```

## 使用方法

### 构建和启动

```bash
# 构建镜像
docker compose build

# 启动所有服务（后台运行）
docker compose up -d

# 查看日志
docker compose logs -f univona

# 查看所有服务状态
docker compose ps
```

### 停止和清理

```bash
# 停止所有服务
docker compose down

# 停止并删除数据卷（注意：会丢失所有数据）
docker compose down -v
```

### 重新构建

```bash
# 重新构建并启动
docker compose up -d --build

# 仅重建 univona 服务
docker compose up -d --build univona
```

## 数据卷挂载

| 卷名 | 容器路径 | 说明 |
|------|----------|------|
| `postgres_data` | `/var/lib/postgresql/data` | PostgreSQL 数据文件 |
| `univona_media` | `/var/lib/univona/media` | 媒体文件（图片、音频等） |
| `redis_data` | `/data` | Redis 持久化数据 |

### 使用主机目录绑定

如需将数据存储在主机指定位置：

```yaml
volumes:
  - /data/univona/postgres:/var/lib/postgresql/data
  - /data/univona/media:/var/lib/univona/media
  - /data/univona/redis:/data
```

## 健康检查

所有服务均配置了健康检查：

| 服务 | 检查方式 | 间隔 | 超时 |
|------|----------|------|------|
| univona | `curl /healthz` | 30s | 5s |
| postgres | `pg_isready` | 10s | 5s |
| redis | `redis-cli ping` | 10s | 5s |

查看健康状态：

```bash
# 查看所有容器健康状态
docker compose ps

# 查看特定容器健康检查日志
docker inspect --format='{{json .State.Health}}' univona-admin-server | jq
```

## 生产环境增强

### 带 nginx 的完整部署

```yaml
# docker-compose.prod.yml

version: "3.9"

services:
  nginx:
    image: nginx:1.27-alpine
    container_name: univona-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - certbot_certs:/etc/letsencrypt:ro
      - certbot_webroot:/var/www/certbot:ro
    depends_on:
      univona:
        condition: service_healthy
    networks:
      - univona-net

  certbot:
    image: certbot/certbot
    container_name: univona-certbot
    volumes:
      - certbot_certs:/etc/letsencrypt
      - certbot_webroot:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  certbot_certs:
    driver: local
  certbot_webroot:
    driver: local
```

启动生产环境：

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### 资源限制

```yaml
services:
  univona:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 512M

  postgres:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
    command: >
      postgres
      -c shared_buffers=512MB
      -c effective_cache_size=1536MB
      -c work_mem=16MB

  redis:
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
```

## 数据库备份

### 手动备份

```bash
# 创建备份
docker compose exec postgres pg_dump -U univona -Fc univona > backup_$(date +%Y%m%d).dump

# 恢复备份
docker compose exec -i postgres pg_restore -U univona -d univona -c < backup_20260215.dump
```

### 自动备份脚本

```bash
#!/bin/bash
# backup-docker.sh

BACKUP_DIR="/var/backups/univona"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# 数据库备份
docker compose exec -T postgres pg_dump -U univona -Fc univona > "$BACKUP_DIR/db_${DATE}.dump"

# 媒体文件备份（使用 docker cp）
docker cp univona-admin-server:/var/lib/univona/media "$BACKUP_DIR/media_${DATE}"

# 清理 30 天前的备份
find "$BACKUP_DIR" -mtime +30 -delete

echo "备份完成: $DATE"
```

## 故障排查

### 常见问题

**容器无法启动：**

```bash
# 查看详细日志
docker compose logs univona

# 检查容器退出码
docker inspect --format='{{.State.ExitCode}}' univona-admin-server
```

**数据库连接失败：**

```bash
# 确认 PostgreSQL 已就绪
docker compose exec postgres pg_isready -U univona

# 测试连接
docker compose exec univona curl -sf http://localhost:3001/healthz
```

**磁盘空间不足：**

```bash
# 查看 Docker 磁盘使用
docker system df

# 清理未使用的资源
docker system prune -a --volumes
```
