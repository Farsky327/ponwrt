# PonWrt Changelog

本文件记录 PonWrt 每次发版的实质变更，版本号与 GitHub Release tag 一一对应。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

## [ponwrt-2026102222] - 2026-10-22

迭代发版，基于 ImmortalWrt 构建，提供 AN7581 / AN7583 两个 target 的固件。

### Added
- **vantage 主题**：`feeds.conf.default` 挂载 `anothaDev/luci-theme-vantage`（提供 `luci-theme-vantage` v1.0.9 与 `luci-app-vantage` 仪表盘），并通过 `files/etc/uci-defaults/99-vantage-theme` 设为默认主题（`luci.main.mediaurlbase=/luci-static/vantage`），刷完即生效。
- **IPTV 支持**：`luci-app-iptv`（来自已挂载的 `pbs05/openwrt-pon-userspace` feed）。
- **USB 支持**：`kmod-usb-net`、`kmod-usb-net-rndis`、`kmod-usb-net-cdc-ether`（USB 网卡）、`kmod-usb-storage`（USB 存储）、`usbutils`、`udev`。
- **LuCI 应用**：`luci-app-wol`（网络唤醒）、`luci-app-zerotier`、`luci-app-arpbind`（IP/MAC 绑定）。
- **中文界面**：`luci-i18n-{upnp,wol,zerotier,iptv,arpbind}-zh-cn`。

### Changed
- **默认后台地址改为 `192.168.6.1`**（`package/base-files/files/bin/config_generate`，DHCP 池随网段自动推导为 192.168.6.x）。从 192.168.1.1 升级后需改用新地址访问。
- **默认主机名改为 `Ponwrt`**（`config_generate`，原 `ponwrt`）。
- **发布产物按型号合并打包**：同一型号（profile）的 `initramfs-recovery` 与 `squashfs-sysupgrade` 镜像合并为一个 zip，命名 `ponwrt-airoha-<target>-<型号>-<构建时间>.zip`；不再一个镜像一个 zip（本次构建 zip 数量由 35 个减为 18 个左右）。镜像类型后缀按白名单剥离，型号自身可含连字符（如 `fiberhome_hg5585f-cu-usb-sfp`）不受影响。

### 安装 / 升级
- 解压对应型号的 zip，其中 `*-initramfs-recovery.itb` 用于 recovery 刷入，`*-squashfs-sysupgrade.itb` 用于 sysupgrade 升级。
- 升级前请备份现有配置；注意后台地址已变为 192.168.6.1。

## [ponwrt-2026102217] - 2026-10-22

迭代发版，基于 ImmortalWrt 构建，提供 AN7581 / AN7583 两个 target 的固件。

### Added
- **WireGuard 支持**：在 `configs/release.config` 模块化叠加层新增 `kmod-wireguard`（内核模块）、`wireguard-tools`（用户态 `wg` / `wg-quick`）、`luci-proto-wireguard`（LuCI「网络 → 接口」中的 WireGuard 协议），两个 target 构建自动并入。

### Changed
- 发布产物格式由每个 target 一个 `tar.zst` 改为每个固件镜像一个独立 `zip` 包：文件名与包内固件均按 `ponwrt-airoha-<target>-<profile>-<type>-<构建时间>.zip` 命名，构建时间为北京时间 `YYYYMMDDHHMM`。不再打包 `packages/` 软件包仓库。
- 清理 `configs/an7581.config` / `configs/an7583.config` 中被注释掉（未启用）的软件包行，配置文件更精简、可读，已启用的包保持不变。

### 安装 / 升级
- 固件以 `zip` 压缩分发，解压后按对应设备刷入。
- 升级前请备份现有配置。

## [ponwrt-2026102213] - 2026-10-22

首个发布版本，基于 ImmortalWrt 构建，提供 AN7581 / AN7583 两个 target 的固件。

### Fixed
- **修复 libffi 缺失导致的 release 构建失败**
  - 根因：上游 ImmortalWrt 在 2026-10-19 将 `libffi` 包从 packages feed 迁移到核心仓库，ponwrt 未同步该变动，导致 `python3` 编译阶段找不到 libffi 而失败，整个 release（an7581 / an7583）无法出包。
  - 修复：从上游 `immortalwrt/immortalwrt@32bec3df41` 原样取回 `package/libs/libffi`（libffi v3.4.7），提交 `590ffa89c9`。
  - 说明：CPython 自 3.12 起已移除 `--with-system-ffi` 选项，非 Windows 平台统一使用系统 libffi，因此只能补回 `libffi` 包，不能靠改 python3 编译参数绕过。

### 安装 / 升级
- 固件以 `tar.zst` 压缩分发，解压后按对应设备刷入。
- 升级前请备份现有配置。
