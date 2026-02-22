# P2P 通信

## 概述

P2P 模块基于 libp2p 与 WebRTC，实现客户端之间的点对点通信能力，支持 NAT 穿透与离线中继回退。

## 核心能力

- 节点发现：mDNS + Kademlia DHT
- NAT 穿透：STUN/TURN + ICE 候选交换
- 信令转发：通过 Admin Server 的 P2P 信令服务中继
- 传输层：QUIC / WebRTC DataChannel

## 典型流程

```mermaid
sequenceDiagram
    participant A as Client A
    participant S as Signaling Server
    participant B as Client B

    A->>S: P2P_SESSION_INIT
    S-->>B: P2P_SESSION_INIT
    A->>S: P2P_OFFER
    S-->>B: P2P_OFFER
    B->>S: P2P_ANSWER
    S-->>A: P2P_ANSWER
    A<->>B: ICE Candidate 交换
    A<->>B: P2P 直连
```

## 相关文档

- [通信协议](../02-系统架构/通信协议.md)
- [WebSocket 协议](../04-API参考/WebSocket协议.md)
- [Admin Server API](../04-API参考/Admin-Server-API.md)
