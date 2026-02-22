# univona-client-rust

## 概述

客户端 Rust 核心库负责身份、加密、存储与 P2P 网络层。Flutter 侧通过 flutter_rust_bridge 调用这些能力。

## 主要 Crates

| Crate | 说明 |
|-------|------|
| `univona-core` | 身份、加密、会话与本地存储 |
| `univona-p2p` | libp2p 网络层与 WebRTC 信令 |
| `univona-bot-sdk` | Bot 开发 SDK |

## 关键模块（univona-core）

- `identity/`：BIP-39 助记词、Ed25519 身份
- `crypto/`：X3DH + Double Ratchet
- `storage/`：SQLCipher 本地数据库
- `search/`：FTS5 全文搜索

## 相关文档

- [端到端加密](../03-功能模块/端到端加密.md)
- [P2P 通信](../03-功能模块/P2P通信.md)
- [Flutter 客户端](./univona-client-flutter.md)
