# LEDE x86_64 EFI — GitHub Actions

基于 [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) 的定制 x86_64 软路由固件编译仓库，使用 GitHub 托管的 Ubuntu Runner 进行编译。

## 当前编译配置

- **源码**：Lean LEDE `master`
- **目标**：x86_64 Generic，EFI；保留 EXT4/SquashFS 镜像输出配置
- **代理插件**：PassWall2 + 中文界面 + 双核心 Xray、sing-box（AnyTLS）
- **其他**：保留 qBittorrent、Aria2、DDNS、WireGuard、Nlbwmon、WOL、TTYD、Argon，暂留 Samba4、DiskMan 以及原有 FRPC / UPnP 配置
- **明确移除**：PassWall 1、SSR Plus、SmartDNS、ZeroTier、网易云解锁、Transmission、aMule、Adbyby、TurboACC、XLNetAcc、多余的独立旧代理程序等
- **基础配置**：仓库 `.config`（迁移后的旧配置）；Feeds：`feeds.conf.default`

## 2026-10-11 首次云编译修复

Run #105 用时约 3 小时 41 分，在最终 package/install 阶段报错。原因是 SSR Plus 缺失 shadowsocks-libev-ss-server、vsftpd 和 vsftpd-alt 重复安装，以及 qbittorrent 和 qBittorrent-static 重复安装。现在选择与 LuCI 插件依赖一致的 vsftpd、动态 qbittorrent，移除旧 SSR Plus（PassWall2 + Xray 已成功安装），并在 make defconfig 后检查这些冲突是否再次出现。第二轮 Run #107 已成功生成 EXT4/SquashFS EFI 固件及 Artifact。

## 2026-10-11 AnyTLS 和精简增强配置（第三轮待验证）

- **AnyTLS**：PassWall2 官方通过 sing-box 支持 AnyTLS。基础核心选为 `luci-app-passwall2_Basic_Core_All`，**同时编入 `sing-box` 和 `xray-core`**。AnyTLS 节点使用 sing-box，其他支持 Xray 的协议仍可选择 Xray；关闭不必要的第三方独立代理组件。保留原有 `iptables` 透明代理模式，不迁移 firewall4。
- **计划移除**：网易云解锁 Go 后端、语言包及旧默认标记，SSR Plus、Adbyby Plus、Shadowsocks-libev 独立 UI、独立 Trojan 系列、V2Ray、旧测速加速/清内存插件，以及 Transmission、aMule、百度网盘 Web、FTP、IPSec/StrongSwan 等不在本次精简配置中的服务。
- **计划保留**：qBittorrent、Aria2、DDNS、WireGuard、Nlbwmon、WOL、TTYD、Argon；Samba4、DiskMan、FRPC、UPnP 暂时不删，避免影响原有存储和远程使用方式。
- **计划新增**：LuCI Statistics（collectd/rrdtool 依赖）、LuCI SQM（CAKE），以及 Tailscale（含 LuCI 管理界面与中文翻译）。Tailscale 需要设备首次启动后自行登录和授权，并不自动加入任何 Tailnet。**SQM 不会自动优化网速，部署后需按实际上下行带宽调参；与流量硬件/软件 offload 同时使用可能导致队列管理失效。**
- **未纳入本轮**：AdGuard Home、Nikki / Momo 和 firewall4。防止 DNS 与防火墙两项架构调整同时发生。
- **清理 Feeds**：删除单独的 `helloworld` Feed，保留 PassWall2 / Passwall packages / packages / luci / routing / telephony。若上游依赖链有变化，以下一次 `make defconfig` 的选项检查结果为准。
- **增加验证**：除 `make defconfig` 检查外，正式构建完成还会校验固件 `manifest` 确实有 sing-box、Xray、Tailscale、PassWall2、Statistics 和 SQM 且没有被移除的旧代理核心。

注意：此第三轮方案尚未执行完整固件编译；Run #107 的成功结论仅适用于当时的 Xray 单核心配置。

## 2026-10-11 正式发布触发策略

- 针对本次成功的 Run #110 配置，合并 PR 时使用带 `[publish-release]` 的 Merge Commit 标题，触发 main 分支的一次正式固件构建和 Release 上传。
- `main` 分支未来普通提交仅执行 preflight，不会每次提交就重新构建或创建 Release。需要自动发布时，在提交标题包含 `[publish-release]`，或手动调度 `publish_release=true`。
- Release 只在完整构建、manifest 软件包核验及 EFI 镜像验证成功后执行。若构建失败则不会发布固件。

## 使用方法

1. 打开 [Actions / Build OpenWrt (LEDE)](../../actions/workflows/build-openwrt.yml)。
2. 点击 **Run workflow**，选择分支。
3. **测试分支**：选择 `fix/restore-lede-passwall2-20261010`，启动完整编译但**不会发布 Release**，可在该次运行的 Artifacts 下载测试结果。
4. **正式发布**：默认仍可在 `main` 手动启动并保持 `publish_release=true`；此外，合并到 `main` 的提交标题若包含 `[publish-release]`，且本次推送涉及固件配置或构建工作流，则自动完整编译、通过校验后创建 [Release](../../releases)。其他普通推送只运行轻量检查，不自动发布。
5. `Update Checker` 默认仅手动启动；确认构建稳定后再启用其定时检查。检测到新的 LEDE 提交时通过 `repository_dispatch` 触发正式编译。

## 兼容性及故障定位

- 原来的 `ubuntu-20.04`、`git.io/depends-ubuntu-2004`、`::set-output` 和旧 Artifact 上传方式已移除。
- PR 默认运行轻量配置与脚本检查，避免每次改动都消耗数小时编译额度。
- Actions 在 `make defconfig` 后验证 PassWall2、中文包、sing-box、Statistics 和 SQM 仍然启用；如果 Feeds 更新后某个包不再提供，构建会**尽早报错**，而不是发布残缺固件。
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
