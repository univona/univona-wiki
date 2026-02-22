# Daemon 配置参考

## 概述

Univona Daemon 使用与 Admin Server 相同的配置加载机制（Figment），支持 `univona-daemon.toml` / `Univona.toml` 与环境变量覆盖。Daemon 主要配置集中在数据库连接与日志输出，其他模块（媒体、推送、同步等）在未显式配置时使用默认值。

## 配置文件示例

```toml
# univona-daemon.toml

[database]
url = "postgres://univona:univona_dev@localhost:5432/univona_daemon"
max_connections = 10

[log]
level = "debug"
json = false
```

> 参考：`univona-daemon/univona-daemon.toml.example`

## 配置加载顺序

1. `univona-daemon.toml`
2. `Univona.toml`
3. `UNIVONA_` 前缀环境变量（最高优先级）

## 配置分组

### database

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `url` | String | — | PostgreSQL 连接字符串（必须） |
| `max_connections` | u32 | `10` | 连接池最大连接数 |

### log

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `level` | String | `info` | 日志级别：`trace`/`debug`/`info`/`warn`/`error` |
| `json` | bool | `false` | 是否输出 JSON 日志 |

## 环境变量覆盖示例

```bash
export UNIVONA__DATABASE__URL="postgres://user:pass@localhost/univona_daemon"
export UNIVONA__DATABASE__MAX_CONNECTIONS="20"
export UNIVONA__LOG__LEVEL="info"
export UNIVONA__LOG__JSON="true"
```

## 相关文档

- [Admin Server 配置参考](./Admin-Server配置.md)
- [环境变量完整列表](./环境变量.md)
- [生产部署](../07-部署指南/生产部署.md)
