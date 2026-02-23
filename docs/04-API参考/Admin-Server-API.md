# Admin Server API 参考

本文档包含管理面板所有端点的详细参考。

## 认证

除 `/api/v1/admin/login` 外，所有管理面板 API 均需要认证。支持两种方式：

- `Authorization: Bearer <admin_jwt>` -- Admin JWT（推荐）
- `X-Admin-Key: <key>` -- 静态密钥（向后兼容）

---

## 管理员登录

### POST /api/v1/admin/login

管理员登录获取 JWT。此端点有独立的登录速率限制。

**认证**：无需认证

**请求 Body：**

```json
{
  "username": "admin",
  "password": "your_password"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| username | string | 是 | 管理员用户名 |
| password | string | 是 | 管理员密码 |

**成功响应 (200)：**

```json
{
  "success": true,
  "username": "admin",
  "role": "super_admin",
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "expires_at": 1709128800
}
```

**错误响应 (401)：**

```json
{
  "error": "Invalid username or password"
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/admin/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "univona"}'
```

---

## 健康检查

### GET /api/health

深度健康检查，包含数据库连接测试和延迟信息。

**认证**：无需认证（但位于 `/api` 路径下）

**成功响应 (200)：**

```json
{
  "status": "healthy",
  "service": "univona-admin-server",
  "version": "0.1.0",
  "uptime_seconds": 3600,
  "database": {
    "status": "ok",
    "latency_ms": 2,
    "error": null
  }
}
```

**curl 示例：**

```bash
curl http://localhost:3000/api/health
```

### GET /healthz

简单健康检查，用于负载均衡器和 Daemon 兼容。

**成功响应 (200)：**

```json
{
  "status": "ok",
  "service": "univona-admin-server",
  "version": "0.1.0",
  "uptime_seconds": 3600
}
```

### GET /api/metrics

Prometheus 格式指标端点。

**响应**：`text/plain`，Prometheus 格式指标文本。

---

## Daemon 节点管理

### GET /api/v1/admin/daemons

获取 Daemon 节点列表。

**认证**：Admin JWT / X-Admin-Key

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 20 | 每页条数（最大 100） |
| offset | integer | 0 | 偏移量 |
| status | string | -- | 按状态筛选（online/offline/error） |

**成功响应 (200)：**

```json
{
  "daemons": [
    {
      "id": "019f1234-...",
      "name": "主节点",
      "endpoint": "https://daemon1.example.com",
      "status": "online",
      "last_heartbeat": "2024-03-01T12:00:00Z",
      "metadata": {},
      "created_at": "2024-02-01T00:00:00Z",
      "updated_at": "2024-03-01T12:00:00Z"
    }
  ],
  "total": 5,
  "limit": 20,
  "offset": 0
}
```

**curl 示例：**

```bash
curl http://localhost:3000/api/v1/admin/daemons \
  -H "Authorization: Bearer <admin_jwt>"
```

### POST /api/v1/admin/daemons

注册新的 Daemon 节点。

**请求 Body：**

```json
{
  "name": "新节点",
  "endpoint": "https://daemon2.example.com",
  "api_key": "optional_api_key",
  "metadata": {
    "region": "us-west"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | string | 是 | 节点名称 |
| endpoint | string | 是 | 节点 API 地址 |
| api_key | string | 否 | 节点 API 密钥（存储为 SHA256 哈希） |
| metadata | object | 否 | 附加元数据 |

**成功响应 (201)：**

返回创建的 Daemon 对象（同列表中的单个元素格式）。

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/admin/daemons \
  -H "Authorization: Bearer <admin_jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "新节点",
    "endpoint": "https://daemon2.example.com"
  }'
```

### GET /api/v1/admin/daemons/{id}

获取指定 Daemon 节点详情。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | Daemon 节点 ID |

**成功响应 (200)：** 单个 Daemon 对象。

**错误响应 (404)：** `{"error": "Daemon not found"}`

### PUT /api/v1/admin/daemons/{id}

更新 Daemon 节点信息。

**请求 Body：**

```json
{
  "name": "更新后的名称",
  "endpoint": "https://new-endpoint.example.com",
  "api_key": "new_api_key",
  "metadata": {"region": "eu-west"}
}
```

所有字段均为可选，仅更新提供的字段。

**成功响应 (200)：** 更新后的 Daemon 对象。

### DELETE /api/v1/admin/daemons/{id}

删除 Daemon 节点。

**成功响应 (200)：**

```json
{
  "deleted": true,
  "id": "019f1234-..."
}
```

### GET /api/v1/admin/daemons/{id}/health

检查 Daemon 节点健康状态。服务端会主动请求 Daemon 的 `/healthz` 端点（5 秒超时），并根据响应更新数据库中的状态。

**成功响应 (200)：**

```json
{
  "id": "019f1234-...",
  "name": "主节点",
  "status": "online",
  "response_time_ms": 42,
  "daemon_info": {
    "status": "ok",
    "version": "0.1.0"
  }
}
```

### GET /api/v1/admin/daemons/{id}/stats

获取 Daemon 节点统计信息。服务端代理请求到 Daemon 的 `/api/daemon/status` 端点。

**成功响应 (200)：**

```json
{
  "id": "019f1234-...",
  "name": "主节点",
  "stats": {
    "online_members": 42,
    "channels": 10,
    "messages_today": 1500
  }
}
```

**错误响应 (502)：** Daemon 连接失败或返回错误。

---

## Daemon 心跳

### POST /api/v1/daemons/{id}/heartbeat

Daemon 节点心跳上报。此端点无需管理员认证。

**请求 Body：**

```json
{
  "online_members": 42,
  "channel_count": 10,
  "message_count": 15000,
  "version": "0.1.0"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| online_members | integer | 否 | 在线用户数 |
| channel_count | integer | 否 | 频道数 |
| message_count | integer | 否 | 消息总数 |
| version | string | 否 | Daemon 版本 |

所有字段均为可选，仅更新提供的字段。心跳会自动将节点状态设为 `online` 并更新 `last_heartbeat` 时间戳。

**成功响应 (200)：**

```json
{
  "status": "ok"
}
```

---

## KYC 管理

### GET /api/v1/admin/kyc

获取 KYC 验证列表。

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 20 | 每页条数（最大 100） |
| offset | integer | 0 | 偏移量 |
| status | string | -- | 按状态筛选（pending/approved/rejected） |

**成功响应 (200)：**

```json
{
  "verifications": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "provider": "provider_name",
      "external_id": "ext_id",
      "status": "pending",
      "verification_type": "identity",
      "submitted_at": "2024-03-01T00:00:00Z",
      "completed_at": null,
      "expires_at": null,
      "result": null,
      "metadata": {},
      "created_at": "2024-03-01T00:00:00Z",
      "updated_at": "2024-03-01T00:00:00Z"
    }
  ],
  "total": 10,
  "limit": 20,
  "offset": 0
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/kyc?status=pending" \
  -H "Authorization: Bearer <admin_jwt>"
```

### PUT /api/v1/admin/kyc/{id}/approve

审批通过 KYC 验证。仅对 `pending` 状态的记录有效。

**成功响应 (200)：**

```json
{
  "status": "approved",
  "id": "uuid"
}
```

### PUT /api/v1/admin/kyc/{id}/reject

拒绝 KYC 验证。

**请求 Body：**

```json
{
  "reason": "证件照不清晰"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| reason | string | 否 | 拒绝原因 |

**成功响应 (200)：**

```json
{
  "status": "rejected",
  "id": "uuid",
  "reason": "证件照不清晰"
}
```

---

## 社区管理

### GET /api/v1/admin/communities

获取所有社区列表。

**查询参数：** 与 KYC 相同的 `limit`/`offset`/`status` 参数。

**成功响应 (200)：**

```json
{
  "communities": [
    {
      "id": "uuid",
      "name": "社区名称",
      "description": "描述",
      "owner_id": "uuid",
      "endpoint": "https://community.example.com",
      "icon_url": null,
      "banner_url": null,
      "member_count": 150,
      "category": "technology",
      "tags": ["rust", "open-source"],
      "is_verified": false,
      "is_listed": true,
      "status": "active",
      "metadata": {},
      "created_at": "2024-02-01T00:00:00Z",
      "updated_at": "2024-03-01T00:00:00Z"
    }
  ],
  "total": 25,
  "limit": 20,
  "offset": 0
}
```

> 说明：`member_count` 由 Daemon 心跳同步更新（约 60 秒刷新周期），因此新部署或刚加入时可能存在短暂延迟。

### PUT /api/v1/admin/communities/{id}/approve

审批通过社区。将状态设为 `active`，`is_listed` 设为 `true`。

**成功响应 (200)：** `{"status": "active", "id": "uuid"}`

### PUT /api/v1/admin/communities/{id}/reject

拒绝社区申请。

**请求 Body：** `{"reason": "原因"}` （reason 可选）

**成功响应 (200)：** `{"status": "rejected", "id": "uuid", "reason": "..."}`

### PUT /api/v1/admin/communities/{id}/suspend

暂停活跃社区。将状态设为 `suspended`，`is_listed` 设为 `false`。

**成功响应 (200)：** `{"status": "suspended", "id": "uuid"}`

---

## 管理员用户管理

### GET /api/v1/admin/users

获取管理员用户列表。

**成功响应 (200)：**

```json
{
  "admins": [
    {
      "id": "uuid",
      "username": "admin",
      "role": "super_admin",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-03-01T00:00:00Z"
    }
  ],
  "total": 3
}
```

### POST /api/v1/admin/users

创建新管理员用户。

**请求 Body：**

```json
{
  "username": "new_admin",
  "password": "secure_password",
  "role": "admin"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| username | string | 是 | 用户名（唯一） |
| password | string | 是 | 密码 |
| role | string | 否 | 角色（默认 `admin`） |

**成功响应 (201)：**

```json
{
  "id": "uuid",
  "username": "new_admin",
  "role": "admin"
}
```

**错误响应 (409)：** `{"error": "Username already exists"}`

### PUT /api/v1/admin/users/{id}

更新管理员信息。

**请求 Body：**

```json
{
  "password": "new_password",
  "role": "super_admin"
}
```

两个字段都是可选的，但至少需要提供一个。

**成功响应 (200)：**

```json
{
  "id": "uuid",
  "username": "admin",
  "role": "super_admin",
  "updated_at": "2024-03-01T12:00:00Z"
}
```

### DELETE /api/v1/admin/users/{id}

删除管理员用户。

**成功响应 (200)：** `{"deleted": true, "id": "uuid"}`

---

## 举报管理

### POST /api/v1/reports

创建新的举报。

**请求 Body：**

```json
{
  "reporter_id": "uuid",
  "target_type": "message",
  "target_id": "uuid",
  "reason": "spam",
  "description": "此用户发送大量垃圾信息",
  "evidence": [{"url": "screenshot.png"}]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| reporter_id | UUID | 是 | 举报者 ID |
| target_type | string | 是 | 目标类型（message/user/channel） |
| target_id | UUID | 是 | 目标 ID |
| reason | string | 是 | 举报原因 |
| description | string | 否 | 详细描述 |
| evidence | JSON | 否 | 证据材料 |

**成功响应 (201)：** 返回完整的举报记录。

### GET /api/v1/reports

获取举报列表。

**查询参数：** `status`/`limit`/`offset`

**成功响应 (200)：**

```json
{
  "reports": [...],
  "total": 50,
  "limit": 20,
  "offset": 0
}
```

### PUT /api/v1/reports/{id}

审核举报。

**请求 Body：**

```json
{
  "reviewed_by": "admin_uuid",
  "status": "resolved",
  "resolution": "已封禁违规用户"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| reviewed_by | UUID | 是 | 审核人 ID |
| status | string | 是 | 状态：reviewed/resolved/dismissed/escalated |
| resolution | string | 否 | 处理结果 |

**成功响应 (200)：** 返回更新后的举报记录。

---

## 审计日志

### GET /api/v1/admin/audit-logs

获取管理操作审计日志。

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 50 | 每页条数（最大 200） |
| offset | integer | 0 | 偏移量 |
| action | string | -- | 按操作类型筛选 |

**成功响应 (200)：**

```json
{
  "logs": [
    {
      "id": 1,
      "admin_id": "uuid",
      "action": "kyc_approve",
      "target_type": "kyc_verification",
      "target_id": "uuid",
      "details": {"reason": "符合要求"},
      "ip_address": "192.168.1.1",
      "created_at": "2024-03-01T12:00:00Z"
    }
  ],
  "total": 100,
  "limit": 50,
  "offset": 0
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/audit-logs?limit=10&action=kyc_approve" \
  -H "Authorization: Bearer <admin_jwt>"
```

---

## 系统信息

### GET /api/v1/admin/system-info

获取系统运行状态和统计信息。

**成功响应 (200)：**

```json
{
  "version": "0.1.0",
  "uptime_seconds": 86400,
  "uptime_display": "1天 0小时 0分钟",
  "database": {
    "pool_size": 10,
    "pool_idle": 7,
    "max_connections": 20
  },
  "counts": {
    "communities": 25,
    "active_communities": 20,
    "daemon_nodes": 5,
    "online_daemons": 3,
    "admins": 3,
    "kyc_verifications": 100,
    "abuse_reports": 50,
    "members": 1000,
    "channels": 200,
    "audit_logs": 500
  },
  "config": {
    "host": "0.0.0.0",
    "port": 3000,
    "log_level": "info"
  }
}
```

**curl 示例：**

```bash
curl http://localhost:3000/api/v1/admin/system-info \
  -H "Authorization: Bearer <admin_jwt>"
```

---

## 内容监听 (Content Monitor)

内容监听模块允许管理员查看 Daemon 节点上的群聊消息和私聊中继消息。仅监控 `plaintext_content` 非空的消息，E2E 加密的内容无法查看。

所有内容监听端点均需要 Admin JWT 或 X-Admin-Key 认证。

### 群聊监控

#### GET /api/v1/admin/monitor/stats

获取内容监控统计概览。

**认证**：Admin JWT / X-Admin-Key

**成功响应 (200)：**

```json
{
  "total_channels": 50,
  "total_messages": 500000,
  "viewable_messages": 350000,
  "encrypted_messages": 150000,
  "today_messages": 1200,
  "active_channels_today": 15,
  "total_members": 1000
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| total_channels | integer | 频道总数 |
| total_messages | integer | 消息总数（不含已删除） |
| viewable_messages | integer | 可查看消息数（plaintext_content 非空） |
| encrypted_messages | integer | 加密消息数（无法查看） |
| today_messages | integer | 今日消息数 |
| active_channels_today | integer | 今日活跃频道数 |
| total_members | integer | 成员总数 |

**curl 示例：**

```bash
curl http://localhost:3000/api/v1/admin/monitor/stats \
  -H "Authorization: Bearer <admin_jwt>"
```

#### GET /api/v1/admin/monitor/channels

获取频道列表（带消息统计）。按最新消息时间降序排列。

**认证**：Admin JWT / X-Admin-Key

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 20 | 每页条数（最大 100） |
| offset | integer | 0 | 偏移量 |
| search | string | -- | 频道名 ILIKE 模糊搜索 |
| channel_type | string | -- | 按频道类型筛选（group/public/private） |

**成功响应 (200)：**

```json
{
  "channels": [
    {
      "id": "uuid",
      "name": "一般讨论",
      "description": "公开讨论频道",
      "channel_type": "public",
      "member_count": 150,
      "total_messages": 12000,
      "viewable_messages": 10000,
      "latest_message_at": "2024-03-01T12:30:00Z",
      "created_at": "2024-01-15T10:00:00Z"
    }
  ],
  "total": 50,
  "limit": 20,
  "offset": 0
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/channels?limit=10&search=讨论" \
  -H "Authorization: Bearer <admin_jwt>"
```

#### GET /api/v1/admin/monitor/channels/{id}/messages

获取指定频道的消息列表。仅返回 `plaintext_content` 非空且未删除的消息。

**认证**：Admin JWT / X-Admin-Key

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | 频道 ID |

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 50 | 每页条数（最大 100） |
| offset | integer | 0 | 偏移量 |
| search | string | -- | 消息内容 ILIKE 模糊搜索 |
| start_time | string | -- | 起始时间（ISO 8601，如 `2024-03-01T00:00:00Z`） |
| end_time | string | -- | 结束时间（ISO 8601） |

**成功响应 (200)：**

```json
{
  "channel": {
    "id": "uuid",
    "name": "一般讨论",
    "channel_type": "public",
    "member_count": 150
  },
  "messages": [
    {
      "id": "uuid",
      "sender_id": "uuid",
      "sender_name": "alice",
      "sender_display_name": "Alice",
      "plaintext_content": "大家好！",
      "message_type": "text",
      "is_encrypted": false,
      "server_timestamp": "2024-03-01T12:30:00Z",
      "client_timestamp": "2024-03-01T12:29:59Z",
      "reply_to": null,
      "edited_at": null
    }
  ],
  "total": 10000,
  "limit": 50,
  "offset": 0
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/channels/{channel_id}/messages?limit=20&search=你好" \
  -H "Authorization: Bearer <admin_jwt>"
```

#### GET /api/v1/admin/monitor/channels/{id}/members

获取指定频道的成员列表。

**认证**：Admin JWT / X-Admin-Key

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | 频道 ID |

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 50 | 每页条数（最大 100） |
| offset | integer | 0 | 偏移量 |

**成功响应 (200)：**

```json
{
  "members": [
    {
      "member_id": "uuid",
      "username": "alice",
      "display_name": "Alice",
      "role": "admin",
      "joined_at": "2024-01-15T10:00:00Z"
    }
  ],
  "total": 150
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/channels/{channel_id}/members" \
  -H "Authorization: Bearer <admin_jwt>"
```

#### GET /api/v1/admin/monitor/search

全局消息搜索。在所有频道中搜索包含指定关键词的消息。

**认证**：Admin JWT / X-Admin-Key

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| keyword | string | -- | **必填**，搜索关键词（1-200 字符） |
| limit | integer | 20 | 每页条数（最大 50） |
| offset | integer | 0 | 偏移量 |
| channel_id | UUID | -- | 限定搜索的频道 ID |
| start_time | string | -- | 起始时间（ISO 8601） |
| end_time | string | -- | 结束时间（ISO 8601） |

**成功响应 (200)：**

```json
{
  "results": [
    {
      "message_id": "uuid",
      "channel_id": "uuid",
      "channel_name": "一般讨论",
      "sender_id": "uuid",
      "sender_name": "alice",
      "sender_display_name": "Alice",
      "plaintext_content": "今天的讨论主题是...",
      "message_type": "text",
      "server_timestamp": "2024-03-01T12:30:00Z"
    }
  ],
  "total": 42,
  "limit": 20,
  "offset": 0
}
```

**错误响应 (400)：** `keyword` 缺失或超过 200 字符。

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/search?keyword=讨论&limit=10" \
  -H "Authorization: Bearer <admin_jwt>"
```

### 私聊中继监控

私聊中继监控用于查看通过 `offline_messages` 表中转的一对一消息。加密类型（`content_type = 'encrypted'`）的消息内容不会展示，仅显示元数据。

#### GET /api/v1/admin/monitor/relay/stats

获取私聊中继消息统计。

**认证**：Admin JWT / X-Admin-Key

**成功响应 (200)：**

```json
{
  "total_messages": 10000,
  "encrypted_messages": 8000,
  "plaintext_messages": 2000,
  "today_messages": 150,
  "unique_senders": 200,
  "unique_recipients": 250,
  "total_size_bytes": 52428800
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| total_messages | integer | 中继消息总数 |
| encrypted_messages | integer | 加密消息数 |
| plaintext_messages | integer | 明文消息数 |
| today_messages | integer | 今日消息数 |
| unique_senders | integer | 不同发送者数 |
| unique_recipients | integer | 不同接收者数 |
| total_size_bytes | integer | 消息总大小（字节） |

**curl 示例：**

```bash
curl http://localhost:3000/api/v1/admin/monitor/relay/stats \
  -H "Authorization: Bearer <admin_jwt>"
```

#### GET /api/v1/admin/monitor/relay/conversations

获取中继对话列表。按双方用户 ID 聚合，显示每对用户间的消息统计。

**认证**：Admin JWT / X-Admin-Key

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 20 | 每页条数（最大 100） |
| offset | integer | 0 | 偏移量 |

**成功响应 (200)：**

```json
{
  "conversations": [
    {
      "user_a_id": "uuid",
      "user_a_name": "alice",
      "user_a_display_name": "Alice",
      "user_b_id": "uuid",
      "user_b_name": "bob",
      "user_b_display_name": "Bob",
      "message_count": 120,
      "encrypted_count": 100,
      "plaintext_count": 20,
      "last_message_at": "2024-03-01T14:00:00Z"
    }
  ],
  "total": 85,
  "limit": 20,
  "offset": 0
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/relay/conversations?limit=10" \
  -H "Authorization: Bearer <admin_jwt>"
```

#### GET /api/v1/admin/monitor/relay/messages

获取中继消息列表。支持按发送者、接收者、内容类型和时间范围筛选。当同时提供 `sender_id` 和 `recipient_id` 时，查询该对用户之间的双向消息。

**认证**：Admin JWT / X-Admin-Key

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 50 | 每页条数（最大 100） |
| offset | integer | 0 | 偏移量 |
| sender_id | UUID | -- | 按发送者筛选 |
| recipient_id | UUID | -- | 按接收者筛选 |
| content_type | string | -- | 按内容类型筛选（encrypted/text 等） |
| start_time | string | -- | 起始时间（ISO 8601） |
| end_time | string | -- | 结束时间（ISO 8601） |

**成功响应 (200)：**

```json
{
  "messages": [
    {
      "id": "uuid",
      "sender_id": "uuid",
      "sender_name": "alice",
      "sender_display_name": "Alice",
      "recipient_id": "uuid",
      "recipient_name": "bob",
      "recipient_display_name": "Bob",
      "content_type": "text",
      "plaintext_preview": "你好，在吗？",
      "is_encrypted": false,
      "size_bytes": 42,
      "created_at": "2024-03-01T14:00:00Z",
      "expires_at": "2024-03-08T14:00:00Z"
    }
  ],
  "total": 120,
  "limit": 50,
  "offset": 0
}
```

> **注意**：`is_encrypted` 为 `true` 时，`plaintext_preview` 为 `null`，加密内容不会展示。

**curl 示例：**

```bash
# 查看两个用户之间的对话
curl "http://localhost:3000/api/v1/admin/monitor/relay/messages?sender_id={user_a_id}&recipient_id={user_b_id}" \
  -H "Authorization: Bearer <admin_jwt>"

# 按内容类型筛选
curl "http://localhost:3000/api/v1/admin/monitor/relay/messages?content_type=text&limit=20" \
  -H "Authorization: Bearer <admin_jwt>"
```

---

## 权限管理

权限管理 API 允许管理员查看和修改社区角色的权限配置。通过 `permission_overrides` 表覆盖默认的角色权限。

所有权限管理端点均需要 Admin JWT 或 X-Admin-Key 认证。

### GET /api/v1/admin/permissions

获取完整权限矩阵，包含所有角色和权限的当前状态。

**认证**：Admin JWT / X-Admin-Key

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "permissions": [
      { "key": "send_message", "category": "message" },
      { "key": "delete_own_message", "category": "message" },
      { "key": "delete_any_message", "category": "message" },
      { "key": "pin_message", "category": "message" },
      { "key": "create_channel", "category": "channel" },
      { "key": "update_channel", "category": "channel" },
      { "key": "delete_channel", "category": "channel" },
      { "key": "view_channel", "category": "channel" },
      { "key": "invite_member", "category": "member" },
      { "key": "kick_member", "category": "member" },
      { "key": "ban_member", "category": "member" },
      { "key": "manage_roles", "category": "member" },
      { "key": "upload_media", "category": "media" },
      { "key": "manage_media", "category": "media" },
      { "key": "view_audit_log", "category": "admin" },
      { "key": "manage_server", "category": "admin" },
      { "key": "manage_channel_encryption", "category": "admin" },
      { "key": "view_content_monitor", "category": "admin" }
    ],
    "roles": ["guest", "member", "moderator", "admin", "owner"],
    "matrix": {
      "guest": {
        "send_message": { "granted": false, "is_override": false },
        "view_channel": { "granted": true, "is_override": false }
      },
      "member": {
        "send_message": { "granted": true, "is_override": false },
        "view_channel": { "granted": true, "is_override": false }
      },
      "moderator": {
        "kick_member": { "granted": true, "is_override": false },
        "view_audit_log": { "granted": false, "is_override": true }
      },
      "owner": {
        "manage_server": { "granted": true, "is_override": false }
      }
    }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| permissions | array | 所有权限列表，含 key 和 category |
| roles | array | 所有角色列表（按等级排列） |
| matrix | object | 角色 → 权限 → {granted, is_override} 矩阵 |
| matrix.*.granted | boolean | 当前是否授予该权限 |
| matrix.*.is_override | boolean | 是否为覆盖值（false 表示默认值） |

**curl 示例：**

```bash
curl http://localhost:3000/api/v1/admin/permissions \
  -H "Authorization: Bearer <admin_jwt>"
```

### PUT /api/v1/admin/permissions

批量更新权限覆盖。Owner 角色的权限不可修改（会被跳过）。

**认证**：Admin JWT / X-Admin-Key

**请求 Body：**

```json
{
  "overrides": [
    { "role": "moderator", "permission": "view_audit_log", "granted": true },
    { "role": "guest", "permission": "send_message", "granted": true },
    { "role": "member", "permission": "create_channel", "granted": true }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| overrides | array | 是 | 覆盖项列表 |
| overrides[].role | string | 是 | 角色名称（guest/member/moderator/admin） |
| overrides[].permission | string | 是 | 权限名称 |
| overrides[].granted | boolean | 是 | 是否授予 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "updated": 3,
    "skipped": 0
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| updated | integer | 成功更新的覆盖数 |
| skipped | integer | 跳过的数量（无效角色/权限或 Owner 角色） |

**curl 示例：**

```bash
curl -X PUT http://localhost:3000/api/v1/admin/permissions \
  -H "Authorization: Bearer <admin_jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "overrides": [
      {"role": "moderator", "permission": "view_audit_log", "granted": true}
    ]
  }'
```

### DELETE /api/v1/admin/permissions/{role}/{permission}

删除指定角色的权限覆盖，恢复为默认权限。

**认证**：Admin JWT / X-Admin-Key

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| role | string | 角色名称（guest/member/moderator/admin/owner） |
| permission | string | 权限名称 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| deleted | boolean | 是否成功删除（false 表示覆盖不存在） |

**curl 示例：**

```bash
curl -X DELETE http://localhost:3000/api/v1/admin/permissions/moderator/view_audit_log \
  -H "Authorization: Bearer <admin_jwt>"
```
