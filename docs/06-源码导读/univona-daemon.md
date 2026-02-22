# univona-daemon

## 概述

社区服务端核心，负责单社区消息路由、频道管理与媒体存储。可独立部署（Mode B），也可注册到 Admin Server（Mode C）。

## 关键入口

- `src/main.rs`：启动与配置加载
- `src/app.rs`：路由注册
- `src/state.rs`：应用状态

## 主要模块

- `api/`：认证、频道、消息、媒体、离线中继
- `ws/`：WebSocket 实时通信
- `relay/`：离线消息
- `media/`：媒体存储
- `rbac/`：权限系统

## 相关文档

- [Daemon API](../04-API参考/Daemon-API.md)
- [Daemon 配置参考](../08-配置参考/Daemon配置.md)
- [数据库设计](../05-数据库设计/Daemon表结构.md)
