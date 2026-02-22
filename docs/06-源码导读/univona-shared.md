# univona-shared

## 概述

`univona-shared` 是所有服务端与客户端共用的 Rust 共享库，负责类型定义、错误体系、RBAC、配置与 Protobuf 生成代码的统一出口。

## 目录结构

```
crates/univona-shared/
├── src/
│   ├── lib.rs
│   ├── auth.rs
│   ├── channel.rs
│   ├── config.rs
│   ├── contacts.rs
│   ├── error.rs
│   ├── pow.rs
│   ├── proto.rs
│   ├── rbac.rs
│   ├── types.rs
│   └── zkp/
└── build.rs
```

## 关键模块

- `auth`：认证结构与 JWT Claims
- `rbac`：角色与权限定义
- `proto`：Protobuf 生成代码导出
- `types`：UUID/分页/时间戳等公共类型

## 相关文档

- [Protobuf 协议](../02-系统架构/Protobuf协议.md)
- [权限控制](../03-功能模块/权限控制.md)
