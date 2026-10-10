# LEDE x86_64 EFI — GitHub Actions

基于 [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) 的定制 x86_64 软路由固件编译仓库，使用 GitHub 托管的 Ubuntu Runner 进行编译。

## 当前编译配置

- **源码**：Lean LEDE `master`
- **目标**：x86_64 Generic，EFI；保留 EXT4/SquashFS 镜像输出配置
- **代理插件**：PassWall2 + 中文界面（首轮默认 Xray 核心）
- **其他**：移除存在缺失依赖的 SSR Plus，保留其它可兼容的插件选择；MosDNS 相关独立选项仍按原配置
- **明确移除**：PassWall 1、SSR Plus、SmartDNS、ZeroTier
- **基础配置**：仓库 `.config`（迁移后的旧配置）；Feeds：`feeds.conf.default`

## 2026-10-11 首次云编译修复

Run #105 用时约 3 小时 41 分，在最终 package/install 阶段报错。原因是 SSR Plus 缺失 shadowsocks-libev-ss-server、vsftpd 和 vsftpd-alt 重复安装，以及 qbittorrent 和 qBittorrent-static 重复安装。现在选择与 LuCI 插件依赖一致的 vsftpd、动态 qbittorrent，移除旧 SSR Plus（PassWall2 + Xray 已成功安装），并在 make defconfig 后检查这些冲突是否再次出现。该修复仍需完整云编译验证。

## 使用方法

1. 打开 [Actions / Build OpenWrt (LEDE)](../../actions/workflows/build-openwrt.yml)。
2. 点击 **Run workflow**，选择分支。
3. **测试分支**：选择 `fix/restore-lede-passwall2-20261010`，启动完整编译但**不会发布 Release**，可在该次运行的 Artifacts 下载测试结果。
4. **正式发布**：PR 合并至 `main` 后，选择 `main` 启动并保持 `publish_release=true`，编译通过、检查产生 EFI 镜像后自动创建 [Release](../../releases)。
5. `Update Checker` 默认仅手动启动；确认构建稳定后再启用其定时检查。检测到新的 LEDE 提交时通过 `repository_dispatch` 触发正式编译。

## 兼容性及故障定位

- 原来的 `ubuntu-20.04`、`git.io/depends-ubuntu-2004`、`::set-output` 和旧 Artifact 上传方式已移除。
- PR 默认运行轻量配置与脚本检查，避免每次改动都消耗数小时编译额度。
- Actions 在 `make defconfig` 后验证 PassWall2、中文包和 Xray 核心仍然启用；如果 Feeds 更新后某个包不再提供，构建会**尽早报错**，而不是发布残缺固件。
- 构建的 Job Summary 会列出原配置中未保留下来的部分软件包选项，便于后续适配；**其它遗留插件并未全部验证兼容**。
- 自动 Release 仅允许 `main` 分支。不会自动清理旧 Releases 或工作流记录，避免诊断信息被删除。
- 每次编译使用最新的上游 LEDE 与 Feeds，可能因上游变化产生新问题。固件升级前先核对包清单、磁盘布局及网络设置，并做好备份。
- GitHub 托管 Job 存在执行时间限制，大量第三方插件可能导致超过编译时限。

## 主要文件

| 文件 | 用途 |
| --- | --- |
| `.github/workflows/build-openwrt.yml` | 构建、验证、Artifact 上传和 Release |
| `.github/workflows/update-checker.yml` | 上游变化检测（默认手动） |
| `.config` | 固件与软件包选择 |
| `feeds.conf.default` | 源码及第三方包仓库 |
| `diy-part1.sh` / `diy-part2.sh` | 可选自定义脚本 |

项目的 Actions 编译方式参考了 [P3TERX/Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt)；LEDE 项目与第三方软件包各自遵循其上游许可证。
