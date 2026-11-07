# 构建 XiaoYu XY-C5 固件

[![构建 XiaoYu XY-C5](https://github.com/sigh233/openwrt-c5/actions/workflows/openwrt-build.yml/badge.svg)](https://github.com/sigh233/openwrt-c5/actions/workflows/openwrt-build.yml)

## 项目说明

本项目提供 GitHub Actions 工作流，用于编译 **XiaoYu XY-C5** (xiaoyu,xy-c5) 的 **OpenWrt 固件**，基于最新版上游代码。

## 功能特性

- **硬件加速**: MT7621 硬件 offload，`kmod-nft-offload` + `firewall4` (自动生成 offload 规则)，MT7530 交换芯片 (DSA)
- **IPv6 支持**: 完整的 IPv6 功能 (ipv6helper, dnsmasq-dhcpv6, 6in4/6rd/6to4)
- **精简**: 所有第三方插件已移除，仅保留核心 LuCI
- **TurboACC**: 硬件 offload + BBR 拥塞控制
- **基础**: 最新版 OpenWrt (official OpenWrt main 分支)

## 构建配置

配置文件: `config/xiaoyu_xy-c5.config`

| 参数 | 值 |
|------|------|
| 源码 | openwrt/openwrt `main` 分支 |
| 目标 | `ramips/mt7621` |
| 设备 | `xiaoyu,xy-c5` |
| 内核 | 最新版本 (6.12) |

## 硬件规格

| 组件 | 规格 |
|------|------|
| 处理器 | MT7621A (Ralink) |
| 内存 | 512MB |
| 闪存 | 32MB |
| 以太网 | 5× 1GE (MT7530 交换芯片) |
| 硬件卸载 | 硬件 (NFT_offload) |
| WiFi | MT7621 (2.4/5GHz) |

## 自动构建

工作流触发条件：

- 手动触发 (`workflow_dispatch`)
- 发布 (`release:upload`)
- 定时任务: 每月 1 号凌晨 (`0 0 1 1 *`)

## 手动触发

请访问 [Actions 标签页](https://github.com/sigh233/openwrt-c5/actions) → **Run workflow** → **Run workflow** 按钮。

## 文件说明

| 文件 | 说明 |
|------|------|
| `config/xiaoyu_xy-c5.config` | OpenWrt 构建配置文件 |
| `.github/workflows/openwrt-build.yml` | GitHub Actions 工作流文件 |
| `diy-part1.sh` / `diy-part2.sh` | 自定义脚本 (当前为空) |
| `README.md` | 项目说明文档 |

## 参考项目

基于 12+ 个项目的研究：

- [Hyy2001X/AutoBuild-Actions-BETA](https://github.com/Hyy2001X/AutoBuild-Actions-BETA) - 配置文件参考
- [P3TERX/Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt) - Workflow 模板参考
- [MrH723/Actions-OpenWrt](https://github.com/MrH723/Actions-OpenWrt) - CI 模板参考
- [speechless22/Actions-OpenWrt](https://github.com/speechless22/Actions-OpenWrt) - 模板参考
