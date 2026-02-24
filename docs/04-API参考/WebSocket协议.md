# WebSocket 协议

## 连接

WebSocket 端点：`/ws?token=<user_jwt>`

由于浏览器 WebSocket API 不支持自定义请求头，JWT token 通过查询参数传递。

### 连接流程

```text
客户端                          服务端
  |                               |
  |  GET /ws?token=<jwt>          |
  |  Upgrade: websocket           |
  |------------------------------>|
  |                               | 验证 JWT (access_token)
  |                               | 注册连接到 ConnectionManager
  |                               | 设置在线状态 (TTL 5min)
  |  101 Switching Protocols      |
  |<------------------------------|
  |                               |
  |                               | 推送离线消息（最多 200 条）
  |  <Binary: 离线消息>           |
  |<------------------------------|
  |                               |
  |  <Binary: Envelope>           |
  |<=============================>|
  |                               |
  |  Ping (30s 间隔)              |
  |<------------------------------|
  |  Pong                         |
  |------------------------------>|
```

### 连接参数

- **最大帧大小**：64 KB
- **最大消息大小**：256 KB
- **请求超时**：30 秒（HTTP 层）

### 认证失败

如果 JWT 无效或已过期，服务端返回 `401 Unauthorized` 而不是升级连接。

## 消息格式

所有 WebSocket 消息使用 **Protobuf 编码**的 `Envelope` 格式传输（Binary 帧），不接受 Text 帧。

```protobuf
syntax = "proto3";
package univona.v1;

message Envelope {
  string message_id = 1;           // 消息唯一 ID (UUIDv7)
  string sender_device_id = 2;     // 发送者设备 ID
  string recipient_device_id = 3;  // 接收者设备 ID（群消息为空）
  string channel_id = 4;           // 频道 ID（频道消息使用）
  MessageType message_type = 5;    // 消息类型
  bytes encrypted_content = 6;     // 加密内容（密文）
  int64 server_timestamp = 7;      // 服务端时间戳（毫秒）
  int64 client_timestamp = 8;      // 客户端时间戳（毫秒）
  uint32 protocol_version = 9;     // 协议版本
  bool is_encrypted = 10;          // 是否端到端加密
}
```

## 消息类型

> 枚举定义以 `protocol/univona/v1/envelope.proto` 为准。

### 内容消息（持久化，需要序列号）

| 类型 ID | 名称 | 方向 | 说明 |
|---------|------|------|------|
| 0 | UNSPECIFIED | 双向 | 未指定类型（保留值） |
| 1 | TEXT | 双向 | 文本消息 |
| 2 | IMAGE | 双向 | 图片消息 |
| 3 | FILE | 双向 | 文件消息 |
| 4 | VOICE | 双向 | 语音消息 |
| 5 | VIDEO | 双向 | 视频消息 |
| 6 | STICKER | 双向 | 贴纸消息 |

### 密钥交换消息（持久化，不需要序列号）

| 类型 ID | 名称 | 方向 | 说明 |
|---------|------|------|------|
| 10 | KEY_EXCHANGE | 双向 | X3DH 密钥交换 |
| 11 | PREKEY_BUNDLE | 双向 | PreKey Bundle |
| 12 | SENDER_KEY_DISTRIBUTION | 双向 | Sender Key 分发 |

### 临时消息（不持久化，直接转发）

| 类型 ID | 名称 | 方向 | 说明 |
|---------|------|------|------|
| 20 | READ_RECEIPT | 双向 | 已读回执 |
| 21 | TYPING_INDICATOR | 双向 | 正在输入指示器 |

### 系统消息（持久化）

| 类型 ID | 名称 | 方向 | 说明 |
|---------|------|------|------|
| 30 | SYSTEM | 服务端→客户端 | 系统通知 |
| 31 | GROUP_MEMBER_CHANGE | 服务端→客户端 | 群成员变更通知 |

### 设备管理消息

| 类型 ID | 名称 | 方向 | 说明 |
|---------|------|------|------|
| 40 | DEVICE_ANNOUNCEMENT | 双向 | 新设备声明 |
| 41 | DEVICE_REVOCATION | 双向 | 设备撤销 |

### P2P 信令消息（不持久化）

| 类型 ID | 名称 | 方向 | 说明 |
|---------|------|------|------|
| 50 | P2P_OFFER | 双向 | WebRTC Offer |
| 51 | P2P_ANSWER | 双向 | WebRTC Answer |
| 52 | P2P_ICE_CANDIDATE | 双向 | ICE 候选 |
| 53 | P2P_SESSION_INIT | 双向 | 会话初始化 |
| 54 | P2P_SESSION_CLOSE | 双向 | 会话关闭 |

## 消息路由

服务端收到消息后按以下逻辑路由：

### 1. 临时消息路由

- **已读回执** (`READ_RECEIPT`)：直接转发到目标用户所有设备，不持久化。
- **输入指示器** (`TYPING_INDICATOR`)：
  - 若携带 `channel_id`：广播给频道所有在线成员。
  - 若携带 `recipient_device_id`：转发给目标用户。
  - 均不持久化。

### 2. 频道消息路由

当消息携带 `channel_id` 且 `recipient_device_id` 为空时：

1. **封禁/禁言检查**：检查发送者是否被封禁或禁言。
2. **分配序列号**：通过数据库原子操作分配频道内递增序列号。
3. **持久化**：将消息存入 `messages` 表。
4. **频道广播**（fanout）：
   - 查找频道所有成员。
   - 在线成员：直接通过 WebSocket 推送。
   - 离线成员：存入离线消息队列 + 发送推送通知。

### 3. 单聊消息路由（1:1）

当消息携带 `recipient_device_id` 时：

1. 解析接收者格式：
   - `member_uuid`：投递给该用户的所有设备。
   - `member_uuid:device_id`：投递给特定设备。
2. 若接收者在线：通过 WebSocket 直接推送。
3. 若接收者离线：
   - 存入离线消息队列。
   - 发送推送通知。

### 路由结果

| 结果 | 说明 |
|------|------|
| `Delivered` | 已投递给至少一个在线设备 |
| `Queued` | 接收者离线，消息已入队 |
| `ChannelFanout` | 频道广播完成（含在线投递数和离线入队数） |
| `Forwarded` | 临时消息已转发（不持久化） |
| `Failed` | 路由失败（如被封禁、UUID 无效等） |

## 心跳机制

- 服务端每 **30 秒** 发送 WebSocket Ping 帧。
- 客户端需回复 Pong 帧。
- **90 秒**内未收到 Pong 则断开连接。
- 客户端也可主动发送 Ping，服务端会回复 Pong。
- 每次收到 Pong 时刷新在线状态缓存（TTL 5 分钟）。

## 上线同步

WebSocket 连接建立后，服务端自动执行：

1. 从离线消息队列拉取最多 200 条待投递消息。
2. 逐条通过 WebSocket 推送给客户端。
3. 批量确认已投递的消息（从队列中移除）。

## 加密策略

频道可设置加密策略，影响消息持久化行为：

| 策略 | 说明 |
|------|------|
| `optional` | 可选加密（默认），未加密消息以明文存储 |
| `preferred` | 推荐加密 |
| `required` | 强制加密，未加密消息的明文不会被存储 |

当策略为 `required` 或消息标记为加密时，`plaintext_content` 字段不会被保存。

## 重连策略

建议客户端实现指数退避重连：

1. 首次重连：立即
2. 第二次：1 秒
3. 第三次：2 秒
4. 后续：每次翻倍，最大 30 秒
5. 重连成功后重置计数器
6. 每次重连后服务端会自动推送离线消息
