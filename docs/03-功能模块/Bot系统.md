# Bot 系统

## 模块概述

Univona Bot 系统允许用户创建和管理自动化机器人。Bot 拥有独立的令牌认证机制和可配置的权限集合，可以被添加到频道中执行自动化任务。Bot 由注册者（Owner）管理生命周期。

## 核心概念

### Bot 注册和管理

每个 Bot 由一个社区成员（Owner）注册和管理：

- **注册**：提供名称、描述、头像和权限列表
- **令牌**：注册时生成唯一的 Bot 令牌，仅在创建时返回一次
- **状态管理**：Bot 可以处于 `active`、`disabled` 或 `suspended` 状态
- **所有权**：只有 Owner 可以更新和删除自己的 Bot

### Bot 令牌

Bot 令牌是 Bot 与服务器通信的凭证：

- 格式：`uvbot_` + UUIDv7（简化格式）
- 令牌仅在创建 Bot 时返回一次（明文），需要安全保管
- 服务端存储令牌的哈希值（`token_hash`），不存储明文
- 令牌丢失后只能重新创建 Bot

### Bot 命令

Bot 可以注册自定义命令供用户调用：

- **命令名称**：在 Bot 范围内唯一
- **描述**：命令的功能说明
- **使用提示**：使用方法的简短说明

### Bot 权限

Bot 的权限通过 JSON 数组配置，定义 Bot 在频道中可以执行的操作：

```json
["SendMessage", "UploadMedia", "PinMessage"]
```

Bot 的权限受限于其 Owner 的权限 -- Bot 不能拥有超出其 Owner 的权限。

## 数据模型

### bots 表

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID | 主键（UUIDv7） |
| `name` | VARCHAR(128) | Bot 名称 |
| `description` | TEXT | Bot 描述 |
| `owner_id` | UUID | 所有者（创建者）ID |
| `token_hash` | VARCHAR(256) | 令牌哈希值 |
| `avatar_url` | TEXT | Bot 头像 URL |
| `permissions` | JSONB | 权限列表 |
| `status` | VARCHAR(32) | 状态：`active`/`disabled`/`suspended` |
| `created_at` | TIMESTAMPTZ | 创建时间 |
| `updated_at` | TIMESTAMPTZ | 更新时间 |

### bot_commands 表

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID | 主键 |
| `bot_id` | UUID | 关联 Bot |
| `name` | VARCHAR(64) | 命令名称（Bot 内唯一） |
| `description` | TEXT | 命令描述 |
| `usage_hint` | VARCHAR(256) | 使用提示 |
| `created_at` | TIMESTAMPTZ | 创建时间 |

唯一约束：`(bot_id, name)`

## API 端点

### Bot CRUD

```
POST   /api/v1/bots         -- 注册 Bot（返回令牌）
GET    /api/v1/bots         -- 列出当前用户的 Bot
GET    /api/v1/bots/{id}    -- 获取 Bot 详情
PUT    /api/v1/bots/{id}    -- 更新 Bot
DELETE /api/v1/bots/{id}    -- 删除 Bot
```

## 配置

```toml
[bots]
# 每用户最大 Bot 数量
max_bots_per_user = 10
# Bot 名称最大长度
max_name_length = 128
```

## 示例

### 注册 Bot

```json
// POST /api/v1/bots
{
  "name": "通知机器人",
  "description": "自动发送频道通知和提醒",
  "avatar_url": "https://example.com/bot-avatar.png",
  "permissions": ["SendMessage", "PinMessage"]
}

// 响应（仅创建时返回令牌）：
{
  "success": true,
  "data": {
    "bot": {
      "id": "01945a2b-3c4d-7e8f-...",
      "name": "通知机器人",
      "description": "自动发送频道通知和提醒",
      "owner_id": "owner-uuid",
      "avatar_url": "https://example.com/bot-avatar.png",
      "permissions": ["SendMessage", "PinMessage"],
      "status": "active",
      "created_at": 1700000000000,
      "updated_at": 1700000000000
    },
    "token": "uvbot_01945a2b3c4d7e8f9a0b1c2d3e4f5a6b"
  }
}
```

> **重要**：`token` 仅在创建响应中返回一次，请妥善保管。

### 更新 Bot

```json
// PUT /api/v1/bots/{id}
{
  "name": "通知助手",
  "description": "更新后的描述",
  "permissions": ["SendMessage", "PinMessage", "UploadMedia"],
  "status": "active"
}

// 响应：
{
  "success": true,
  "data": {
    "updated": true
  }
}
```

### Bot 状态管理

```json
// 禁用 Bot
// PUT /api/v1/bots/{id}
{
  "status": "disabled"
}

// 重新启用 Bot
// PUT /api/v1/bots/{id}
{
  "status": "active"
}
```

可用的状态值：
- `active`：正常运行
- `disabled`：Owner 手动禁用
- `suspended`：管理员挂起

### 删除 Bot

```bash
curl -X DELETE https://community.example.com/api/v1/bots/01945a2b... \
  -H "Authorization: Bearer eyJ..."
```

```json
// 响应：
{
  "success": true,
  "data": {
    "deleted": true
  }
}
```

删除 Bot 会级联删除所有关联的命令注册记录。
