# 小娱 XY-C5 OpenWrt 精简固件

针对 **小娱 XY-C5**（`ramips / mt7621`，设备标识 `xiaoyu_xy-c5`）自动编译精简 OpenWrt 固件。

固件产物发布在本仓库的 Releases 中，下载 `*squashfs-sysupgrade.bin` 即可。

## 设计目标

| 项目 | 说明 |
| --- | --- |
| 精简 | 只保留路由器基础功能，不塞入多余插件；包集合 = 官方镜像默认集 + 网页管理 + IPv6 隧道 |
| 基础功能齐全 | 网络 / 防火墙 / DHCP / DNS / PPPoE / SSH / 网页管理 |
| IPv6 保留 | `odhcp6c`、`odhcpd-ipv6only`（上游默认）+ `6in4` / `6rd` / `6to4` 隧道 |
| 硬件加速保留 | `kmod-nft-offload` + `kmod-nf-flow` + `kmod-nf-conntrack-netlink`（MT7621 nftables 流量卸载） |

## 关键取舍（相对旧版配置的改动）

旧版配置存在几处与官方镜像不一致的强制包，本次已全部移除：

1. **`uboot-envtools`（已删除）**
   该包会带入 `/etc/board.d/05_fw_defaults`、`/etc/uci-defaults/30_uboot-envtools`、`/usr/sbin/fw_loadenv`，
   并被 `/lib/preinit/82_config_generate` 在 preinit 极早期调用 `fw_printenv` 读取 U-Boot 环境变量。
   官方镜像中完全没有这个包。设备经 Breed 刷入时 U-Boot 环境变量布局与原生 U-Boot 不一致，
   这条早期启动分支属于「官方镜像中不存在」的额外路径。

2. **`opkg`（已删除）**
   上游 `main` 已全面切换到 `apk` 包管理器，强制装 `opkg` 会与 `apk-mbedtls` 并存，
   并额外引入 `/etc/opkg.conf` 与 `/etc/uci-defaults/20_migrate-feeds` 迁移钩子。

3. **`ipv6helper`（已删除）**
   该包在新版上游已被移除，写了也只会被 `make defconfig` 丢弃。

4. **构建源固定到官方快照同源 commit**
   官方快照中 `xiaoyu_xy-c5` 镜像可正常启动，对应 `openwrt/openwrt` commit
   `dd72c4c198330e208c2357625693a03996cae741`。工作流默认检出该 commit，
   使产物与已验证可启动的官方镜像同源（拉取失败时自动回退 `main` 并打印实际 commit）。

5. **构建权限（上游已知问题 #9545）**
   上游构建对源码树 unix 权限敏感，`umask` 不是 `0022 / 0002` 时会「编译成功但设备启动挂死」。
   工作流每个构建步骤都显式 `umask 0022`。

## 仓库结构

```
.github/workflows/build.yml      GitHub Actions 构建流程
config/xiaoyu_xy-c5.config       精简配置（只写增量，其余由 make defconfig 按上游默认补全）
```

> 刻意不再使用 `diy-part1.sh` / `diy-part2.sh`：它们原本只是 `echo`，不产生任何实际改动，
> 保留只会让人误以为构建过程有额外定制。

## 构建流程

`workflow_dispatch` 手动触发，或在 `config/**`、`.github/workflows/build.yml` 变更时自动触发。

1. 安装构建依赖、释放磁盘空间
2. 签出本仓库
3. 克隆上游 `openwrt/openwrt`（优先固定 commit，失败回退 `main`）
4. `./scripts/feeds update -a` + `install -a`
5. 载入 `config/xiaoyu_xy-c5.config` 并 `make defconfig`
6. **配置自检**：关键包必须存在；`uboot-envtools` / `opkg` 出现即失败
7. `make download -j$(nproc)`
8. `make -j$(nproc)`（失败降级 `-j1`、`-j1 V=s`）
9. **产物自检**：`.manifest` 关键包齐全、禁止包缺席；解包 rootfs 确认无 `fw_loadenv` /
   `30_uboot-envtools` / `05_fw_defaults` 早期启动钩子
10. `gh release create` 发布全部产物（含 `config.buildinfo`、`profiles.json`、`sha256sums`）
11. 仅保留最近 3 个 Release

## 刷机

产物为 `openwrt-ramips-mt7621-xiaoyu_xy-c5-squashfs-sysupgrade.bin`，用 Breed 的「固件更新」直接刷入即可。

- 跨固件切换时建议不保留配置。
- `*initramfs-kernel.bin` 为内存运行镜像，可用于临时验证，不写入 Flash。

## 如何增删功能包

编辑 `config/xiaoyu_xy-c5.config`，加一行：

```
CONFIG_PACKAGE_包名=y
```

不必手工补全依赖，`make defconfig` 会处理。但**不要**加入 `uboot-envtools` 与 `opkg`
（工作流第 6 步会直接失败拦截）。
