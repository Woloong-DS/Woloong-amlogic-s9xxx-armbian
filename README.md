# Woloong Amlogic S9xxx Armbian

[中文说明](README.md) | [English](README.en.md) | 上游项目：[ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian)

本仓库用于构建 **Woloong OS** 的 Armbian 系统镜像（面向 Amlogic S905x3 / HK1-BOX 等电视盒子），fork 自 [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian)。

- **品牌**：登录 MOTD 大字 logo 为 `Woloong OS`，系统信息中的 `VENDOR` / `VENDORPRETTYNAME` 同样为 `Woloong OS`（保留 Armbian 官方 logo 形态，只改文字）。
- **内核**：来自自建内核仓库 [Woloong-DS/Woloong-kernel](https://github.com/Woloong-DS/Woloong-kernel)，版本形如 `6.18.55-Woloong`，内置 Klipper / CAN 所需驱动。
- **节奏**：每月 8 日、23 日（UTC 20:00）自动构建镜像，与内核构建（1 日、16 日）**错开半个月**，这样每次构建都能用上刚发布的新内核。

## 目录

- [定制清单](#定制清单)
- [品牌是怎么生效的](#品牌是怎么生效的)
- [自动化构建](#自动化构建)
- [构建产物](#构建产物)
- [烧录与使用](#烧录与使用)
- [实机验证](#实机验证)
- [同步上游](#同步上游)
- [排障 FAQ](#排障-faq)
- [文档与许可证](#文档与许可证)

## 定制清单

相对上游的改动（`git diff upstream/main`）：

| 文件 | 类型 | 作用 |
|------|------|------|
| `.github/workflows/build-armbian-arm64-server-image.yml` | 改 | 默认参数改 Woloong 值、定时 8/23 日、全部 inputs 加 `\|\|` 兜底、`permissions: contents: write`、action 指向本仓库、overlay 预检步骤 |
| `.github/workflows/build-armbian-using-official-image.yml` | 改 | `kernel_repo` 默认 `Woloong-DS/Woloong-kernel` |
| `.github/workflows/build-armbian-using-releases-files.yml` | 改 | 同上 |
| `.github/workflows/build-armbian-x86-server-image.yml` | 改 | 同上 |
| `build-armbian/armbian-files/common-files/etc/default/armbian-motd` | 增 | `VENDOR="Woloong OS"`、`MOTD_DISABLE="tips"` |
| `build-armbian/armbian-files/common-files/usr/local/bin/woloong-motd-fix` | 增 | 幂等品牌修复脚本（MOTD 开关 + 两个 release 文件） |
| `build-armbian/armbian-files/common-files/etc/apt/apt.conf.d/99-woloong-branding` | 增 | `DPkg::Post-Invoke` 在每次 apt/dpkg 后执行上面的修复脚本 |
| `build-armbian/armbian-files/common-files/usr/sbin/armbian-fix` | 改 | 首次启动写 `/etc/armbian-release`、`/etc/armbian-image-release` 时输出 `Woloong OS` |
| `build-armbian/armbian-files/common-files/etc/custom_service/start_service.sh` | 改 | 开机调用品牌修复脚本 |
| `rebuild` | 改 | 打包阶段改写 `VENDOR`/`VENDORPRETTYNAME`；overlay 拷贝日志与预检 |

## 品牌是怎么生效的

登录 banner 由 `/etc/update-motd.d/10-armbian-header` 渲染，它依次 source `/etc/armbian-image-release`、`/etc/armbian-release`、`/etc/default/armbian-motd`，再以 `figlet -f small` 打印 `VENDORPRETTYNAME`（优先）或 `VENDOR`。为了让定制在**任何一次系统更新之后**都还在，仓库做了三层保险：

1. **打包期**：`rebuild` 改写 `/etc/armbian-release` 的 `VENDOR`/`VENDORPRETTYNAME`；
2. **首次启动**：`armbian-fix` 再次写入两个 release 文件（上游这里会写回 `Armbian OS`，本仓库已改为 `Woloong OS`）；
3. **运行期自愈**：`woloong-motd-fix` 由 apt `Post-Invoke` 钩子与开机 `start_service.sh` 调用，发现品牌被改回去就自动修回（幂等）。

> 为什么需要第 3 层：`armbian-fix` 会在首次启动/内核升级流程中重写 `/etc/armbian-release`，仅靠打包期的改写会被覆盖。

## 自动化构建

主工作流：`Build Armbian arm64 server image`

| 触发方式 | 说明 |
|----------|------|
| 定时 `schedule` | `0 20 8,23 * *`（UTC）= 北京时间次日 04:00，每月两次 |
| 手动 `workflow_dispatch` | Actions 页面 Run workflow，参数见下表 |
| `repository_dispatch` | 供外部（如内核仓库）用 API 触发 |

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `set_release` | `bookworm` | 发行版（Debian 12） |
| `armbian_board` | `s905x3-hk1` | 目标盒子 |
| `armbian_kernel` | `6.18.y` | 内核分支 |
| `kernel_repo` | `Woloong-DS/Woloong-kernel` | 内核来源仓库 |
| `kernel_usage` | `stable` | 使用内核仓库的 `kernel_stable` Release |
| `armbian_fstype` | `ext4` | 根分区文件系统 |
| `armbian_edition` | `server` | 镜像类型 |
| `builder_name` | `woloong` | 镜像名中的构建者标记 |
| `config_board` | `odroidn2` | 复用的基础 Armbian 板级配置 |

构建步骤：

1. **Checkout**
2. **Verify Woloong overlay in action checkout** —— 预检 `common-files` 里的品牌文件是否随 action 一起被检出，缺了立即失败（避免白跑一小时编译）
3. **Initialize the build environment** —— 清理 runner、装依赖、统一 UTC
4. **Create virtual disk for extended storage** —— `/builder` 扩展存储
5. **Download Armbian source code**
6. **Compile Armbian [bookworm]**
7. **Organize output files and reclaim disk space**
8. **Upload Armbian image to Release** —— 上传到 `Armbian_<release>_arm64_<edition>_<YYYY.MM>`
9. **Rebuild Armbian for target boards** —— 调**本仓库**的 `build_target: armbian` action（`uses: Woloong-DS/Woloong-amlogic-s9xxx-armbian@main`），下载 Woloong 内核并套用 overlay
10. **Upload rebuilt images to Release**

## 构建产物

Release 标签形如 `Armbian_bookworm_arm64_server_2026.10`，资产：

| 资产 | 说明 |
|------|------|
| `Armbian_26.11.0_amlogic_s905x3-hk1_bookworm_6.18.55_server_2026.10.12.img.gz` | 可直接烧录到 HK1-BOX 的镜像 |
| 基础 rootfs / img | 中间产物，便于复用与手动 rebuild |

## 烧录与使用

```bash
# 1) 烧录：balenaEtcher / rufus 写入 TF 卡或 eMMC
# 2) 首次启动：按 Armbian 流程创建用户（本机设备用户为 pi，root 密码 1234）

# 3) 写入 eMMC
sudo armbian-install

# 4) 升级内核到本仓库最新版
sudo armbian-update -k 6.18.y -u stable -r Woloong-DS/Woloong-kernel
```

## 实机验证

已在 **HK1-BOX** 实机上刷入并核对：

| 项目 | 结果 |
|------|------|
| 系统 | Armbian OS 26.11.0 / Debian 12 bookworm，`BOARD="S905x3-Hk1"` |
| 内核 | `6.18.55-Woloong` |
| 登录 logo | figlet 大字 `Woloong OS` + `v26.11.0 for Aml.S905x3 running Armbian Linux 6.18.55-Woloong` |
| 品牌字段 | `/etc/armbian-release`、`/etc/armbian-image-release`、`/etc/default/armbian-motd` 均为 `Woloong OS` |
| CAN 驱动 | `gs_usb.ko` 已在 `/lib/modules/6.18.55-Woloong/kernel/drivers/net/can/usb/` |
| 存储 / 内存 | eMMC 29.1G（根分区 28G，占用 8%）；内存 3.6Gi（空闲 2.9Gi） |
| 服务 / 网络 | 无失败单元；wlan0 已联网，eth0 未插 |

## 同步上游

```bash
git fetch upstream
git merge upstream/main
```

注意：

- 上游 action 里硬编码了自身 `/home/runner/work/_actions/ophub/...` 路径，并且 `common-files` 使用相对路径（在 `cd build/output/images` 之后会失效）。因此工作流必须 `uses:` **本仓库**，而不是 `ophub/amlogic-s9xxx-armbian`。
- 冲突点通常在 `rebuild`、`.github/workflows/*`、`common-files/usr/sbin/armbian-fix`，解决原则是保留 Woloong 品牌相关改动。
- 推送前请确保是完整克隆（`--depth=1` / `--filter=blob:none` 的克隆会出现 `did not receive expected object ... remote unpack failed`）。

## 排障 FAQ

**Q：上传 Release 报 `Resource not accessible by integration`？**
GITHUB_TOKEN 没有写权限。所有 workflow 都必须声明 `permissions: contents: write`（本仓库已加），同时确认仓库 `Settings → Actions → General → Workflow permissions` 允许读写。

**Q：`ln: failed to create symbolic link '/home/runner/work/_actions/ophub/.../build'`？**
上游把 action 路径写死了。本仓库改为按 `${{ github.repository }}` 动态解析 action 目录。

**Q：`cp: target directory 'build-armbian/armbian-files/common-files/': No such file or directory`？**
上游 `common-files` 用相对路径，在 `cd build/output/images` 之后失效。解决办法是让 workflow `uses:` 本仓库（overlay 随 action 一起被检出），并由"Verify Woloong overlay"步骤提前拦住。

**Q：日志里 `Extension search Directory does not exist: /builder/build/userpatches/extensions`？**
工作流里需要 `mkdir -p /builder/build/userpatches/extensions`（本仓库已恢复）。

**Q：改了仓库但镜像里没生效？**
镜像构建用的是 `uses: Woloong-DS/Woloong-amlogic-s9xxx-armbian@main`，必须先把改动推到 `main`，再重新触发构建。

## 文档与许可证

- 上游英文文档（原 `README.md`，保留原始内容）：[`README.ophub.md`](README.ophub.md)
- 上游中文文档：[`README.cn.md`](README.cn.md)、日文：[`README.ja.md`](README.ja.md)
- 本仓库英文说明：[`README.en.md`](README.en.md)
- 内核编译/使用说明：[`compile-kernel/`](compile-kernel)
- 许可证：见 [`LICENSE`](LICENSE)（沿用上游）
