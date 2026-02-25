# Daemon 用户 API 参考

本文档包含所有 Daemon 用户端点的详细参考。

## 认证

除认证端点外，所有 Daemon 用户 API 均需要用户 JWT。

```
Authorization: Bearer <user_jwt>
```

JWT 通过 [认证流程](./认证流程.md) 中的 Challenge-Response 或注册流程获取。

---

## 用户认证

### POST /api/v1/auth/challenge

请求认证挑战。返回一个 32 字节随机 nonce，有效期 5 分钟。

**认证**：无需认证

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "challenge_id": "uuid",
    "nonce": "<64字符hex编码的32字节随机数>",
    "expires_at": 1709100300
  }
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/auth/challenge
```

### POST /api/v1/auth/verify

使用 Ed25519 签名验证挑战响应，获取令牌。

**认证**：无需认证

**请求 Body：**

```json
{
  "challenge_id": "uuid",
  "identity_key": "<Ed25519公钥hex>",
  "signature": "<签名hex>",
  "device_id": "device_uuid"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| challenge_id | string | 是 | 挑战 ID |
| identity_key | string | 是 | Ed25519 公钥的 hex 编码 |
| signature | string | 是 | 对 nonce 的 Ed25519 签名的 hex 编码 |
| device_id | string | 是 | 设备 ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "expires_at": 1709103600,
    "member_id": "uuid"
  }
}
```

### POST /api/v1/auth/register

注册新用户。

**认证**：无需认证

**请求 Body：**

```json
{
  "username": "alice",
  "identity_key": "<32字节Ed25519公钥的hex编码>",
  "display_name": "Alice",
  "device": {
    "device_id": "uuid",
    "device_name": "iPhone 15",
    "platform": "ios",
    "device_identity_key": "<hex>",
    "signed_prekey": "<hex>",
    "signed_prekey_signature": "<hex>",
    "signed_prekey_id": 1
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| username | string | 是 | 用户名（需通过验证规则） |
| identity_key | string | 是 | Ed25519 公钥 hex（必须 32 字节） |
| display_name | string | 否 | 显示名称 |
| device | object | 是 | 设备信息（见下方） |

**device 对象：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| device_id | string | 是 | 设备 UUID |
| device_name | string | 否 | 设备名称 |
| platform | string | 否 | 平台（ios/android/desktop/web） |
| device_identity_key | string | 是 | 设备身份密钥 hex |
| signed_prekey | string | 是 | 签名 PreKey hex |
| signed_prekey_signature | string | 是 | 签名 PreKey 的签名 hex |
| signed_prekey_id | integer | 是 | 签名 PreKey ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "expires_at": 1709103600,
    "member_id": "uuid"
  }
}
```

**错误响应 (409)：** 用户名或 identity_key 已注册

### POST /api/v1/auth/refresh

刷新访问令牌。

**认证**：无需认证

**请求 Body：**

```json
{
  "refresh_token": "<refresh_token>"
}
```

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "access_token": "<new_access_token>",
    "refresh_token": "<new_refresh_token>",
    "expires_at": 1709107200,
    "member_id": "uuid"
  }
}
```

---

## PreKey 管理

Signal 协议的 X3DH 密钥交换需要 PreKey。

### POST /api/v1/prekeys

上传 PreKey Bundle（签名 PreKey + 一次性 PreKey）。

**请求 Body：**

```json
{
  "signed_prekey": "<hex编码的签名PreKey>",
  "signed_prekey_signature": "<hex编码的签名>",
  "signed_prekey_id": 5,
  "one_time_prekeys": [
    {"id": 100, "public_key": "<hex>"},
    {"id": 101, "public_key": "<hex>"},
    {"id": 102, "public_key": "<hex>"}
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| signed_prekey | string | 是 | 签名 PreKey 的 hex 编码 |
| signed_prekey_signature | string | 是 | 对签名 PreKey 的签名 hex |
| signed_prekey_id | integer | 是 | 签名 PreKey ID |
| one_time_prekeys | array | 是 | 一次性 PreKey 列表 |

**one_time_prekeys 元素：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | integer | PreKey ID |
| public_key | string | 公钥 hex 编码 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "uploaded": 3
  }
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/prekeys \
  -H "Authorization: Bearer <user_jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "signed_prekey": "abcd1234...",
    "signed_prekey_signature": "ef567890...",
    "signed_prekey_id": 5,
    "one_time_prekeys": [
      {"id": 100, "public_key": "aabb..."},
      {"id": 101, "public_key": "ccdd..."}
    ]
  }'
```

### GET /api/v1/prekeys/{member_id}/{device_id}

获取指定成员的 PreKey Bundle。会消耗一个一次性 PreKey（如果有的话）。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| member_id | UUID | 目标成员 ID |
| device_id | UUID | 目标设备 ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "identity_key": "<hex>",
      "signed_prekey": "<hex>",
      "signed_prekey_signature": "<hex>",
      "signed_prekey_id": 5,
      "one_time_prekey": "<hex>",
      "one_time_prekey_id": 100,
      "device_id": "uuid"
    }
  ]
}
```

> 返回的是该成员所有设备的 PreKey Bundle 数组。`one_time_prekey` 和 `one_time_prekey_id` 可能为 `null`（当 OPK 已耗尽时）。

### DELETE /api/v1/prekeys/{device_id}/{key_id}

删除指定的一次性 PreKey。只能删除属于自己设备的 PreKey。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| device_id | UUID | 设备 ID |
| key_id | integer | PreKey ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {"deleted": true}
}
```

### GET /api/v1/prekeys/{device_id}/count

获取指定设备的剩余 PreKey 数量。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "count": 25,
    "threshold": 10,
    "needs_replenish": false
  }
}
```

> 当 `count` 低于 `threshold`（10）时，`needs_replenish` 为 `true`，客户端应上传新的一次性 PreKey。

---

## 频道管理

### POST /api/v1/channels

创建新频道。创建者自动成为频道 Owner。

**请求 Body：**

```json
{
  "name": "技术交流群",
  "description": "讨论技术话题",
  "channel_type": "public",
  "avatar_url": "https://example.com/avatar.png",
  "encryption_policy": "optional"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | string | 是 | 频道名称（需通过验证） |
| description | string | 否 | 频道描述 |
| channel_type | string | 是 | 类型：public/private/direct |
| avatar_url | string | 否 | 头像 URL |
| encryption_policy | string | 否 | 加密策略：optional(默认)/preferred/required |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "技术交流群",
    "description": "讨论技术话题",
    "channel_type": "public",
    "avatar_url": "https://example.com/avatar.png",
    "created_by": "member_uuid",
    "member_count": 1,
    "encryption_policy": "optional",
    "created_at": 1709100000000,
    "updated_at": 1709100000000
  }
}
```

### GET /api/v1/channels

列出当前用户已加入的所有频道。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "技术交流群",
      "description": "...",
      "channel_type": "public",
      "avatar_url": null,
      "created_by": "uuid",
      "member_count": 0,
      "encryption_policy": "optional",
      "created_at": 1709100000000,
      "updated_at": 1709100000000
    }
  ]
}
```

### GET /api/v1/channels/{id}

获取频道详情（含成员数）。

**成功响应 (200)：** 单个频道对象。

### PUT /api/v1/channels/{id}

更新频道信息。需要 `UpdateChannel` 权限。

**请求 Body：**

```json
{
  "name": "新名称",
  "description": "新描述",
  "channel_type": "private",
  "avatar_url": "https://example.com/new-avatar.png"
}
```

所有字段均为可选。

**成功响应 (200)：** `{"success": true, "data": {"updated": true}}`

### DELETE /api/v1/channels/{id}

删除频道。需要 `DeleteChannel` 权限。

**成功响应 (200)：** `{"success": true, "data": {"deleted": true}}`

### PUT /api/v1/channels/{id}/encryption-policy

更新频道加密策略。需要 `ManageChannelEncryption` 权限（Admin 或 Owner）。

**请求 Body：**

```json
{
  "encryption_policy": "required"
}
```

| 值 | 说明 |
|----|------|
| optional | 可选加密（默认） |
| preferred | 推荐加密 |
| required | 强制加密 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "updated": true,
    "encryption_policy": "required"
  }
}
```

### PUT /api/v1/channels/{id}/history-visible

更新频道历史消息可见性。启用后，新加入的成员可以查看加入前的历史消息；禁用后，新成员只能看到加入后的消息。需要 `UpdateChannel` 权限（Admin 或 Owner）。

**请求 Body：**

```json
{
  "history_visible": true
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| history_visible | boolean | 是 | 是否允许新成员查看历史消息 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "updated": true,
    "history_visible": true
  }
}
```

**curl 示例：**

```bash
curl -X PUT http://localhost:3000/api/v1/channels/{channel_id}/history-visible \
  -H "Authorization: Bearer <user_jwt>" \
  -H "Content-Type: application/json" \
  -d '{"history_visible": false}'
```

### POST /api/v1/channels/{id}/members

添加频道成员。需要 `InviteMember` 权限。

**请求 Body：**

```json
{
  "member_id": "uuid",
  "role": "member"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| member_id | string | 是 | 要添加的成员 ID |
| role | string | 否 | 角色（默认 `member`） |

**成功响应 (200)：** `{"success": true, "data": {"added": true}}`

### GET /api/v1/channels/{id}/members

列出频道所有成员。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "member_id": "uuid",
      "username": "alice",
      "display_name": "Alice",
      "role": "owner",
      "joined_at": 1709100000000
    }
  ]
}
```

### DELETE /api/v1/channels/{id}/members/{member_id}

移除频道成员。需要 `KickMember` 权限。

**成功响应 (200)：** `{"success": true, "data": {"removed": true}}`

---

## 消息

### GET /api/v1/channels/{id}/messages

获取频道消息（游标分页，按时间倒序）。

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| before | string | -- | 游标：返回此消息 ID 之前的消息 |
| limit | integer | 50 | 每页条数（最大 100） |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "channel_id": "uuid",
      "sender_id": "uuid",
      "message_type": "1",
      "encrypted_content": [222, 173, ...],
      "is_encrypted": true,
      "plaintext_content": null,
      "server_timestamp": 1709100000000,
      "client_timestamp": 1709099999000,
      "reply_to": null,
      "created_at": 1709100000000
    }
  ]
}
```

**curl 示例：**

```bash
# 获取最新 20 条消息
curl "http://localhost:3000/api/v1/channels/{channel_id}/messages?limit=20" \
  -H "Authorization: Bearer <user_jwt>"

# 获取指定消息之前的 20 条消息（翻页）
curl "http://localhost:3000/api/v1/channels/{channel_id}/messages?before={msg_id}&limit=20" \
  -H "Authorization: Bearer <user_jwt>"
```

### PUT /api/v1/messages/{id}

编辑消息。只有消息发送者本人可以编辑。编辑后会通过 WebSocket 广播 `MESSAGE_EDITED` 事件。

**请求 Body：**

```json
{
  "encrypted_content": [222, 173, ...],
  "plaintext_content": "编辑后的内容"
}
```

**成功响应 (200)：** `{"success": true, "data": null}`

### DELETE /api/v1/messages/{id}

删除消息（软删除）。发送者本人或频道管理员（admin/owner）可以删除。删除后会通过 WebSocket 广播 `MESSAGE_DELETED` 事件。

**成功响应 (200)：** `{"success": true, "data": null}`

### POST /api/v1/messages/{id}/pin

置顶消息。需要频道 admin 或 owner 权限。会通过 WebSocket 广播 `MESSAGE_PINNED` 事件。

**成功响应 (200)：** `{"success": true, "data": null}`

### DELETE /api/v1/messages/{id}/pin

取消置顶消息。需要频道 admin 或 owner 权限。会通过 WebSocket 广播 `MESSAGE_UNPINNED` 事件。

**成功响应 (200)：** `{"success": true, "data": null}`

### GET /api/v1/channels/{id}/pinned

获取频道置顶消息列表。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "message_id": "uuid",
      "channel_id": "uuid",
      "pinned_by": "uuid",
      "pinned_at": 1709100000000
    }
  ]
}
```

---

## 媒体文件

### POST /api/v1/media/check-hash

检查文件 SHA-256 哈希是否已存在（客户端去重）。如果文件已存在，会自动增加引用计数并返回已有文件信息，无需重复上传。

**请求 Body：**

```json
{
  "sha256_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| sha256_hash | string | 是 | 文件的 SHA-256 哈希值（64 位 hex 编码） |

**成功响应 (200)：**

文件已存在：

```json
{
  "success": true,
  "data": {
    "exists": true,
    "content_id": "uuid",
    "file_name": "photo.jpg",
    "mime_type": "image/jpeg",
    "file_size": 1048576
  }
}
```

文件不存在：

```json
{
  "success": true,
  "data": {
    "exists": false,
    "content_id": null,
    "file_name": null,
    "mime_type": null,
    "file_size": null
  }
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/media/check-hash \
  -H "Authorization: Bearer <user_jwt>" \
  -H "Content-Type: application/json" \
  -d '{"sha256_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"}'
```

### POST /api/v1/media/upload

上传媒体文件。使用 `multipart/form-data` 格式，最大 50MB。

**请求格式**：`multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file | binary | 是 | 文件数据 |
| file_name | string | 否 | 文件名（也可通过 multipart header 获取） |
| mime_type | string | 否 | MIME 类型（也可通过 multipart header 获取） |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "content_id": "uuid",
    "file_name": "photo.jpg",
    "mime_type": "image/jpeg",
    "file_size": 1048576,
    "has_thumbnail": true
  }
}
```

> 图片文件会自动生成 200x200 缩略图。

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/media/upload \
  -H "Authorization: Bearer <user_jwt>" \
  -F "file=@photo.jpg"
```

### GET /api/v1/media/{content_id}

下载媒体文件。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| content_id | string | 媒体内容 ID |

**成功响应 (200)**：二进制文件流，带有 `Content-Type` 和 `Content-Disposition` 头。

### GET /api/v1/media/{content_id}/thumbnail

下载缩略图。仅图片文件有缩略图。

**成功响应 (200)**：JPEG 格式的缩略图。

**错误响应 (404)：** 缩略图不存在

### DELETE /api/v1/media/{content_id}

删除媒体文件。只有上传者本人可以删除。会同时删除缩略图和数据库记录。

**成功响应 (200)：** `{"success": true, "data": null}`

---

## 离线消息中继

### POST /api/v1/relay/messages

上传离线消息到中继队列。

**请求 Body：**

```json
{
  "recipient_id": "uuid",
  "encrypted_payload": [222, 173, ...],
  "message_id": "uuid"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| recipient_id | string | 是 | 接收者 ID |
| encrypted_payload | bytes | 是 | 加密载荷（不超过 1MB） |
| message_id | string | 是 | 消息唯一 ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "stored": true
  }
}
```

**错误响应 (429)：** 配额超限

### GET /api/v1/relay/messages

拉取离线消息。最多返回 100 条。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "sender_id": "uuid",
      "message_id": "msg_uuid",
      "encrypted_payload": [222, 173, ...],
      "created_at": 1709100000000
    }
  ]
}
```

### DELETE /api/v1/relay/messages/{message_id}

确认离线消息已接收，从队列中移除。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| message_id | string | 消息 ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

---

## 推送令牌

### POST /api/v1/push/token

注册或更新推送令牌。

**请求 Body：**

```json
{
  "platform": "fcm",
  "token": "firebase_token_string",
  "device_id": "uuid"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| platform | string | 是 | 推送平台：`fcm`/`apns`/`unified_push` |
| token | string | 是 | 推送令牌（不能为空） |
| device_id | string | 是 | 设备 UUID |

> 同一用户 + 设备 + 平台的令牌会被覆盖更新。

**成功响应 (200)：** `{"success": true, "data": null}`

### GET /api/v1/push/tokens

列出当前用户的所有推送令牌。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "platform": "fcm",
      "device_id": "uuid",
      "created_at": 1709100000000,
      "updated_at": 1709100000000
    }
  ]
}
```

### DELETE /api/v1/push/token

注销推送令牌。

**请求 Body：**

```json
{
  "platform": "fcm",
  "device_id": "uuid"
}
```

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

---

## Bot 管理

### POST /api/v1/bots

注册新 Bot。创建时会生成一个 Bot Token，**仅在创建响应中返回一次**。

**请求 Body：**

```json
{
  "name": "通知机器人",
  "description": "自动发送通知",
  "avatar_url": "https://example.com/bot-avatar.png",
  "permissions": ["send_message", "read_channel"]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | string | 是 | Bot 名称（1-128 字符） |
| description | string | 否 | 描述 |
| avatar_url | string | 否 | 头像 URL |
| permissions | array | 否 | 权限列表 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "bot": {
      "id": "uuid",
      "name": "通知机器人",
      "description": "自动发送通知",
      "owner_id": "uuid",
      "avatar_url": "https://example.com/bot-avatar.png",
      "permissions": ["send_message", "read_channel"],
      "status": "active",
      "created_at": 1709100000000,
      "updated_at": 1709100000000
    },
    "token": "uvbot_019f1234567890abcdef..."
  }
}
```

> **重要**：`token` 字段仅在创建时返回，请妥善保存。

### GET /api/v1/bots

列出当前用户拥有的所有 Bot。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "通知机器人",
      "description": "...",
      "owner_id": "uuid",
      "avatar_url": null,
      "permissions": [],
      "status": "active",
      "created_at": 1709100000000,
      "updated_at": 1709100000000
    }
  ]
}
```

### GET /api/v1/bots/{id}

获取 Bot 详情。

### PUT /api/v1/bots/{id}

更新 Bot 信息。只有 Bot 所有者可以更新。

**请求 Body：**

```json
{
  "name": "新名称",
  "description": "新描述",
  "avatar_url": "https://example.com/new-avatar.png",
  "permissions": ["send_message"],
  "status": "disabled"
}
```

所有字段均为可选。`status` 可选值：`active`/`disabled`/`suspended`。

**成功响应 (200)：** `{"success": true, "data": {"updated": true}}`

### DELETE /api/v1/bots/{id}

删除 Bot。只有 Bot 所有者可以删除。

**成功响应 (200)：** `{"success": true, "data": {"deleted": true}}`

---

## 表情反应（Reactions）

### POST /api/v1/messages/{id}/reactions

为消息添加表情反应。幂等操作，重复添加相同 emoji 不会报错。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | 消息 ID |

**请求 Body：**

```json
{
  "emoji": "👍"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| emoji | string | 是 | 表情符号（不能为空） |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "added": true
  }
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/messages/{message_id}/reactions \
  -H "Authorization: Bearer <user_jwt>" \
  -H "Content-Type: application/json" \
  -d '{"emoji": "👍"}'
```

### GET /api/v1/messages/{id}/reactions

获取消息的所有表情反应，按 emoji 分组，按数量降序排列。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | 消息 ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "emoji": "👍",
      "count": 5,
      "member_ids": ["uuid1", "uuid2", "uuid3", "uuid4", "uuid5"]
    },
    {
      "emoji": "❤️",
      "count": 2,
      "member_ids": ["uuid1", "uuid3"]
    }
  ]
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/messages/{message_id}/reactions" \
  -H "Authorization: Bearer <user_jwt>"
```

### DELETE /api/v1/messages/{id}/reactions/{emoji}

删除自己对消息的某个表情反应。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | 消息 ID |
| emoji | string | 要移除的表情符号（URL 编码） |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "removed": true
  }
}
```

> `removed` 为 `false` 表示该反应不存在（可能已被删除）。

**curl 示例：**

```bash
curl -X DELETE "http://localhost:3000/api/v1/messages/{message_id}/reactions/%F0%9F%91%8D" \
  -H "Authorization: Bearer <user_jwt>"
```

---

## 邀请链接（Invites）

### POST /api/v1/channels/{id}/invites

创建频道邀请链接。需要 `InviteMember` 权限。自动生成 8 位 base62 随机邀请码。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | 频道 ID |

**请求 Body：**

```json
{
  "max_uses": 10,
  "expires_hours": 24
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| max_uses | integer | 否 | 最大使用次数（不填则无限制） |
| expires_hours | integer | 否 | 过期时间（小时，不填则永不过期） |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "channel_id": "uuid",
    "code": "aB3xKm9p",
    "created_by": "uuid",
    "max_uses": 10,
    "use_count": 0,
    "expires_at": 1709186400000,
    "created_at": 1709100000000
  }
}
```

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/channels/{channel_id}/invites \
  -H "Authorization: Bearer <user_jwt>" \
  -H "Content-Type: application/json" \
  -d '{"max_uses": 10, "expires_hours": 24}'
```

### GET /api/v1/channels/{id}/invites

列出频道的所有有效邀请链接（已过期或达到使用上限的不会返回）。需要 `InviteMember` 权限。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | 频道 ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "channel_id": "uuid",
      "code": "aB3xKm9p",
      "created_by": "uuid",
      "max_uses": 10,
      "use_count": 3,
      "expires_at": 1709186400000,
      "created_at": 1709100000000
    }
  ]
}
```

### POST /api/v1/invites/{code}/join

使用邀请码加入频道。会检查邀请是否过期以及是否达到使用上限。重复加入不会报错（幂等）。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| code | string | 邀请码 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "channel_id": "uuid",
    "channel_name": "技术交流群",
    "joined": true
  }
}
```

**错误响应：**
- **400**：邀请已过期或已达使用上限
- **404**：邀请码不存在

**curl 示例：**

```bash
curl -X POST http://localhost:3000/api/v1/invites/aB3xKm9p/join \
  -H "Authorization: Bearer <user_jwt>"
```

### DELETE /api/v1/invites/{code}

撤销邀请链接。邀请创建者本人可直接撤销；非创建者需要频道 `InviteMember` 权限。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| code | string | 邀请码 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "revoked": true
  }
}
```

**curl 示例：**

```bash
curl -X DELETE http://localhost:3000/api/v1/invites/aB3xKm9p \
  -H "Authorization: Bearer <user_jwt>"
```

---

## 已读状态（Read States）

### PUT /api/v1/channels/{id}/read

更新指定频道的已读位置。使用 upsert 语义：如果不存在则创建，存在则更新。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| id | UUID | 频道 ID |

**请求 Body：**

```json
{
  "last_read_message_id": "uuid"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| last_read_message_id | UUID | 是 | 最后已读消息 ID |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "updated": true
  }
}
```

**curl 示例：**

```bash
curl -X PUT http://localhost:3000/api/v1/channels/{channel_id}/read \
  -H "Authorization: Bearer <user_jwt>" \
  -H "Content-Type: application/json" \
  -d '{"last_read_message_id": "01234567-89ab-cdef-0123-456789abcdef"}'
```

### GET /api/v1/channels/unread

获取当前用户已加入的所有频道的未读消息计数。通过对比消息时间戳和已读位置计算。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "channel_id": "uuid",
      "unread_count": 15,
      "last_read_message_id": "uuid"
    },
    {
      "channel_id": "uuid2",
      "unread_count": 0,
      "last_read_message_id": "uuid"
    }
  ]
}
```

> `last_read_message_id` 为 `null` 表示该频道从未标记已读，所有消息均为未读。

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/channels/unread" \
  -H "Authorization: Bearer <user_jwt>"
```

---

## 语音房间（Voice Rooms）

Daemon 提供独立的语音房间 API（不依赖频道）。客户端可创建房间、加入/离开、查询参与者并切换静音。

```
POST   /api/v1/voice/rooms
GET    /api/v1/voice/rooms/{room_id}
DELETE /api/v1/voice/rooms/{room_id}
POST   /api/v1/voice/rooms/{room_id}/join
POST   /api/v1/voice/rooms/{room_id}/leave
GET    /api/v1/voice/rooms/{room_id}/participants
POST   /api/v1/voice/rooms/{room_id}/mute
```

> 具体请求/响应字段以 `voice` 模块的结构体定义为准。

---

## 联系人系统（Contacts）

```
POST /api/v1/contacts/lookup
GET  /api/v1/contacts/requests
POST /api/v1/contacts/requests
POST /api/v1/contacts/requests/{id}/accept
POST /api/v1/contacts/requests/{id}/reject
GET  /api/v1/contacts
POST /api/v1/contacts/presence
```

> 接受请求后会返回私聊频道信息（用于创建 DM 会话）。

### POST /api/v1/contacts/presence

批量查询联系人在线状态（基于 Redis `online:{member_id}` 缓存）。

**请求 Body：**

```json
{
  "member_ids": ["uuid1", "uuid2"]
}
```

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "members": [
      { "member_id": "uuid1", "online": true },
      { "member_id": "uuid2", "online": false }
    ]
  }
}
```

> 在线状态缓存由 WebSocket 心跳刷新（默认 TTL 300 秒）。

---

## 社区管理（Community Admin）

以下端点用于频道/服务器级别的用户管理操作，需要相应的 RBAC 权限。

### 封禁管理

#### POST /api/v1/admin/bans

封禁用户。需要 `BanMember` 权限。全服封禁需要 `Admin` 角色。

**请求 Body：**

```json
{
  "member_id": "uuid",
  "channel_id": "uuid",
  "reason": "违反社区规范",
  "duration_hours": 24
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| member_id | string | 是 | 要封禁的用户 ID |
| channel_id | string | 否 | 频道 ID（不填则为全服封禁） |
| reason | string | 否 | 封禁原因 |
| duration_hours | integer | 否 | 封禁时长（小时），不填则永久封禁 |

**权限规则：**
- 不能封禁同级或更高角色的用户
- 频道封禁需要频道内 `BanMember` 权限
- 全服封禁需要 `Admin` 角色

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "id": "ban_uuid",
    "member_id": "uuid",
    "channel_id": "uuid",
    "banned_by": "uuid",
    "reason": "违反社区规范",
    "expires_at": 1709186400000,
    "created_at": 1709100000000
  }
}
```

> 被封禁的在线用户会被立即断开 WebSocket 连接。

#### GET /api/v1/admin/bans

列出封禁记录（分页）。需要 `BanMember` 权限。

**查询参数：** `page`/`per_page`

**成功响应 (200)：** 带分页信息的封禁记录列表。

#### DELETE /api/v1/admin/bans/{ban_id}

解除封禁。需要 `BanMember` 权限。

**成功响应 (200)：** `{"success": true, "data": {"removed": true}}`

### 禁言管理

#### POST /api/v1/channels/{id}/mute/{member_id}

禁言用户。需要 `KickMember` 权限。

**请求 Body：**

```json
{
  "duration_hours": 1.0,
  "reason": "频繁刷屏"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| duration_hours | number | 否 | 禁言时长（小时），默认 1 小时 |
| reason | string | 否 | 禁言原因 |

**权限规则：** 不能禁言同级或更高角色的用户。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "muted": true,
    "muted_until": 1709103600000
  }
}
```

#### DELETE /api/v1/channels/{id}/mute/{member_id}

解除禁言。需要 `KickMember` 权限。

**成功响应 (200)：** `{"success": true, "data": {"unmuted": true}}`

### 踢出用户

#### POST /api/v1/channels/{id}/kick/{member_id}

踢出频道成员。需要 `KickMember` 权限。不能踢出同级或更高角色的用户。

**成功响应 (200)：** `{"success": true, "data": {"kicked": true}}`

### 角色管理

#### PUT /api/v1/channels/{id}/members/{member_id}/role

变更成员角色。需要 `ManageRoles` 权限。

**请求 Body：**

```json
{
  "role": "admin"
}
```

**角色层级（从高到低）：**

| 角色 | 说明 |
|------|------|
| owner | 所有者（不能通过此接口设置） |
| admin | 管理员 |
| moderator | 版主 |
| member | 普通成员 |
| guest | 访客 |

**权限规则：**
- 不能修改同级或更高角色的用户
- 不能将用户设为比自己更高或相同的角色
- 不能直接设为 Owner（需使用所有权转让流程）

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "updated": true,
    "new_role": "admin"
  }
}
```

### 审计日志

#### GET /api/v1/community/audit-log

查询社区审计日志。需要 `ViewAuditLog` 权限（Admin 角色）。

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| page | integer | 1 | 页码 |
| per_page | integer | 20 | 每页条数（最大 100） |
| action | string | -- | 按操作类型筛选（ban/unban/mute/unmute/kick/role_change） |
| actor_id | string | -- | 按操作者 ID 筛选 |
| since | string | -- | 起始日期（YYYY-MM-DD） |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "actor_id": "uuid",
      "action": "ban",
      "target_type": "member",
      "target_id": "uuid",
      "details": {
        "channel_id": "uuid",
        "reason": "违规",
        "duration_hours": 24
      },
      "created_at": 1709100000000
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 50,
    "total_pages": 3
  }
}
```

### 服务器统计

#### GET /api/v1/community/stats

获取服务器统计信息。需要 `ManageServer` 权限（Owner 角色）。

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "total_members": 1000,
    "online_members": 150,
    "total_channels": 50,
    "total_messages": 500000,
    "total_media_size_bytes": 10737418240,
    "active_bans": 5,
    "uptime_seconds": 86400
  }
}
```

### 成员列表

#### GET /api/v1/community/members

列出所有服务器成员（分页）。需要 `ManageServer` 权限（Owner 角色）。

**查询参数：** `page`/`per_page`

**成功响应 (200)：**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "username": "alice",
      "display_name": "Alice",
      "role": "member",
      "status": "active",
      "created_at": 1709100000000
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 1000,
    "total_pages": 50
  }
}
```

---

## KYC GDPR 数据导出

### GET /api/v1/kyc/export/{user_id}

导出用户的所有 KYC 相关数据，符合 GDPR 数据可携权要求。包含验证记录、文件元数据（不含文件内容）和访问日志。每次调用都会记录到审计日志。

**路径参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| user_id | UUID | 用户 ID |

**成功响应 (200)：**

```json
{
  "user_id": "uuid",
  "exported_at": "2024-02-28T12:00:00Z",
  "verifications": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "provider": "manual",
      "external_id": null,
      "status": "approved",
      "verification_type": "identity",
      "submitted_at": "2024-02-27T10:00:00Z",
      "completed_at": "2024-02-27T15:00:00Z",
      "expires_at": null,
      "result": null,
      "metadata": null,
      "created_at": "2024-02-27T10:00:00Z",
      "updated_at": "2024-02-27T15:00:00Z"
    }
  ],
  "documents": [
    {
      "id": "uuid",
      "verification_id": "uuid",
      "document_type": "id_front",
      "original_filename": "id_front.jpg",
      "content_type": "image/jpeg",
      "file_size": 524288,
      "file_hash": "e3b0c44298fc1c14...",
      "uploaded_at": "2024-02-27T10:05:00Z",
      "deleted_at": null
    }
  ],
  "access_logs": [
    {
      "document_id": "uuid",
      "accessed_by": "admin_uuid",
      "action": "download",
      "accessed_at": "2024-02-27T16:00:00Z"
    }
  ]
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/kyc/export/{user_id}" \
  -H "Authorization: Bearer <admin_jwt>"
```

---

## 内容监听 (Content Monitor)

内容监听模块允许具有 `ManageServer` 权限（Owner 角色）的管理员查看服务器上的群聊消息和私聊中继消息。仅监控 `plaintext_content` 非空的消息，E2E 加密的内容无法查看。

所有内容监听端点均需要用户 JWT，且用户必须具有 `ManageServer` 权限。

### 群聊监控

#### GET /api/v1/admin/monitor/stats

获取内容监控统计概览。

**权限**：`ManageServer`

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "total_channels": 50,
    "total_messages": 500000,
    "viewable_messages": 350000,
    "encrypted_messages": 150000,
    "today_messages": 1200,
    "active_channels_today": 15,
    "total_members": 1000
  }
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
  -H "Authorization: Bearer <user_jwt>"
```

#### GET /api/v1/admin/monitor/channels

获取频道列表（带消息统计）。按最新消息时间降序排列。

**权限**：`ManageServer`

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
  "success": true,
  "data": {
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
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/channels?limit=10&search=讨论" \
  -H "Authorization: Bearer <user_jwt>"
```

#### GET /api/v1/admin/monitor/channels/{id}/messages

获取指定频道的消息列表。仅返回 `plaintext_content` 非空且未删除的消息。

**权限**：`ManageServer`

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
  "success": true,
  "data": {
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
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/channels/{channel_id}/messages?limit=20&search=你好" \
  -H "Authorization: Bearer <user_jwt>"
```

#### GET /api/v1/admin/monitor/channels/{id}/members

获取指定频道的成员列表。

**权限**：`ManageServer`

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
  "success": true,
  "data": {
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
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/channels/{channel_id}/members" \
  -H "Authorization: Bearer <user_jwt>"
```

#### GET /api/v1/admin/monitor/search

全局消息搜索。在所有频道中搜索包含指定关键词的消息。利用 `idx_messages_plaintext_trgm` GIN 索引加速搜索。

**权限**：`ManageServer`

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
  "success": true,
  "data": {
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
}
```

**错误响应 (400)：** `keyword` 缺失或超过 200 字符。

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/search?keyword=讨论&limit=10" \
  -H "Authorization: Bearer <user_jwt>"
```

### 私聊中继监控

私聊中继监控用于查看通过 `offline_messages` 表中转的一对一消息。加密类型（`content_type = 'encrypted'`）的消息内容不会展示，仅显示元数据。

#### GET /api/v1/admin/monitor/relay/stats

获取私聊中继消息统计。

**权限**：`ManageServer`

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
    "total_messages": 10000,
    "encrypted_messages": 8000,
    "plaintext_messages": 2000,
    "today_messages": 150,
    "unique_senders": 200,
    "unique_recipients": 250,
    "total_size_bytes": 52428800
  }
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
  -H "Authorization: Bearer <user_jwt>"
```

#### GET /api/v1/admin/monitor/relay/conversations

获取中继对话列表。按双方用户 ID 聚合，显示每对用户间的消息统计。按最后消息时间降序排列。

**权限**：`ManageServer`

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| limit | integer | 20 | 每页条数（最大 100） |
| offset | integer | 0 | 偏移量 |

**成功响应 (200)：**

```json
{
  "success": true,
  "data": {
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
}
```

**curl 示例：**

```bash
curl "http://localhost:3000/api/v1/admin/monitor/relay/conversations?limit=10" \
  -H "Authorization: Bearer <user_jwt>"
```

#### GET /api/v1/admin/monitor/relay/messages

获取中继消息列表。支持按发送者、接收者、内容类型和时间范围筛选。当同时提供 `sender_id` 和 `recipient_id` 时，查询该对用户之间的双向消息。

**权限**：`ManageServer`

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
  "success": true,
  "data": {
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
}
```

> **注意**：`is_encrypted` 为 `true` 时，`plaintext_preview` 为 `null`，加密内容不会展示。

**curl 示例：**

```bash
# 查看两个用户之间的对话
curl "http://localhost:3000/api/v1/admin/monitor/relay/messages?sender_id={user_a_id}&recipient_id={user_b_id}" \
  -H "Authorization: Bearer <user_jwt>"

# 按内容类型筛选
curl "http://localhost:3000/api/v1/admin/monitor/relay/messages?content_type=text&limit=20" \
  -H "Authorization: Bearer <user_jwt>"
```
