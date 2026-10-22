# PonWrt Changelog

本文件记录 PonWrt 每次发版的实质变更，版本号与 GitHub Release tag 一一对应。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

## [Unreleased]

### Changed
- 发布产物格式由每个 target 一个 `tar.zst` 改为每个固件镜像一个独立 `zip` 包：文件名与包内固件均按 `ponwrt-airoha-<target>-<profile>-<type>-<构建时间>.zip` 命名，构建时间为北京时间 `YYYYMMDDHHMM`。不再打包 `packages/` 软件包仓库。

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
