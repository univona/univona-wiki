# Protobuf 协议

## 概述

Univona 的实时消息与关键业务事件使用 Protocol Buffers v3 编码。协议定义位于 `univona/protocol/univona/v1/`，服务端通过 `prost-build` 生成 Rust 代码，客户端通过 `protoc` 生成 Dart 代码。

## 协议文件列表

| 文件 | 说明 |
|------|------|
| `envelope.proto` | 消息信封与 `MessageType` 枚举 |
| `content.proto` | 消息内容结构（文本/附件等） |
| `identity.proto` | 身份协议与设备注册 |
| `receipt.proto` | 回执协议 |
| `group.proto` | 群组/频道相关协议 |
| `system.proto` | 系统消息与服务器通知 |
| `channel_policy.proto` | 频道策略（加密/权限） |
| `p2p.proto` | P2P 信令协议 |

## Envelope 核心结构

```protobuf
message Envelope {
  string message_id = 1;           // UUIDv7
  string sender_device_id = 2;
  string recipient_device_id = 3;  // 空 = 频道消息
  string channel_id = 4;
  MessageType message_type = 5;
  bytes encrypted_content = 6;     // 密文
  int64 server_timestamp = 7;
  int64 client_timestamp = 8;
  uint32 protocol_version = 9;
  bool is_encrypted = 10;
}
```

## 生成方式

```bash
# Rust 侧（build.rs 自动执行）
cargo build -p univona-shared

# 可选：使用 buf 工具链
cd protocol && buf generate
```

## 相关文档

- [通信协议](./通信协议.md)
- [WebSocket 协议](../04-API参考/WebSocket协议.md)
- [端到端加密](../03-功能模块/端到端加密.md)
