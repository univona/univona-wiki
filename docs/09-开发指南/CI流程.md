# CI/CD 流程

本页为 CI/CD 的导航入口，详细内容拆分为两部分：

- [CI/CD 架构与机制](CI架构与机制.md)
- [Release 流程](Release流程.md)
- [CI 状态](CI状态.md)

## 快速摘要

- 采用 Fork Runner 架构，解决 org Actions billing 暂停问题
- 所有工作流在个人 fork 执行，并回报 org commit status
- `Sync Upstream` 负责保持 fork 与 org 同步
- Release 通过 tag 触发，产物发布到 GitHub Release 并注册到 admin-server
