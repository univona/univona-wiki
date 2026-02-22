# univona-client-flutter

## 概述

Flutter UI 负责用户界面、状态管理、网络调用与本地缓存，与 Rust 核心通过 flutter_rust_bridge 进行通信。

## 目录结构

```
flutter/
├── lib/
│   ├── app/        # 路由与主题
│   ├── models/     # 数据模型 (freezed)
│   ├── providers/  # 状态管理 (Riverpod)
│   ├── screens/    # UI 页面
│   ├── services/   # REST/WS 服务
│   └── widgets/    # 可复用组件
└── rust/           # FFI 入口
```

## 关键模块

- `services/`：AuthService、ChannelService、MessagingService
- `providers/`：连接与聊天状态管理
- `screens/`：会话列表、聊天、联系人、设置等

## 相关文档

- [客户端 Rust 核心](./univona-client-rust.md)
- [API 总览](../04-API参考/API总览.md)
- [消息系统](../03-功能模块/消息系统.md)
