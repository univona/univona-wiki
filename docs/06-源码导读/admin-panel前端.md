# admin-panel 前端

## 概述

Admin Server 内置的管理面板前端位于 `univona-admin-server/admin-panel/`，使用 React + TypeScript + Ant Design 构建，并通过 `rust-embed` 打包进二进制发布。

## 技术栈

- React 18
- TypeScript
- Ant Design 5
- React Router 6
- Vite 构建

## 主要页面

- 仪表板
- KYC 审批
- 社区目录
- 节点管理
- 内容监听
- 举报管理
- 权限管理
- 审计日志

## 构建方式

```bash
cd admin-panel
npm install
npm run build
```

构建产物输出到 `admin-panel-dist/`，由 Rust 二进制嵌入并对外提供 `/admin` 路由。

## 相关文档

- [univona-admin-server](./univona-admin-server.md)
- [Admin Server API](../04-API参考/Admin-Server-API.md)
