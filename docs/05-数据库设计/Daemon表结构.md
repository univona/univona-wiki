# Daemon 表结构

## 概述

Daemon 复用 Admin Server 的大部分核心表结构，但不包含目录、KYC、全局用户等中枢功能表。下表列出 Daemon 核心使用的主要表。

## 核心表

| 分类 | 表名 | 说明 |
|------|------|------|
| 身份 | `members` / `devices` | 成员与设备注册 |
| 认证 | `auth_challenges` | Challenge-Response 挑战 |
| 密钥 | `prekeys` | X3DH 一次性预密钥 |
| 频道 | `channels` / `channel_members` | 频道与成员关系 |
| 消息 | `messages` | 消息存储（支持明文/密文） |
| 离线 | `offline_messages` | 离线消息队列 |
| 已读 | `read_states` | 已读状态 |
| 反应 | `reactions` | 表情反应 |
| 置顶 | `pinned_messages` | 置顶消息 |
| 媒体 | `media_attachments` | 媒体文件与缩略图 |
| 权限 | `permission_overrides` | 权限覆盖 |
| 邀请 | `invites` | 频道邀请链接 |
| 封禁 | `bans` | 封禁与禁言 |
| Bot | `bots` / `bot_commands` | 机器人与命令 |
| 同步 | `sync_cursors` | 多设备同步游标 |
| 推送 | `push_tokens` | 推送令牌 |
| 联系人 | `contact_requests` | 联系人请求与私聊频道 |
| 配置 | `server_config` | 服务端配置 KV |
| 审计 | `audit_log` | 操作日志 |

## 与 Admin Server 的差异

Daemon **不包含**以下中枢级表：`communities`、`global_users`、`admins`、`daemon_nodes`、`kyc_*`、`abuse_reports`、`panel_config` 等。

## 相关文档

- [Admin Server 表结构](./Admin-Server表结构.md)
- [ER 关系图](./ER关系图.md)
- [迁移历史](./迁移历史.md)
