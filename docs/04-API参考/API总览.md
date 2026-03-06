# API 总览

本文档汇总 Univona 当前三类 API 面：

1. `univona-admin-server` 的管理 API
2. `univona-admin-server` 的内置用户 API（嵌套在 `/api/v1`）
3. `univona-daemon` 的用户/社区 API（前缀 `/api/v1`）

最后核对日期：2026-03-07

## 认证方式

| 方式 | 适用场景 | Header |
|------|----------|--------|
| Admin JWT | 管理面板操作 | `Authorization: Bearer <admin_jwt>` |
| X-Admin-Key | 管理 API 兼容认证 | `X-Admin-Key: <key>` |
| 用户 JWT | 客户端 API | `Authorization: Bearer <user_jwt>` |
| 无认证 | 健康检查、挑战认证、更新检查 | -- |

## 管理 API（Admin-Server）

前缀：`/api/v1/admin/...`

核心模块：

| 模块 | 代表路径 |
|------|----------|
| 管理员认证 | `POST /api/v1/admin/login` |
| Daemon 节点管理 | `/api/v1/admin/daemons` |
| 社区审核与目录管理 | `/api/v1/admin/communities` |
| KYC 审核 | `/api/v1/admin/kyc` |
| 权限矩阵 | `/api/v1/admin/permissions` |
| 内容监控 | `/api/v1/admin/monitor/...` |
| 发布管理 | `/api/v1/admin/releases` |
| 审计/系统信息 | `/api/v1/admin/audit-logs`、`/api/v1/admin/system-info` |

附加公开管理接口：

| 方法 | 路径 |
|------|------|
| GET | `/api/health` |
| GET | `/api/metrics` |
| GET | `/healthz` |

## 内置用户 API（Admin-Server）

Admin-Server 把 Daemon 用户能力内置在同一服务中，对外统一挂在 `/api/v1/...`。

与本轮迭代直接相关的新增端点：

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/messages/{id}/recall` | 撤回消息 |
| GET | `/api/v1/channels/{id}/notification-level` | 获取通知级别 |
| PUT | `/api/v1/channels/{id}/notification-level` | 设置通知级别 |
| GET | `/api/v1/channels/{id}/mentions` | 获取 @mentions |
| POST | `/api/v1/categories` | 创建分类 |
| GET | `/api/v1/categories` | 列出分类 |
| PUT | `/api/v1/categories/{id}` | 更新分类 |
| DELETE | `/api/v1/categories/{id}` | 删除分类 |
| PUT | `/api/v1/channels/{id}/category` | 设置频道分类 |
| POST | `/api/v1/user/export` | GDPR 数据导出 |
| DELETE | `/api/v1/user` | 请求删除账号 |
| POST | `/api/v1/user/cancel-deletion` | 取消删除请求 |

此外还包括：`/api/v1/prekeys/*`、`/api/v1/channels/*`、`/api/v1/messages/*`、`/api/v1/media/*`、`/api/v1/relay/*`、`/api/v1/p2p/*`、`/api/v1/devices/*`、`/api/v1/sync`、`/api/v1/updates/check` 等。

## Daemon API（独立部署）

前缀：`/api/v1/...`

典型能力：

| 模块 | 代表路径 |
|------|----------|
| 认证 | `/api/v1/auth/challenge` |
| 频道与消息 | `/api/v1/channels/*`、`/api/v1/messages/*` |
| 媒体 | `/api/v1/media/upload` |
| 离线中继 | `/api/v1/relay/messages` |
| 联系人 | `/api/v1/contacts/*` |
| 语音房间 | `/api/v1/voice/rooms/*` |
| 增量同步 | `/api/v1/sync/cursor/{channel_id}` |
| 设备管理 | `/api/v1/devices` |
| 版本更新 | `/api/v1/updates/check` |

与本轮迭代直接相关的新增端点（Daemon）：

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/messages/{id}/recall` |
| GET | `/api/v1/channels/{id}/notification-level` |
| PUT | `/api/v1/channels/{id}/notification-level` |
| GET | `/api/v1/channels/{id}/mentions` |
| POST/GET | `/api/v1/categories` |
| PUT/DELETE | `/api/v1/categories/{id}` |
| PUT | `/api/v1/channels/{id}/category` |

## WebSocket

| 服务 | 路径 | 认证 |
|------|------|------|
| Admin-Server | `/ws?token=<user_jwt>` | 用户 JWT |
| Daemon | `/ws?token=<user_jwt>` | 用户 JWT |

实时消息主协议为 Protobuf `Envelope`，部分系统事件（如消息撤回）会以 JSON 事件帧形式广播。

## 常见混淆点

- Admin-Server 代码里常写 `/v1/...`，但外部实际路径是 `/api/v1/...`（因为在 `app.rs` 里统一 `nest("/api", ...)`）。
- Daemon 的 `PreKey` 获取路径是 `/api/v1/prekeys/{member_id}`。
- Daemon 没有 `POST /api/v1/media/check-hash`，该端点仅在 Admin-Server 内置用户 API 中存在。

## 相关文档

- [Daemon 用户 API 参考](./Daemon-API.md)
- [Admin Server API 参考](./Admin-Server-API.md)
- [WebSocket 协议](./WebSocket协议.md)
