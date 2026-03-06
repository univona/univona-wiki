# Daemon 用户 API 参考

本文档覆盖 `univona-daemon` 当前实际路由（以 `src/app.rs` 与 `src/api/*.rs` 为准）。

最后核对日期：2026-03-07

## 认证

除公开端点外，所有 Daemon 用户 API 需要用户 JWT：

```http
Authorization: Bearer <user_jwt>
```

## 路由前缀说明

- Daemon 外部 API 前缀固定为 `/api/v1/...`
- 健康检查与 WS 端点为 `/healthz`、`/ws`
- 本文档仅描述 Daemon 仓库实现；Admin-Server 内置用户 API 见 [Admin-Server-API](./Admin-Server-API.md)

## 公开端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/healthz` | 健康检查 |
| GET | `/api/daemon/status` | Daemon 状态 |
| GET | `/api/v1/updates/check` | 客户端更新检查 |
| POST | `/api/v1/auth/challenge` | 请求 challenge |
| POST | `/api/v1/auth/verify` | 验证签名并登录 |
| POST | `/api/v1/auth/register` | 注册用户 |
| POST | `/api/v1/auth/refresh` | 刷新 token |

## 核心用户端点

### PreKey

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/prekeys` |
| GET | `/api/v1/prekeys/{member_id}` |
| DELETE | `/api/v1/prekeys/{device_id}/{key_id}` |
| GET | `/api/v1/prekeys/{device_id}/count` |

### 个人资料与设备

| 方法 | 路径 |
|------|------|
| GET | `/api/v1/members/me` |
| PUT | `/api/v1/members/me` |
| GET | `/api/v1/devices` |
| DELETE | `/api/v1/devices/{device_id}` |

### 频道

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/channels` |
| GET | `/api/v1/channels` |
| GET | `/api/v1/channels/{id}` |
| PUT | `/api/v1/channels/{id}` |
| DELETE | `/api/v1/channels/{id}` |
| PUT | `/api/v1/channels/{id}/encryption-policy` |
| POST | `/api/v1/channels/{id}/members` |
| GET | `/api/v1/channels/{id}/members` |
| DELETE | `/api/v1/channels/{id}/members/{uid}` |

### 消息

| 方法 | 路径 |
|------|------|
| GET | `/api/v1/channels/{id}/messages` |
| GET | `/api/v1/channels/{id}/messages/search` |
| PUT | `/api/v1/messages/{id}` |
| DELETE | `/api/v1/messages/{id}` |
| POST | `/api/v1/messages/{id}/pin` |
| DELETE | `/api/v1/messages/{id}/pin` |
| GET | `/api/v1/channels/{id}/pinned` |
| POST | `/api/v1/messages/{id}/recall` |
| GET | `/api/v1/channels/{id}/mentions` |

### 分类与通知级别

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/categories` |
| GET | `/api/v1/categories` |
| PUT | `/api/v1/categories/{id}` |
| DELETE | `/api/v1/categories/{id}` |
| PUT | `/api/v1/channels/{id}/category` |
| GET | `/api/v1/channels/{id}/notification-level` |
| PUT | `/api/v1/channels/{id}/notification-level` |

### 媒体

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/media/upload` |
| GET | `/api/v1/media/{content_id}` |
| DELETE | `/api/v1/media/{content_id}` |
| GET | `/api/v1/media/{content_id}/thumbnail` |

说明：Daemon 当前没有 `POST /api/v1/media/check-hash`，该端点属于 Admin-Server 内置用户 API。

### 离线消息中继

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/relay/messages` |
| GET | `/api/v1/relay/messages` |
| DELETE | `/api/v1/relay/messages/{message_id}` |

### 推送

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/push/token` |
| DELETE | `/api/v1/push/token` |
| GET | `/api/v1/push/tokens` |

### Bot / Invites

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/bots` |
| GET | `/api/v1/bots` |
| GET | `/api/v1/bots/{id}` |
| PUT | `/api/v1/bots/{id}` |
| DELETE | `/api/v1/bots/{id}` |
| POST | `/api/v1/channels/{id}/invites` |
| GET | `/api/v1/channels/{id}/invites` |
| POST | `/api/v1/invites/{code}/join` |
| DELETE | `/api/v1/invites/{code}` |

说明：`/api/v1/messages/{id}/reactions*` 当前未在 Daemon `src/app.rs` 挂载；该能力在 Admin-Server 内置用户 API 中可用。

### 联系人

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/contacts/lookup` |
| POST | `/api/v1/contacts/requests` |
| GET | `/api/v1/contacts/requests` |
| POST | `/api/v1/contacts/requests/{id}/accept` |
| POST | `/api/v1/contacts/requests/{id}/reject` |
| GET | `/api/v1/contacts` |
| POST | `/api/v1/contacts/presence` |

### 同步

| 方法 | 路径 |
|------|------|
| GET | `/api/v1/sync/cursor/{channel_id}` |
| PUT | `/api/v1/sync/cursor/{channel_id}` |
| GET | `/api/v1/sync/unread/{channel_id}` |

### 语音房间

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/voice/rooms` |
| GET | `/api/v1/voice/rooms/{room_id}` |
| DELETE | `/api/v1/voice/rooms/{room_id}` |
| POST | `/api/v1/voice/rooms/{room_id}/join` |
| POST | `/api/v1/voice/rooms/{room_id}/leave` |
| GET | `/api/v1/voice/rooms/{room_id}/participants` |
| POST | `/api/v1/voice/rooms/{room_id}/mute` |

## 管理端点（Daemon 内）

### 社区管理

| 方法 | 路径 |
|------|------|
| POST | `/api/v1/admin/bans` |
| GET | `/api/v1/admin/bans` |
| DELETE | `/api/v1/admin/bans/{ban_id}` |
| POST | `/api/v1/channels/{id}/mute/{member_id}` |
| DELETE | `/api/v1/channels/{id}/mute/{member_id}` |
| POST | `/api/v1/channels/{id}/kick/{member_id}` |
| PUT | `/api/v1/channels/{id}/members/{member_id}/role` |
| GET | `/api/v1/admin/audit-log` |
| GET | `/api/v1/admin/stats` |
| GET | `/api/v1/admin/members` |

### 内容监控

| 方法 | 路径 |
|------|------|
| GET | `/api/v1/admin/monitor/stats` |
| GET | `/api/v1/admin/monitor/channels` |
| GET | `/api/v1/admin/monitor/channels/{id}/messages` |
| GET | `/api/v1/admin/monitor/channels/{id}/members` |
| GET | `/api/v1/admin/monitor/search` |
| GET | `/api/v1/admin/monitor/relay/stats` |
| GET | `/api/v1/admin/monitor/relay/conversations` |
| GET | `/api/v1/admin/monitor/relay/messages` |

### 权限与发布

| 方法 | 路径 |
|------|------|
| GET | `/api/v1/admin/permissions` |
| PUT | `/api/v1/admin/permissions` |
| DELETE | `/api/v1/admin/permissions/{role}/{permission}` |
| POST | `/api/v1/admin/releases` |
| GET | `/api/v1/admin/releases` |
| DELETE | `/api/v1/admin/releases/{id}` |

## 新增端点详解（本轮核对）

### 1. 撤回消息

`POST /api/v1/messages/{id}/recall`

- 发送者本人可撤回
- 非发送者需为频道 `admin/owner`
- 服务器会把消息标记为 `recalled_at/recalled_by` 并清空 `plaintext_content`

### 2. 频道通知级别

`GET /api/v1/channels/{id}/notification-level`  
`PUT /api/v1/channels/{id}/notification-level`

请求体：

```json
{
  "level": "all"
}
```

可选值：`all` / `mentions_only` / `muted`

### 3. @提及

`GET /api/v1/channels/{id}/mentions`

- 返回当前用户在该频道中被提及记录（最近 50 条）
- 数据来自 `mentions` 表

### 4. 频道分类

`POST/GET /api/v1/categories`  
`PUT/DELETE /api/v1/categories/{id}`  
`PUT /api/v1/channels/{id}/category`

说明：

- 分类 CRUD 需要管理员身份
- 设置频道分类需要该频道 `admin/owner`

## 与旧文档差异（重要）

- PreKey 获取路径是 `/api/v1/prekeys/{member_id}`（不是 `{member_id}/{device_id}`）
- Daemon 当前没有 `/api/v1/channels/unread`、`/api/v1/channels/{id}/read`（这些属于 Admin-Server 内置用户 API）
- Daemon 当前没有 `/api/v1/media/check-hash`
- Daemon 当前没有 `/api/v1/messages/{id}/reactions*`（这些属于 Admin-Server 内置用户 API）
- Daemon 当前没有 `KYC 导出`端点

## 相关文档

- [API 总览](./API总览.md)
- [Admin Server API 参考](./Admin-Server-API.md)
- [WebSocket 协议](./WebSocket协议.md)
