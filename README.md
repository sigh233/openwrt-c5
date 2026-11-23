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
   新版上游已全面切换到 `apk` 包管理器，强制装 `opkg` 会与 `apk-mbedtls` 并存，
   并额外引入 `/etc/opkg.conf` 与 `/etc/uci-defaults/20_migrate-feeds` 迁移钩子。

3. **`ipv6helper`（已删除）**
   该包在新版上游已被移除，写了也只会被 `make defconfig` 丢弃。

4. **构建源固定到官方稳定 tag（不是 `main`）**
   官方 **v25.12.6** 稳定版镜像在 Breed 下可正常启动。工作流固定检出该 tag
   （commit `64900e756efa667774214f9c6b17c113222e01f7`），按 `refs/tags/v25.12.6`
   拉取，并强校验 `HEAD` 与预期 SHA 一致；**拉取失败直接中止，不再回退 `main`**。
   为什么必须放弃 `main`，见下方「踩坑记录」。

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
3. 克隆上游 `openwrt/openwrt`：`git fetch --depth 1 origin refs/tags/v25.12.6`，
   校验 `HEAD` 等于预期 SHA，不符即失败退出
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

## 踩坑记录：`main` 分支固件刷入后不启动

### 现象

用 Breed 刷入本仓库编译的 `sysupgrade.bin` 后：

- **网口灯亮**，**sys 灯不亮**，拿不到 IP，无法判断是否引导
- 断电重启、长按复位键，**Breed 仍能进入**
- 硬件未做过任何改动
- 同一台机器刷**官方 v25.12.6 稳定版** `sysupgrade.bin` 可以正常启动

### 为什么 sys 灯不亮是关键线索

`sys` 灯由内核加载后的 LED 驱动点亮。它不亮 = 内核没有跑起来（连 preinit 都没到），
不是某个软件包在启动早期报错。若真是 `uboot-envtools` 那类问题，内核是能起来的，
表现应为「灯有反应、偶尔能 ping 通、或反复重启」，而不是完全静默。

### 排查过程

| 步骤 | 检查内容 | 结果 |
| --- | --- | --- |
| 1 | 产物是否正常生成 | Run #2 全绿，`sysupgrade.bin` 7.1 MB，132 个包齐全 |
| 2 | 是否被 `uboot-envtools` 拖累 | 已删除该包，产物自检确认无早期启动钩子——排除 |
| 3 | `.config` 是否改动了内核参数 | `config.buildinfo` 共 39 行**全是 `CONFIG_PACKAGE_*`**，无任何 `CONFIG_KERNEL_*` / `CONFIG_TARGET_KERNEL_PARTSIZE` / `CONFIG_TARGET_ROOTFS_*`，即内核压缩方式、加载地址、分区大小全部沿用上游默认——排除 |
| 4 | 内核版本对照 | 自编固件 `main@dd72c4c` = **6.18.60**；官方稳定版 `v25.12.6` = **6.12.112** |

### 根因

上游 `main` 分支在此期间已从 6.12 跨到 **6.18**，处于新内核的开发期。
这一代切换中 `ramips/mt7621` 的 `sysupgrade` 引导方式发生了变动，**Breed 不认**，
因此内核根本无法被加载。

配置本身没有问题：`config.buildinfo` 里一个内核参数都没改，所有差异都来自上游分支的选择。

### 修复

1. `UPSTREAM_TAG` 新增为 `v25.12.6`，`UPSTREAM_SHA` 指向其 commit `64900e7`
2. 克隆方式由 `git fetch origin main` 改为 `git fetch origin refs/tags/v25.12.6`
3. 增加 `HEAD` 与预期 SHA 的强校验，不符即报错退出
4. **删除「失败回退 `main`」逻辑** —— 这是最关键的一条。旧逻辑在固定 commit
   拉取失败时会静默回退到 `main` 最新，等于又编出一个 6.18 的固件，白忙一场。
   现在宁可构建失败，也不产出引导不了的镜像。

### 如何验证这次修复是否生效

构建日志里应能看到：

```
已切换到 v25.12.6 (64900e756efa667774214f9c6b17c113222e01f7)
```

产物自检的包版本中，内核应为 **`6.12.x-r1`**（不再是 `6.18.x-r1`）。
刷入后 `sys` 灯应正常点亮。

### 后续升级原则

不要直接跟随 `main`。需要升级时，先确认真实设备上**官方稳定版镜像能正常启动**，
再用该稳定版对应的 tag 作为 `UPSTREAM_SHA`，并在日志中核对内核版本与产物。

## 如何增删功能包

编辑 `config/xiaoyu_xy-c5.config`，加一行：

```
CONFIG_PACKAGE_包名=y
```

不必手工补全依赖，`make defconfig` 会处理。但**不要**加入 `uboot-envtools` 与 `opkg`
（工作流第 6 步会直接失败拦截）。
