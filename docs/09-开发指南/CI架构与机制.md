# CI/CD 架构与机制（Fork Runner）

## 概述

由于 univona GitHub Organization 的 Actions 因 billing 问题被暂停，采用个人 fork 作为 CI Runner 的模式运行所有工作流，并将状态回报到 org 主仓库。

## 架构拓扑

```text
univona/univona-daemon (org, private)     univona/univona-client (org, private)
         │                                          │
         │  fork                                    │  fork
         ▼                                          ▼
yellowpeachxgp/univona-daemon (fork)     yellowpeachxgp/univona-client (fork)
  └─ CI 在这里运行                         └─ CI 在这里运行
  └─ 结果回报到 org commit status           └─ 结果回报到 org commit status
```

## 流程概览

```text
开发者提交 → org 主仓库
   └─ Sync Upstream → fork 同步
       └─ fork 触发 CI/Build
           └─ 回报 commit status → org
```

## 本地 Git Remote 约定

- `origin` → `univona/<repo>`（org 主仓库）
- `ci` → `yellowpeachxgp/<repo>`（个人 fork，CI runner）

推送策略：

- 功能代码：`git push origin main`
- CI 相关：`git push ci main`
- 两边内容通过 `Sync Upstream` 工作流自动保持一致

## 工作流清单

univona-daemon：

| 文件 | 名称 | 触发条件 | 功能 |
| --- | --- | --- | --- |
| `ci.yml` | CI | push/PR to main, workflow_dispatch | Clippy + Test + SQLx Check |
| `sync.yml` | Sync Upstream | cron 每 2 小时, workflow_dispatch | 从 org 同步到 fork |

univona-client：

| 文件 | 名称 | 触发条件 | 功能 |
| --- | --- | --- | --- |
| `build-macos.yml` | Build macOS | push/PR to main, tags `v*`, workflow_dispatch | 构建 macOS `.app` |
| `build-android.yml` | Build Android | push/PR to main, tags `v*`, workflow_dispatch | 构建 Android APK |
| `build-windows.yml` | Build Windows | push/PR to main, tags `v*`, workflow_dispatch | 构建 Windows exe |
| `release.yml` | Release | push tags `v*` | 构建 + 创建 GitHub Release + 注册到 admin-server |
| `sync.yml` | Sync Upstream | cron 每 2 小时, workflow_dispatch | 从 org 同步到 fork |

## 关键机制

### 1. 上游同步（Sync Upstream）

使用 GitHub 内置的 `merge-upstream` REST API，不直接访问上游私有仓库：

```bash
RESPONSE=$(gh api repos/${{ github.repository }}/merge-upstream \
  -f branch=main \
  --jq '.merge_type // "none"' 2>&1) || true
```

优势：不需要 GH_PAT 对上游仓库的访问权限，GitHub 自动基于 fork 关系处理。同步后若有变更，自动触发对应 CI/Build 工作流。

### 2. 状态回报（Report Status to Upstream）

每个 CI 工作流末尾通过 GitHub Commit Status API 将结果写回 org 仓库：

```bash
curl -sf -X POST \
  -H "Authorization: token ${{ secrets.GH_PAT }}" \
  "https://api.github.com/repos/univona/<repo>/statuses/$COMMIT_SHA" \
  -d '{"state":"success/failure","target_url":"<fork actions URL>","context":"CI (fork)"}'
```

效果：org 仓库的 commit 页面会显示 fork CI 的状态徽章和链接。

### 3. 私有依赖解析

所有仓库依赖 `univona/univona`（共享库，私有）。CI 中的处理流程：

```bash
# 1. Checkout 共享库到工作区内
- uses: actions/checkout@v4
  with:
    repository: univona/univona
    token: ${{ secrets.GH_PAT }}
    path: _univona

# 2. 配置 Git 认证（Cargo git 依赖解析用）
- run: git config --global url."https://x-access-token:${{ secrets.GH_PAT }}@github.com/".insteadOf "https://github.com/"

# 3. Patch 路径覆盖（避免网络 fetch，使用本地 checkout）
# Daemon: sed 替换 Cargo.toml 中的相对路径
- run: sed -i 's|path = "../univona/crates/univona-shared"|path = "_univona/crates/univona-shared"|' Cargo.toml

# Client: 追加 [patch] 段
- run: |
    echo '[patch."https://github.com/univona/univona.git"]'
    echo 'univona-shared = { path = "_univona/crates/univona-shared" }' >> Cargo.toml
```

### 4. Concurrency 控制

同一分支或 tag 的新 push 会自动取消旧的运行：

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### 5. 所需 GitHub Secrets

| Secret 名称 | 用途 | 配置位置 |
| --- | --- | --- |
| `GH_PAT` | 访问 org 私有仓库 + 回报 commit status | fork repo settings |
| `GITHUB_TOKEN` | fork 自身操作 + sync | 自动提供 |
| `UNIVONA_ADMIN_API_KEY` | Release 时注册版本到 admin-server | fork repo settings（可选） |

### 6. 平台特定处理

| 平台 | 特殊依赖 | 说明 |
| --- | --- | --- |
| macOS | `brew install protobuf` | `protoc` 编译器 |
| Android | `rustup target add aarch64/armv7/x86_64`, Java 17 | 三架构交叉编译 |
| Windows | vcpkg OpenSSL + `OPENSSL_NO_VENDOR=1` | 避免从源码编译 OpenSSL |
| Ubuntu（daemon） | PostgreSQL 16, SQLx CLI | 数据库迁移 + 查询检查 |
