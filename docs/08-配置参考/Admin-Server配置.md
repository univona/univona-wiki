# Admin Server 配置参考

Univona Admin Server 使用 [Figment](https://docs.rs/figment) 进行配置管理，支持多种配置源。

## 配置加载顺序

配置按以下顺序加载，后加载的值覆盖先加载的值：

1. `univona-daemon.toml` - 兼容旧版 daemon 配置文件
2. `Univona.toml` - 主配置文件（推荐）
3. `UNIVONA_` 前缀的环境变量 - 最高优先级

> 对于嵌套配置，环境变量使用双下划线 `__` 作为分隔符。例如 `UNIVONA__DATABASE__URL` 对应 `[database] url`。

## Univona.toml 完整配置

```toml
# ─── 服务器 ────────────────────────────────────────────────
# 监听地址
# 默认值: "0.0.0.0"
host = "0.0.0.0"

# 监听端口
# 默认值: 3001
port = 3001

# ─── 数据库 ────────────────────────────────────────────────
[database]
# PostgreSQL 连接字符串（必须）
url = "postgres://user:password@localhost:5432/univona"

# 最大连接数
# 默认值: 10
max_connections = 10

# ─── Redis（可选） ─────────────────────────────────────────
[redis]
# Redis 连接地址
# 需要启用 redis-cache feature 编译
url = "redis://localhost:6379"

# ─── 日志 ──────────────────────────────────────────────────
[log]
# 日志级别: trace, debug, info, warn, error
# 默认值: "info"
# 也可通过 RUST_LOG 环境变量覆盖
level = "info"

# 是否使用 JSON 格式输出日志
# 默认值: false
# 生产环境推荐开启，便于日志收集工具解析
json = false

# ─── 安全 ──────────────────────────────────────────────────
[security]
# CORS 允许的源列表
# 默认值: ["http://localhost:3000"]
# 设置 ["*"] 等同于允许所有源（仅建议开发环境使用）
cors_origins = ["https://your-domain.com", "https://admin.your-domain.com"]

# ─── 推送通知（可选） ──────────────────────────────────────
# 如果不配置推送，用户仍可使用所有功能，只是不会收到推送通知

[push.fcm]
# Google Cloud 项目 ID
project_id = "your-gcp-project-id"

# FCM 服务账号 JSON 文件路径
service_account_path = "/etc/univona/fcm-service-account.json"

[push.apns]
# APNs .p8 私钥文件路径
key_path = "/etc/univona/apns-key.p8"

# Apple Developer 中的 Key ID
key_id = "ABCDE12345"

# Apple Developer 中的 Team ID
team_id = "TEAM123456"

# App Bundle ID（用作 APNs topic）
bundle_id = "com.your.app"

# 是否使用沙盒环境
# 默认值: false
sandbox = false
```

## 配置分组说明

### server（顶级字段）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `host` | String | `"0.0.0.0"` | 监听地址。生产环境建议设为 `"127.0.0.1"`，通过反向代理访问 |
| `port` | u16 | `3001` | 监听端口 |

### database

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `url` | String | **必须** | PostgreSQL 连接字符串，格式：`postgres://user:pass@host:port/dbname` |
| `max_connections` | u32 | `10` | 连接池最大连接数。推荐值：CPU 核数 * 2 + 磁盘数 |

### redis（可选）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `url` | String | — | Redis 连接地址。需要使用 `--features redis-cache` 编译 |

### log

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `level` | String | `"info"` | 日志级别。可选：`trace`、`debug`、`info`、`warn`、`error` |
| `json` | bool | `false` | 是否使用 JSON 格式输出。生产环境推荐开启 |

### security

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `cors_origins` | Vec\<String\> | `["http://localhost:3000"]` | CORS 允许源列表。`["*"]` 允许所有源 |

### push.fcm（可选）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `project_id` | String | — | Google Cloud 项目 ID |
| `service_account_path` | String | — | FCM 服务账号 JSON 文件路径 |

### push.apns（可选）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `key_path` | String | — | APNs .p8 私钥文件路径 |
| `key_id` | String | — | Apple Developer Key ID |
| `team_id` | String | — | Apple Developer Team ID |
| `bundle_id` | String | — | App Bundle ID |
| `sandbox` | bool | `false` | 是否使用 APNs 沙盒环境 |

## 环境变量覆盖示例

配置文件中的所有选项都可以通过环境变量覆盖：

```bash
# 顶级字段
export UNIVONA__HOST="127.0.0.1"
export UNIVONA__PORT="8080"

# 数据库配置
export UNIVONA__DATABASE__URL="postgres://user:pass@localhost/univona"
export UNIVONA__DATABASE__MAX_CONNECTIONS="20"

# 日志配置
export UNIVONA__LOG__LEVEL="debug"
export UNIVONA__LOG__JSON="true"

# 安全配置
export UNIVONA__SECURITY__CORS_ORIGINS='["https://example.com"]'

# Redis 配置
export UNIVONA__REDIS__URL="redis://localhost:6379"
```

> 注意：部分功能（如 JWT Secret、Admin API Key）直接使用独立的环境变量，不通过 Figment 配置系统。详见[环境变量完整列表](./环境变量.md)。

## 最小配置

开发环境最小配置：

```toml
[database]
url = "postgres://localhost/univona"
```

其余配置项均有合理的默认值。
