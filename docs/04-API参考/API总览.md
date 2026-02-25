# API 总览

Univona Admin Server 提供两套 API 体系，统一部署在同一服务上。

## 认证方式

| 方式 | 适用场景 | Header |
|------|----------|--------|
| Admin JWT | 管理面板操作 | `Authorization: Bearer <admin_jwt>` |
| X-Admin-Key | 程序化管理访问（向后兼容） | `X-Admin-Key: <key>` |
| 用户 JWT | Daemon 用户操作 | `Authorization: Bearer <user_jwt>` |
| 无认证 | 健康检查、用户认证端点 | -- |

## 端点分类

### 管理面板 API（Admin JWT / X-Admin-Key）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/admin/login` | 管理员登录 |
| GET | `/api/health` | 深度健康检查 |
| GET | `/api/metrics` | Prometheus 指标 |
| GET/POST | `/api/v1/admin/daemons` | Daemon 节点列表/注册 |
| GET/PUT/DELETE | `/api/v1/admin/daemons/{id}` | Daemon 节点详情/更新/删除 |
| GET | `/api/v1/admin/daemons/{id}/health` | Daemon 健康检查 |
| GET | `/api/v1/admin/daemons/{id}/stats` | Daemon 统计 |
| GET | `/api/v1/admin/kyc` | KYC 验证列表 |
| PUT | `/api/v1/admin/kyc/{id}/approve` | 审批通过 KYC |
| PUT | `/api/v1/admin/kyc/{id}/reject` | 拒绝 KYC |
| GET | `/api/v1/admin/communities` | 社区列表 |
| PUT | `/api/v1/admin/communities/{id}` | 更新社区信息 |
| PUT | `/api/v1/admin/communities/{id}/approve` | 审批通过社区 |
| PUT | `/api/v1/admin/communities/{id}/reject` | 拒绝社区 |
| PUT | `/api/v1/admin/communities/{id}/suspend` | 暂停社区 |
| PUT | `/api/v1/admin/communities/{id}/restore` | 恢复社区 |
| PUT | `/api/v1/admin/communities/{id}/resubmit` | 重新提交审核 |
| GET/POST | `/api/v1/admin/users` | 管理员用户列表/创建 |
| PUT/DELETE | `/api/v1/admin/users/{id}` | 管理员更新/删除 |
| POST | `/api/v1/reports` | 创建举报 |
| GET | `/api/v1/reports` | 举报列表 |
| PUT | `/api/v1/reports/{id}` | 审核举报 |
| GET | `/api/v1/admin/audit-logs` | 审计日志 |
| GET | `/api/v1/admin/system-info` | 系统信息 |
| GET | `/api/v1/admin/permissions` | 获取权限矩阵 |
| PUT | `/api/v1/admin/permissions` | 批量更新权限覆盖 |
| DELETE | `/api/v1/admin/permissions/{role}/{permission}` | 删除权限覆盖 |
| GET | `/api/v1/admin/p2p/sessions` | P2P 会话列表 |
| GET | `/api/v1/admin/p2p/stats` | P2P 统计 |
| GET | `/api/v1/admin/p2p/active` | 当前活跃会话 |
| GET/POST | `/api/v1/admin/p2p/ice-servers` | ICE 服务器列表/新增 |
| PUT/DELETE | `/api/v1/admin/p2p/ice-servers/{id}` | ICE 服务器更新/删除 |
| GET | `/api/v1/admin/monitor/stats` | 内容监控统计概览 |
| GET | `/api/v1/admin/monitor/channels` | 监控频道列表 |
| GET | `/api/v1/admin/monitor/channels/{id}/messages` | 频道消息列表 |
| GET | `/api/v1/admin/monitor/channels/{id}/members` | 频道成员列表 |
| GET | `/api/v1/admin/monitor/search` | 全局消息搜索 |
| GET | `/api/v1/admin/monitor/relay/stats` | 私聊中继统计 |
| GET | `/api/v1/admin/monitor/relay/conversations` | 中继对话列表 |
| GET | `/api/v1/admin/monitor/relay/messages` | 中继消息列表 |
| POST | `/api/v1/directory/communities` | 注册社区到目录 |
| GET | `/api/v1/directory/communities` | 搜索/浏览社区目录 |
| GET/PUT/DELETE | `/api/v1/directory/communities/{id}` | 社区详情/更新/删除 |
| GET | `/api/v1/directory/search` | 搜索社区（关键词） |
| POST | `/api/v1/directory/register` | 提交社区注册 |
| POST | `/api/v1/kyc/initiate` | 发起 KYC 验证 |
| GET | `/api/v1/kyc/status/{verification_id}` | 获取 KYC 验证状态 |
| GET | `/api/v1/kyc/my-status` | 获取我的 KYC 状态 |
> 注：KYC 文件上传/导出等端点在当前代码中尚未实现（以 `univona-admin-server/src/api/kyc.rs` 为准）。
> 注：目录与 KYC 端点当前未挂载管理员鉴权中间件，生产环境通常由网关或面板侧进行访问控制。

### 兼容 Daemon 只读 API（Legacy）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/v1/daemon/members` | 只读成员列表 |
| GET | `/api/v1/daemon/members/{id}` | 只读成员详情 |
| GET | `/api/v1/daemon/channels` | 只读频道列表 |
| GET | `/api/v1/daemon/channels/{id}` | 只读频道详情 |
| GET | `/api/v1/daemon/config` | 只读 Daemon 配置 |
| GET | `/api/v1/daemon/stats` | 只读 Daemon 统计 |

### 用户认证 API（公开）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/auth/challenge` | 请求认证挑战 |
| POST | `/api/v1/auth/verify` | 验证挑战响应 |
| POST | `/api/v1/auth/register` | 注册新用户 |
| POST | `/api/v1/auth/refresh` | 刷新访问令牌 |

### Daemon 用户 API（用户 JWT）

> 说明：下表为 **univona-daemon 实际路由**（以 `univona-daemon/src/app.rs` 为准）。

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/v1/prekeys` | 上传 PreKey |
| GET | `/api/v1/prekeys/{member_id}` | 获取 PreKey Bundle |
| DELETE | `/api/v1/prekeys/{device_id}/{key_id}` | 删除 PreKey |
| GET | `/api/v1/prekeys/{device_id}/count` | PreKey 计数 |
| POST/GET | `/api/v1/channels` | 创建/列出频道 |
| GET/PUT/DELETE | `/api/v1/channels/{id}` | 频道详情/更新/删除 |
| PUT | `/api/v1/channels/{id}/encryption-policy` | 更新加密策略 |
| POST/GET | `/api/v1/channels/{id}/members` | 添加/列出频道成员 |
| DELETE | `/api/v1/channels/{id}/members/{uid}` | 移除频道成员 |
| GET | `/api/v1/channels/{id}/messages` | 获取频道消息 |
| PUT/DELETE | `/api/v1/messages/{id}` | 编辑/删除消息 |
| POST/DELETE | `/api/v1/messages/{id}/pin` | 置顶/取消置顶 |
| GET | `/api/v1/channels/{id}/pinned` | 获取置顶消息 |
| POST/GET | `/api/v1/bots` | 创建/列出 Bot |
| GET/PUT/DELETE | `/api/v1/bots/{id}` | Bot 详情/更新/删除 |
| POST | `/api/v1/media/upload` | 上传媒体文件 |
| GET/DELETE | `/api/v1/media/{content_id}` | 下载/删除媒体 |
| GET | `/api/v1/media/{content_id}/thumbnail` | 获取缩略图 |
| POST/GET | `/api/v1/relay/messages` | 上传/获取离线消息 |
| DELETE | `/api/v1/relay/messages/{message_id}` | 确认离线消息 |
| POST | `/api/v1/push/token` | 注册推送令牌 |
| DELETE | `/api/v1/push/token` | 注销推送令牌 |
| GET | `/api/v1/push/tokens` | 列出推送令牌 |
| POST | `/api/v1/voice/rooms` | 创建语音房间 |
| GET | `/api/v1/voice/rooms/{room_id}` | 获取房间信息 |
| DELETE | `/api/v1/voice/rooms/{room_id}` | 关闭语音房间 |
| POST | `/api/v1/voice/rooms/{room_id}/join` | 加入语音房间 |
| POST | `/api/v1/voice/rooms/{room_id}/leave` | 离开语音房间 |
| GET | `/api/v1/voice/rooms/{room_id}/participants` | 获取语音参与者 |
| POST | `/api/v1/voice/rooms/{room_id}/mute` | 切换静音 |
| POST/GET | `/api/v1/contacts/requests` | 发送/列出联系人请求 |
| POST | `/api/v1/contacts/requests/{id}/accept` | 接受请求 |
| POST | `/api/v1/contacts/requests/{id}/reject` | 拒绝请求 |
| POST | `/api/v1/contacts/lookup` | 查找联系人 |
| GET | `/api/v1/contacts` | 列出联系人 |
| POST | `/api/v1/contacts/presence` | 批量查询联系人在线状态 |
| POST/GET | `/api/v1/admin/bans` | 创建/列出封禁 |
| DELETE | `/api/v1/admin/bans/{ban_id}` | 解除封禁 |
| POST/DELETE | `/api/v1/channels/{id}/mute/{member_id}` | 禁言/解除禁言 |
| POST | `/api/v1/channels/{id}/kick/{member_id}` | 踢出用户 |
| PUT | `/api/v1/channels/{id}/members/{member_id}/role` | 变更角色 |
| GET | `/api/v1/admin/audit-log` | 社区审计日志 |
| GET | `/api/v1/admin/stats` | 服务器统计 |
| GET | `/api/v1/admin/members` | 成员列表 |
| GET | `/api/v1/admin/monitor/stats` | 内容监控统计概览 |
| GET | `/api/v1/admin/monitor/channels` | 监控频道列表 |
| GET | `/api/v1/admin/monitor/channels/{id}/messages` | 频道消息列表 |
| GET | `/api/v1/admin/monitor/channels/{id}/members` | 频道成员列表 |
| GET | `/api/v1/admin/monitor/search` | 全局消息搜索 |
| GET | `/api/v1/admin/monitor/relay/stats` | 私聊中继统计 |
| GET | `/api/v1/admin/monitor/relay/conversations` | 中继对话列表 |
| GET | `/api/v1/admin/monitor/relay/messages` | 中继消息列表 |
| GET | `/api/v1/admin/permissions` | 获取权限矩阵 |
| PUT | `/api/v1/admin/permissions` | 批量更新权限覆盖 |
| DELETE | `/api/v1/admin/permissions/{role}/{permission}` | 删除权限覆盖 |

### WebSocket

| 路径 | 说明 |
|------|------|
| `/ws?token=<user_jwt>` | WebSocket 实时通信 |

### 健康检查（公开）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/healthz` | 简单健康检查（返回 `community_name`） |

### Daemon 状态（公开）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/daemon/status` | Daemon 运行状态 |

## 通用响应格式

所有 API 响应使用统一的 JSON 格式：

**成功响应：**

```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

**错误响应：**

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "NOT_FOUND",
    "message": "资源不存在"
  }
}
```

## HTTP 状态码

| 状态码 | 含义 |
|--------|------|
| 200 | 请求成功 |
| 201 | 资源创建成功 |
| 400 | 请求参数错误 |
| 401 | 未认证 |
| 403 | 权限不足 |
| 404 | 资源不存在 |
| 409 | 资源冲突（如用户名重复） |
| 429 | 请求频率过高 |
| 500 | 服务器内部错误 |
| 502 | 上游服务不可用（如 Daemon 连接失败） |
| 504 | 请求超时（默认 30 秒） |

## 分页

列表端点支持分页查询参数：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `limit` | integer | 20 | 每页条数（最大 100） |
| `offset` | integer | 0 | 偏移量 |
| `page` | integer | 1 | 页码（部分端点使用） |
| `per_page` | integer | 20 | 每页条数（部分端点使用） |

某些端点支持带分页信息的响应：

```json
{
  "success": true,
  "data": [...],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```
