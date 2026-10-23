# PonWrt Changelog

本文件记录 PonWrt 每次发版的实质变更，版本号与 GitHub Release tag 一一对应。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

## [ponwrt-2026102312] - 2026-10-23

修复 `ponwrt-2026102309` 中 Ponwrt 品牌定制未生效的问题（界面仍显示 Vantage），其余内容不变。

### Fixed
- **品牌定制 overlay 路径错误**：vantage 主题的 `header.ut` 由 luci feed 的 `luci.mk` 安装到 `UCODE_LIBRARYDIR=/usr/share/ucode/luci/template/themes/vantage/`，此前 overlay 放在了 `/usr/share/ucode/template/themes/vantage/`（少了 `luci/` 一层），文件从未覆盖主题包内的同名模板，侧栏/顶栏仍渲染上游的 V 形 logo 与 `Vantage` 字样。已移至正确路径。
- **登录页品牌未定制**：登录页由主题的 `sysauth.ut` 渲染（非 `header.ut`），此前未做定制，登录页一直显示 V 形 logo 与 `Vantage`。现已同步改为 P 形 logo（白竖 + 绿碗 + 绿点）与 `Ponwrt`。
- **默认主题检查失效**：`files/etc/uci-defaults/99-vantage-theme` 的存在性检查写成 `/luci-static/vantage/cascade.css`——`/luci-static` 是 URL 前缀而非文件系统路径（实际在 `/www/luci-static` 下），且主题主样式表是 `vantage.css` 而非 `cascade.css`，检查永远失败，默认主题实际从未自动生效（此前界面为 vantage 是手动选择的）。已改为检查 `/www/luci-static/vantage/vantage.css`，刷完即默认 vantage。

### 安装 / 升级
- 与 `ponwrt-2026102309` 相同的产物结构（10 个设备 zip + sha256sums）；如已刷 2026102309 且手动选过 vantage 主题，刷本版后品牌即为 Ponwrt；favicon 变更可能需强制刷新浏览器（Ctrl+F5）清除缓存。
- 升级前请备份现有配置。

## [ponwrt-2026102309] - 2026-10-23

迭代发版，基于 ImmortalWrt 构建，提供 AN7581 / AN7583 两个 target 的固件。相对 `ponwrt-2026102222` 的变更如下。

### Added
- **Ponwrt 品牌标识**：vantage 主题侧栏图标由 V 形改为 P 形（白竖 + 绿碗 + 绿点，沿用主题配色），品牌字由 `Vantage` 改为 `Ponwrt`，favicon 同步替换；通过 `files/` overlay 主题模板实现，明暗模式自动适配。

### Changed
- **去掉「Argon 主题设置」菜单**：argon 主题保留；ImmortalWrt 的 `luci-theme-argon` 内嵌了 argon-config 的菜单定义，故用 `files/usr/share/luci/menu.d/luci-app-argon-config.json`（空对象）覆盖，菜单不再出现。
- **去掉「仪表盘」菜单**：不再安装 `luci-app-vantage`（Vantage 仪表盘应用），vantage 主题本身保留。
- **发布产物修复**：Nokia XG-040G-MF 的 `mf`（NAND 出厂版）与 `mf-ubi`（UBI 版）两个 profile 归一为一个设备 zip，内含 8 个镜像；`ponwrt-2026102222` 中二者因归一化规则未匹配而分裂成两个 zip。

### 安装 / 升级
- 解压对应设备的 zip 后刷入；默认后台地址 192.168.6.1，默认主机名 Ponwrt。
- 升级前请备份现有配置。

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
- **发布产物按设备合并打包**：同一设备的所有镜像合并为一个 zip，命名 `ponwrt-airoha-<target>-<设备>-<构建时间>.zip`；不再一个镜像一个 zip。镜像类型后缀按白名单剥离（含 an7583 的 `initramfs-uImage` / `squashfs-factory-kernel` / `squashfs-factory-rootfs` / `bl31-uboot` / `preloader`），型号自身可含连字符。**同一设备的多 profile 变体归并**：FiberHome HG5585F 的 `ct`/`cu` 与各自 `-usb-sfp` 版、Nokia XG-040G-MD 的 `ubi` 与 `ubi-usb-sfp`、Nokia XG-040G-MF 的 NAND 出厂版与 `ubi` 版，分别并入同一个设备 zip（如 `ponwrt-airoha-an7583-nokia_xg-040g-mf-<构建时间>.zip` 含 8 个镜像：factory-kernel/rootfs、sysupgrade、initramfs-uImage、recovery、preloader、bl31-uboot）。

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
