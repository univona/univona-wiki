# univona-panel

## 概述

轻量级管理面板服务，提供独立的 Web UI 来管理多个 Daemon 节点。可独立部署，也可嵌入 Admin Server。

## 目录结构

```
univona-panel/
├── src/                 # Rust 后端
├── admin-panel/         # React 前端 SPA
├── admin-panel-dist/    # 构建产物
└── migrations/
```

## 主要模块

- `daemon_manager.rs`：Daemon 节点管理与代理
- `api/`：Daemons/Channels/ContentMonitor/Health
- `admin-panel/`：React + MUI 前端

## 相关文档

- [Admin Panel 前端](./admin-panel前端.md)
- [Daemon-Panel-AdminServer](../02-系统架构/Daemon-Panel-AdminServer.md)
