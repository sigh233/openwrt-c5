# OpenWrt Build for XiaoYu XY-C5

[![Build OpenWrt](https://github.com/sigh233/openwrt-c5/actions/workflows/openwrt-build.yml/badge.svg)](https://github.com/sigh233/openwrt-c5/actions/workflows/openwrt-build.yml)

## Overview

This repository provides GitHub Actions workflows to compile **OpenWrt firmware for XiaoYu XY-C5** (xiaoyu,xy-c5) based on the latest upstream source.

## Features

- **Hardware Acceleration**: MT7621 hardware offloading via `kmod-nft-offload` + `firewall4` (auto-generated offload rules), MT7530 switch (DSA)
- **IPv6**: Full IPv6 support (ipv6helper, dnsmasq-dhcpv6, 6in4/6rd/6to4)
- **Minimal**: All third-party plugins removed; only core LuCI remains
- **TurboACC**: Hardware offload + BBR Congestion Control
- **Base**: Latest OpenWrt (official OpenWrt main branch), kernel 6.12

## Build Configuration

Located in `config/xiaoyu_xy-c5.config`:

| Component | Version |
|-----------|---------|
| Source | openwrt/openwrt `main` branch |
| Target | `ramips/mt7621` |
| Device | `xiaoyu,xy-c5` |
| Kernel | Latest (6.12) |

## Hardware

| Component | Spec |
|-----------|------|
| SoC | MT7621A (Ralink) |
| RAM | 512MB |
| Flash | 32MB |
| Ethernet | 5× 1GE (MT7530 switch) |
| Offload | Hardware (NFT_offload) |
| WiFi | MT7621 (2.4/5GHz) |

## Automated Builds

The workflow runs on:
- Manual trigger (`workflow_dispatch`)
- On release (`release:upload`)
- Monthly cron: `0 0 1 1 *` (Jan 1, 00:00 UTC)

## Manual Trigger

To build now, use the [Actions tab](https://github.com/sigh233/openwrt-c5/actions) → **Run workflow** → **Run workflow** button.

## Files

| File | Purpose |
|------|---------|
| `config/xiaoyu_xy-c5.config` | OpenWrt build config |
| `.github/workflows/openwrt-build.yml` | GitHub Actions workflow |
| `diy-part1.sh`, `diy-part2.sh` | Custom scripts (no-ops) |

## Reference

Based on research of 12+ repositories:
- [Hyy2001X/AutoBuild-Actions-BETA](https://github.com/Hyy2001X/AutoBuild-Actions-BETA) - Config
- [P3TERX/Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt) - Workflow Template
- [MrH723/Actions-OpenWrt](https://github.com/MrH723/Actions-OpenWrt) - CI templates
- [speechless22/Actions-OpenWrt](https://github.com/speechless22/Actions-OpenWrt) - Template
