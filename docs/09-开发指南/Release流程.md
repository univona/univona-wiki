# Release 流程

## 触发方式

- 创建并推送 tag：`git tag vX.Y.Z && git push origin vX.Y.Z`
- 触发 `release.yml`（仅 tag push）

## 主要流程

1. `build-macos` job 生成 DMG + ZIP
2. `build-android` job 生成 APK
3. `build-windows` job 生成 Windows exe
4. `release` job 下载所有 artifacts 并计算 SHA256
5. 创建 GitHub Release（`univona/univona-client`）
6. 调用 `POST /api/v1/admin/releases` 注册版本到 admin-server

## 关键依赖

- `UNIVONA_ADMIN_API_KEY`：Release 时注册版本到 admin-server（可选）
- `GH_PAT`：访问 org 私有仓库并回报 commit status
