# Admin Server 表结构

Univona Admin Server 使用 PostgreSQL 数据库，通过 SQLx 迁移系统自动管理表结构。首次启动时会自动执行所有迁移脚本。

## 表结构概览

数据库表按功能分为以下几大类：

| 分类 | 表数量 | 说明 |
|------|--------|------|
| 核心表 | 6 | 成员、设备、频道等社区基础结构 |
| 认证表 | 2 | 管理员和用户认证 |
| 消息表 | 5 | 消息存储、反应、置顶、已读 |
| 媒体表 | 1 | 媒体文件元数据 |
| 管理表 | 4 | Daemon 节点、全局用户、社区、KYC |
| 推送表 | 1 | 推送令牌注册 |
| 同步表 | 1 | 多设备同步游标 |
| 社区表 | 3 | 邀请、封禁、Bot |
| 审计表 | 2 | 审计日志、面板配置 |
| 权限表 | 1 | 权限覆盖 |
| 举报表 | 1 | 滥用举报 |

---

## 核心表

### members — 社区成员

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 成员唯一标识 |
| `username` | VARCHAR(64) | NOT NULL, UNIQUE | 用户名 |
| `display_name` | VARCHAR(128) | — | 显示名称 |
| `identity_key` | BYTEA | NOT NULL | 身份公钥（Signal 协议） |
| `avatar_url` | TEXT | — | 头像 URL |
| `role` | VARCHAR(32) | NOT NULL, DEFAULT 'member' | 角色：`member`、`moderator`、`admin` |
| `status` | VARCHAR(32) | NOT NULL, DEFAULT 'active' | 状态：`active`、`suspended` |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

### devices — 成员设备

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 设备唯一标识 |
| `member_id` | UUID | NOT NULL, FK -> members(id) ON DELETE CASCADE | 所属成员 |
| `device_name` | VARCHAR(128) | — | 设备名称 |
| `platform` | VARCHAR(32) | NOT NULL | 平台：`android`、`ios`、`desktop`、`web` |
| `identity_key` | BYTEA | NOT NULL | 设备身份公钥 |
| `signed_prekey` | BYTEA | NOT NULL | 签名预密钥 |
| `signed_prekey_signature` | BYTEA | NOT NULL | 签名预密钥签名 |
| `signed_prekey_id` | INTEGER | NOT NULL | 签名预密钥 ID |
| `last_seen_at` | TIMESTAMPTZ | — | 最后在线时间 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 注册时间 |

**索引：** `idx_devices_member_id` ON (member_id)

### prekeys — X3DH 一次性预密钥

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | SERIAL | PRIMARY KEY | 自增 ID |
| `device_id` | UUID | NOT NULL, FK -> devices(id) ON DELETE CASCADE | 所属设备 |
| `prekey_id` | INTEGER | NOT NULL | 预密钥 ID |
| `public_key` | BYTEA | NOT NULL | 预密钥公钥 |
| `used` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已使用 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |

**索引：**
- `idx_prekeys_device_id` ON (device_id)
- `idx_prekeys_unused` ON (device_id, used) WHERE NOT used -- 部分索引，加速未使用预密钥查询

### channels — 频道（群组/对话）

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 频道唯一标识 |
| `name` | VARCHAR(128) | NOT NULL | 频道名称 |
| `description` | TEXT | — | 频道描述 |
| `channel_type` | VARCHAR(32) | NOT NULL, DEFAULT 'group' | 类型：`group`、`direct`、`announcement` |
| `avatar_url` | TEXT | — | 频道头像 |
| `created_by` | UUID | FK -> members(id) | 创建者 |
| `encryption_policy` | TEXT | NOT NULL, DEFAULT 'plaintext' | 加密策略：`plaintext`、`e2e_required`、`admin_encrypted`（旧数据可能存在 `optional/required` 历史值） |
| `message_seq` | BIGINT | NOT NULL, DEFAULT 0 | 消息序列号计数器 |
| `history_visible` | BOOLEAN | NOT NULL, DEFAULT true | 新成员是否可见历史消息 |
| `announcement` | TEXT | — | 频道公告 |
| `category_id` | UUID | FK -> channel_categories(id) ON DELETE SET NULL | 所属分类 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

> 系统会自动创建一个 `general` 默认频道（ID: `00000000-0000-0000-0000-000000000001`）。

### channel_members — 频道成员关系

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `channel_id` | UUID | NOT NULL, FK -> channels(id) ON DELETE CASCADE | 频道 ID |
| `member_id` | UUID | NOT NULL, FK -> members(id) ON DELETE CASCADE | 成员 ID |
| `role` | VARCHAR(32) | NOT NULL, DEFAULT 'member' | 频道角色 |
| `muted_until` | TIMESTAMPTZ | — | 禁言截止时间 |
| `notification_level` | TEXT | NOT NULL, DEFAULT 'all' | 通知级别：`all`/`mentions_only`/`muted` |
| `joined_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 加入时间 |

**主键：** (channel_id, member_id)

**索引：** `idx_channel_members_member` ON (member_id)

### server_config — 服务器配置键值对

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `key` | VARCHAR(128) | PRIMARY KEY | 配置键 |
| `value` | JSONB | NOT NULL | 配置值（JSON 格式） |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**默认值：**
- `server.name`: `"Univona Community"`
- `server.description`: `"A Univona community server"`
- `server.max_members`: `1000`
- `server.registration_open`: `true`

---

## 认证表

### admins — 管理员账户

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 管理员唯一标识 |
| `username` | VARCHAR(64) | NOT NULL, UNIQUE | 管理员用户名 |
| `password_hash` | VARCHAR(256) | NOT NULL | bcrypt 密码哈希 |
| `role` | VARCHAR(32) | NOT NULL, DEFAULT 'admin' | 角色：`admin`、`super_admin` |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

### auth_challenges — 用户认证挑战

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | TEXT | PRIMARY KEY | 挑战 ID |
| `nonce` | TEXT | NOT NULL | 随机挑战 nonce |
| `used` | BOOLEAN | NOT NULL, DEFAULT false | 是否已使用 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | 过期时间 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |

**索引：** `idx_auth_challenges_expires` ON (expires_at)

---

## 消息表

### messages — 消息存储

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 消息唯一标识 |
| `channel_id` | UUID | NOT NULL, FK -> channels(id) ON DELETE CASCADE | 所属频道 |
| `sender_id` | UUID | NOT NULL | 发送者 ID |
| `encrypted_content` | BYTEA | — | 加密消息内容 |
| `plaintext_content` | TEXT | — | 明文消息内容（非加密频道） |
| `is_encrypted` | BOOLEAN | NOT NULL, DEFAULT true | 是否加密 |
| `message_type` | TEXT | NOT NULL, DEFAULT 'text' | 消息类型：`text`、`image`、`file`、`system` |
| `seq_num` | BIGINT | — | 频道内序列号 |
| `server_timestamp` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 服务端时间戳 |
| `client_timestamp` | BIGINT | — | 客户端时间戳（毫秒） |
| `reply_to` | UUID | FK -> messages(id) ON DELETE SET NULL | 回复消息 ID |
| `forward_from_channel` | TEXT | — | 转发来源频道 |
| `forward_from_sender` | TEXT | — | 转发来源发送者 |
| `retention_policy` | VARCHAR(32) | NOT NULL, DEFAULT 'permanent' | 保留策略 |
| `source_ip` | VARCHAR(45) | — | 来源 IP（合规用途） |
| `expires_at` | TIMESTAMPTZ | — | 过期时间（阅后即焚） |
| `recalled_at` | TIMESTAMPTZ | — | 撤回时间 |
| `recalled_by` | UUID | — | 撤回操作者 |
| `is_pinned` | BOOLEAN | NOT NULL, DEFAULT false | 是否置顶 |
| `edited_at` | TIMESTAMPTZ | — | 编辑时间 |
| `deleted_at` | TIMESTAMPTZ | — | 软删除时间 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |

**索引：**
- `idx_messages_channel_created` ON (channel_id, created_at)
- `idx_messages_sender` ON (sender_id)
- `idx_messages_seq_num` ON (seq_num)
- `idx_messages_is_pinned` ON (is_pinned) WHERE is_pinned = true
- `idx_messages_deleted_at` ON (deleted_at) WHERE deleted_at IS NOT NULL

### offline_messages — 离线消息队列

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 消息唯一标识 |
| `recipient_id` | UUID | NOT NULL | 接收者 ID |
| `sender_id` | UUID | NOT NULL | 发送者 ID |
| `encrypted_payload` | BYTEA | NOT NULL | 加密载荷 |
| `content_type` | TEXT | NOT NULL, DEFAULT 'encrypted' | 内容类型 |
| `payload_size` | INTEGER | NOT NULL, DEFAULT 0 | 载荷大小（字节） |
| `message_id` | TEXT | — | 客户端消息 ID（用于 ack/去重） |
| `recipient_device_id` | UUID | — | 指定接收设备 |
| `expires_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() + 30 days | 过期时间（默认 30 天） |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |

**索引：**
- `idx_offline_messages_recipient_created` ON (recipient_id, created_at)
- `idx_offline_messages_expires` ON (expires_at)
- `idx_offline_messages_sender` ON (sender_id)

### reactions — 消息表情反应

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 反应唯一标识 |
| `message_id` | UUID | NOT NULL, FK -> messages(id) ON DELETE CASCADE | 消息 ID |
| `member_id` | UUID | NOT NULL | 成员 ID |
| `emoji` | TEXT | NOT NULL | Emoji 字符 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |

**唯一约束：** (message_id, member_id, emoji)

**索引：**
- `idx_reactions_message` ON (message_id)
- `idx_reactions_member` ON (member_id)

### pinned_messages — 置顶消息

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 置顶记录 ID |
| `channel_id` | UUID | NOT NULL, FK -> channels(id) ON DELETE CASCADE | 频道 ID |
| `message_id` | UUID | NOT NULL, FK -> messages(id) ON DELETE CASCADE | 消息 ID |
| `pinned_by` | UUID | NOT NULL | 置顶操作者 ID |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 置顶时间 |

**唯一约束：** (channel_id, message_id)

### read_states — 已读状态

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 记录 ID |
| `channel_id` | UUID | NOT NULL, FK -> channels(id) ON DELETE CASCADE | 频道 ID |
| `member_id` | UUID | NOT NULL | 成员 ID |
| `last_read_message_id` | UUID | FK -> messages(id) ON DELETE SET NULL | 最后已读消息 |
| `last_read_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 最后已读时间 |

**唯一约束：** (channel_id, member_id)

**索引：** `idx_read_states_member` ON (member_id)

### mentions — @提及记录

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 记录 ID |
| `message_id` | UUID | NOT NULL, FK -> messages(id) ON DELETE CASCADE | 被提及消息 |
| `channel_id` | UUID | NOT NULL | 频道 ID |
| `mentioned_member_id` | UUID | NOT NULL | 被提及用户 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |

### channel_categories — 频道分类

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 分类 ID |
| `name` | TEXT | NOT NULL | 分类名 |
| `position` | INTEGER | NOT NULL, DEFAULT 0 | 排序 |
| `created_by` | UUID | — | 创建者 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |

---

## 媒体表

### media_attachments — 媒体文件元数据

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 媒体文件 ID |
| `content_id` | TEXT | NOT NULL, UNIQUE | 内容标识（用于下载） |
| `uploader_id` | UUID | NOT NULL | 上传者 ID |
| `file_name` | TEXT | NOT NULL | 原始文件名 |
| `mime_type` | TEXT | NOT NULL | MIME 类型 |
| `file_size` | BIGINT | NOT NULL | 文件大小（字节） |
| `sha256_hash` | TEXT | NOT NULL | 文件 SHA256 哈希 |
| `storage_path` | TEXT | NOT NULL | 本地存储路径 |
| `thumbnail_path` | TEXT | — | 缩略图路径（图片/视频） |
| `width` | INTEGER | — | 图片/视频宽度（像素） |
| `height` | INTEGER | — | 图片/视频高度（像素） |
| `duration_secs` | REAL | — | 音频/视频时长（秒） |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |

**索引：**
- `idx_media_attachments_uploader` ON (uploader_id)
- `idx_media_attachments_content_id` ON (content_id)

---

## 管理表

### daemon_nodes — Daemon 节点

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 节点唯一标识 |
| `name` | VARCHAR(128) | NOT NULL | 节点名称 |
| `endpoint` | TEXT | NOT NULL, UNIQUE | 节点 API 端点 URL |
| `api_key_hash` | VARCHAR(128) | NOT NULL | API Key 哈希 |
| `status` | VARCHAR(32) | NOT NULL, DEFAULT 'offline' | 状态：`online`、`offline`、`error` |
| `last_heartbeat` | TIMESTAMPTZ | — | 最后心跳时间 |
| `online_members` | INTEGER | NOT NULL, DEFAULT 0 | 在线成员数 |
| `channel_count` | INTEGER | NOT NULL, DEFAULT 0 | 频道数量 |
| `message_count` | BIGINT | NOT NULL, DEFAULT 0 | 消息总数 |
| `version` | TEXT | — | Daemon 版本号 |
| `metadata` | JSONB | DEFAULT '{}' | 扩展元数据 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

### global_users — 全局用户目录

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY | 用户唯一标识 |
| `username` | VARCHAR(64) | NOT NULL, UNIQUE | 全局用户名 |
| `display_name` | VARCHAR(128) | — | 显示名称 |
| `identity_key` | BYTEA | NOT NULL | 身份公钥 |
| `home_daemon_id` | UUID | FK -> daemon_nodes(id) | 所属 Daemon 节点 |
| `avatar_url` | TEXT | — | 头像 URL |
| `status` | VARCHAR(32) | NOT NULL, DEFAULT 'active' | 用户状态 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**索引：** `idx_global_users_daemon` ON (home_daemon_id)

### communities — 社区目录

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 社区唯一标识 |
| `name` | VARCHAR(128) | NOT NULL | 社区名称 |
| `description` | TEXT | — | 社区描述 |
| `owner_id` | UUID | NOT NULL | 所有者 ID |
| `endpoint` | TEXT | NOT NULL | 社区服务端点 |
| `icon_url` | TEXT | — | 图标 URL |
| `banner_url` | TEXT | — | 横幅图片 URL |
| `member_count` | INTEGER | NOT NULL, DEFAULT 0 | 成员数量 |
| `category` | VARCHAR(64) | — | 分类 |
| `tags` | TEXT[] | DEFAULT '{}' | 标签数组 |
| `is_verified` | BOOLEAN | NOT NULL, DEFAULT FALSE | 是否已认证 |
| `is_listed` | BOOLEAN | NOT NULL, DEFAULT TRUE | 是否在目录中可见 |
| `status` | VARCHAR(32) | NOT NULL, DEFAULT 'pending' | 状态：`pending`、`active`、`suspended` |
| `metadata` | JSONB | DEFAULT '{}' | 扩展元数据 |
| `search_vector` | TSVECTOR | — | 全文搜索向量（自动维护） |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 更新时间 |

**索引：**
- `idx_communities_search` ON (search_vector) USING GIN -- 全文搜索
- `idx_communities_category` ON (category)
- `idx_communities_status` ON (status)
- `idx_communities_listed` ON (is_listed, status)
- `idx_communities_owner` ON (owner_id)

**触发器：** `communities_search_trigger` - 在 INSERT 或 UPDATE 时自动更新 `search_vector` 字段。

### kyc_verifications — KYC 验证记录

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 记录 ID |
| `user_id` | UUID | NOT NULL | 用户 ID |
| `provider` | VARCHAR(64) | NOT NULL, DEFAULT 'mock' | 验证提供商 |
| `external_id` | VARCHAR(256) | — | 外部系统 ID |
| `status` | VARCHAR(32) | NOT NULL, DEFAULT 'pending' | 状态：`pending`、`approved`、`rejected` |
| `verification_type` | VARCHAR(64) | NOT NULL, DEFAULT 'identity' | 验证类型 |
| `submitted_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 提交时间 |
| `completed_at` | TIMESTAMPTZ | — | 完成时间 |
| `expires_at` | TIMESTAMPTZ | — | 过期时间 |
| `result` | JSONB | DEFAULT '{}' | 验证结果详情 |
| `metadata` | JSONB | DEFAULT '{}' | 扩展元数据 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 更新时间 |

**索引：**
- `idx_kyc_user` ON (user_id)
- `idx_kyc_status` ON (status)
- `idx_kyc_provider` ON (provider, external_id)

---

## 推送表

### push_tokens — 推送令牌

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 记录 ID |
| `member_id` | UUID | NOT NULL | 成员 ID |
| `device_id` | UUID | NOT NULL | 设备 ID |
| `platform` | TEXT | NOT NULL, CHECK IN ('fcm', 'apns', 'unified_push') | 推送平台 |
| `token` | TEXT | NOT NULL | 推送令牌 |
| `endpoint` | TEXT | — | UnifiedPush 端点（可选） |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 更新时间 |

**唯一约束：** (member_id, device_id, platform)

**索引：**
- `idx_push_tokens_member` ON (member_id)
- `idx_push_tokens_device` ON (device_id)

---

## 同步表

### sync_cursors — 多设备同步游标

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 记录 ID |
| `member_id` | UUID | NOT NULL | 成员 ID |
| `device_id` | UUID | NOT NULL | 设备 ID |
| `channel_id` | UUID | NOT NULL, FK -> channels(id) ON DELETE CASCADE | 频道 ID |
| `last_read_seq` | BIGINT | NOT NULL, DEFAULT 0 | 最后已读序列号 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 更新时间 |

**唯一约束：** (member_id, device_id, channel_id)

**索引：**
- `idx_sync_cursors_member_device` ON (member_id, device_id)
- `idx_sync_cursors_channel` ON (channel_id)

---

## 社区表

### invites — 邀请链接

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 邀请 ID |
| `channel_id` | UUID | FK -> channels(id) ON DELETE CASCADE | 关联频道（可选） |
| `code` | TEXT | NOT NULL, UNIQUE | 邀请码 |
| `created_by` | UUID | NOT NULL | 创建者 ID |
| `max_uses` | INTEGER | — | 最大使用次数（NULL 为无限） |
| `use_count` | INTEGER | NOT NULL, DEFAULT 0 | 已使用次数 |
| `expires_at` | TIMESTAMPTZ | — | 过期时间（NULL 为永不过期） |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |

**索引：**
- `idx_invites_channel` ON (channel_id)
- `idx_invites_code` ON (code)
- `idx_invites_expires` ON (expires_at)

### bans — 封禁记录

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 封禁记录 ID |
| `member_id` | UUID | NOT NULL | 被封禁成员 ID |
| `banned_by` | UUID | NOT NULL | 执行封禁的成员 ID |
| `reason` | TEXT | — | 封禁原因 |
| `channel_id` | UUID | FK -> channels(id) ON DELETE CASCADE | 关联频道（NULL 为全局封禁） |
| `expires_at` | TIMESTAMPTZ | — | 封禁截止时间（NULL 为永久封禁） |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 封禁时间 |

**索引：**
- `idx_bans_member` ON (member_id)
- `idx_bans_channel` ON (channel_id)
- `idx_bans_expires` ON (expires_at)

### bots — Bot 注册

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | Bot ID |
| `name` | TEXT | NOT NULL | Bot 名称 |
| `owner_id` | UUID | NOT NULL | 所有者成员 ID |
| `description` | TEXT | — | Bot 描述 |
| `avatar_url` | TEXT | — | Bot 头像 URL |
| `token_hash` | TEXT | NOT NULL | API Token 哈希 |
| `permissions` | JSONB | NOT NULL, DEFAULT '[]' | 权限列表 |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true | 是否启用 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 更新时间 |

**索引：** `idx_bots_owner` ON (owner_id)

### bot_commands — Bot 命令

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 命令 ID |
| `bot_id` | UUID | NOT NULL, FK -> bots(id) ON DELETE CASCADE | 所属 Bot |
| `name` | TEXT | NOT NULL | 命令名称 |
| `description` | TEXT | — | 命令描述 |
| `usage` | TEXT | — | 使用说明 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |

**唯一约束：** (bot_id, name)

---

## 举报表

### abuse_reports — 滥用举报

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | 举报 ID |
| `reporter_id` | UUID | NOT NULL | 举报者 ID |
| `target_type` | VARCHAR(64) | NOT NULL | 举报目标类型：`user`、`message`、`channel` |
| `target_id` | UUID | NOT NULL | 举报目标 ID |
| `reason` | VARCHAR(64) | NOT NULL | 举报原因：`spam`、`harassment`、`illegal` 等 |
| `description` | TEXT | — | 详细描述 |
| `evidence` | JSONB | DEFAULT '[]' | 证据附件 |
| `status` | VARCHAR(32) | NOT NULL, DEFAULT 'pending' | 状态：`pending`、`reviewing`、`resolved`、`dismissed` |
| `reviewed_by` | UUID | — | 审核管理员 ID |
| `reviewed_at` | TIMESTAMPTZ | — | 审核时间 |
| `resolution` | TEXT | — | 处理结果说明 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 创建时间 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | 更新时间 |

**索引：**
- `idx_reports_target` ON (target_type, target_id)
- `idx_reports_status` ON (status)
- `idx_reports_reporter` ON (reporter_id)

---

## 权限表

### permission_overrides — 权限覆盖

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `role` | VARCHAR(32) | NOT NULL | 角色名称（guest/member/moderator/admin/owner） |
| `permission` | VARCHAR(64) | NOT NULL | 权限名称 |
| `granted` | BOOLEAN | NOT NULL | 是否授予 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**主键：** (role, permission)

> 此表为空时，所有权限完全按默认角色层级生效（向后兼容）。Owner 角色的权限始终硬编码为全部授予，不受覆盖表影响。

---

## 审计表

### audit_log — 审计日志

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | BIGSERIAL | PRIMARY KEY | 自增 ID |
| `admin_id` | UUID | FK -> admins(id) | 操作管理员 ID |
| `action` | VARCHAR(128) | NOT NULL | 操作类型：`login`、`create_user`、`ban_user` 等 |
| `target_type` | VARCHAR(64) | — | 目标实体类型 |
| `target_id` | VARCHAR(128) | — | 目标实体 ID |
| `details` | JSONB | DEFAULT '{}' | 操作详情 |
| `ip_address` | INET | — | 操作者 IP 地址 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 操作时间 |

**索引：**
- `idx_audit_log_admin` ON (admin_id)
- `idx_audit_log_created` ON (created_at)

### panel_config — 面板配置键值对

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `key` | VARCHAR(128) | PRIMARY KEY | 配置键 |
| `value` | JSONB | NOT NULL | 配置值 |
| `updated_at` | TIMESTAMPTZ | NOT NULL, DEFAULT now() | 更新时间 |

**默认值：**
- `panel.name`: `"Univona Panel"`
- `panel.registration_open`: `true`

---

## 迁移文件列表

| 文件 | 说明 |
|------|------|
| `20260212000000_initial_schema.sql` | 初始表结构（Daemon 核心表 + Panel 管理表） |
| `20260220000000_directory.sql` | 社区目录表和全文搜索 |
| `20260221000000_kyc.sql` | KYC 验证表 |
| `20260222000000_reports.sql` | 滥用举报表 |
| `20260228000000_daemon_full_integration.sql` | Daemon 完整功能集成（消息、媒体、推送、Bot 等） |
| `20260301000000_permission_overrides.sql` | 权限覆盖表 |

所有迁移均为幂等操作（`IF NOT EXISTS` / `DO $$ ... EXCEPTION WHEN ... $$`），支持安全地重复执行。

---

## 补充：Admin Server 特有表

以下表为 Admin Server 相比 Daemon 的新增表结构（目录、KYC、全局用户、举报等）：

| 表名 | 说明 |
|------|------|
| `global_users` | 全局用户目录（跨社区发现） |
| `communities` | 社区目录（注册/搜索/审批） |
| `daemon_nodes` | Daemon 节点注册与心跳 |
| `admins` | 管理员账户 |
| `kyc_verifications` | KYC 验证记录 |
| `kyc_documents` | KYC 文件加密存储 |
| `kyc_access_log` | KYC 访问审计 |
| `abuse_reports` | 滥用举报 |
| `account_deletion_requests` | GDPR 删除请求 |
| `panel_config` | 面板配置键值对 |

> 详细字段可参考 Admin Server 的迁移文件（`univona-admin-server/migrations/`）。

## 相关文档

- [Daemon 表结构](./Daemon表结构.md)
- [ER 关系图](./ER关系图.md)
- [迁移历史](./迁移历史.md)
