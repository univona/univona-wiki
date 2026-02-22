# univona-admin-server

## 概述

官方中枢服务端，内置增强版 Panel + Daemon，并提供目录服务、KYC、举报、内容监控等中枢能力。代码量最大，API 模块最丰富。

## 关键入口

- `src/main.rs`：应用入口、组件初始化、后台任务启动
- `src/app.rs`：路由注册（100+ 端点）
- `src/state.rs`：AppState 依赖容器

## 主要模块

- `api/`：业务端点（认证、频道、消息、KYC、目录、P2P 等）
- `ws/`：WebSocket 实时消息与路由
- `rbac/`：权限系统
- `media/`：媒体上传、存储、缩略图
- `p2p/`：P2P 会话管理与信令
- `push/`：推送通知（FCM/APNs/UnifiedPush）
- `tasks/`：离线消息/审计清理

## 相关文档

- [API 总览](../04-API参考/API总览.md)
- [Admin Server 配置参考](../08-配置参考/Admin-Server配置.md)
- [数据库设计](../05-数据库设计/Admin-Server表结构.md)
