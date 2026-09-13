# Qualcommax NSS Builder（个人定制版 · Redmi AX6）

### Redmi AX6 专用 OpenWrt 固件云编译 — NSS 硬件加速，基于上游 EDMA 驱动

[![Build](https://img.shields.io/github/actions/workflow/status/a544883434-ui/Qualcommax_NSS_Builder/build.yml?branch=main&style=flat-square&logo=github&label=Build)](https://github.com/a544883434-ui/Qualcommax_NSS_Builder/actions/workflows/build.yml)
[![License](https://img.shields.io/github/license/a544883434-ui/Qualcommax_NSS_Builder?style=flat-square&label=License)](LICENSE)
[![Last Commit](https://img.shields.io/github/last-commit/a544883434-ui/Qualcommax_NSS_Builder?style=flat-square&label=Last%20Commit)](https://github.com/a544883434-ui/Qualcommax_NSS_Builder/commits/main)

> **这是 [JuliusBairaktaris/Qualcommax_NSS_Builder](https://github.com/JuliusBairaktaris/Qualcommax_NSS_Builder) 的个人 fork。**
> 只为一台机器服务：**红米 AX6 硬改版（IPQ8072A + 1GB RAM + 240MB NAND，stock 大分区布局）**。
> 已裁剪到单设备构建，与上游的 39 设备全家桶无关。

## 这个 fork 改了什么

相比上游，本 fork 的差异：

| 项目 | 说明 |
|---|---|
| **单设备矩阵** | 只构建 `ipq807x-1g` 组里的 `redmi_ax6-stock`（换芯 SoC + 1GB 内存 + stock 大分区定制变体） |
| **新增设备定义** | `redmi_ax6-stock`：custom U-Boot 布局（`KERNEL_SIZE` + `ARTIFACTS`，产 **factory.ubi**，可走 uboot 网页直刷），配套 `ipq8071-ax6-stock.dts`（SMEM 分区）与 board 级脚本接入（caldata / 02_network / 01_leds / platform.sh / bootcount / ubootenv 共 6 处） |
| **内存组选择** | 1g 组：`NSS_MEM_PROFILE_HIGH` + `ATH11K_MEM_PROFILE_1G`，按实机内存打满 profile |
| **产物** | `sysupgrade.bin` + `factory.ubi` 两种格式，release 附 buildinfo 与 sha256sums |

上游原版面向 39 台设备 + AX3600 专属优化 + mesh/PPE 测试线，这里全部裁掉——
单设备不需要那些复杂度。技术栈本身未动：NSS 硬件卸载跑在 OpenWrt 主线的
`qca_edma` / `qca_ppe` 以太网驱动上（来自
[PR #22381](https://github.com/openwrt/openwrt/pull/22381)），而非各家 NSS 构建常用的
`qca-nss-dp` / `qca-ssdk` 方案。源码来自
[openwrt-nss-edma](https://github.com/a544883434-ui/openwrt-nss-edma)（同款 fork，含上述 AX6 补丁）。

## 下载

从 [Releases](https://github.com/a544883434-ui/Qualcommax_NSS_Builder/releases) 取最新版：

- `...redmi_ax6-stock-squashfs-sysupgrade.bin` — 已在跑 OpenWrt 时系统内升级
- `...redmi_ax6-stock-squashfs-factory.ubi` — custom U-Boot 网页（192.168.1.1）直刷

> [!WARNING]
> 这是**硬改机定制固件**：SoC 换芯（8071A→8072A）+ 大分区布局，只适配同规格改装机。
> 原装 Redmi AX6 请用上游官方版或社区固件，**不要刷本仓库产物**。

## 运行时模型（与上游一致）

`nss` 服务在启动早期一次性完成数据面决策：装载 NSS 数据面、引导固件、让 Wi-Fi
直接以卸载模式（wifili）上线。`nss-up` 随网络就绪叠加 ECM（及配置后的 SQM）。
日志看 `logread -e nss`；健康检查 `nss-status` 或 LuCI **Status → NSS Offload**。

万能恢复开关（sysupgrade 后仍生效）：

```sh
uci set nss.general.enabled='0'; uci commit nss
```

## 默认集成

| 模块 | 说明 |
|---|---|
| NSS 数据面 | `kmod-qca-nss-drv` + `kmod-qca-ppe-nss` glue |
| 连接卸载 | ECM（IPv4 NAT / IPv6 路由 / PPPoE-over-VLAN） |
| 桥接卸载 | 有线 LAN 硬件桥接 |
| 组播 | `kmod-qca-mcs` |
| SQM | NSS qdisc + `sqm-scripts-nss`（默认停用模板，按实测线路带宽的 90-95% 配置后启用） |
| Wi-Fi | ath11k NSS offload（wifili），双射频 |
| 安全 | OpenSSH（抗量子 KEX/AEAD/ETM）、ASLR/PIE/FORTIFY/RELRO/seccomp、WAN DROP + BCP38 |
| 固件 | `NSS.FW.12.5-210-HK.R`，HIGH 内存 profile |

Wi-Fi 出厂禁用（镜像不可能内置密码）：接网线，LuCI → Network → Wireless 配 SSID/密钥并启用射频。

## 自行构建

仓库即工作流：fork 后改 [`devices/`](devices/) 配置，push 触发 GitHub Actions。
`env:` 参数集中在 [`.github/workflows/build.yml`](.github/workflows/build.yml)。

本地构建：

```sh
git clone --branch nss-edma-rework https://github.com/a544883434-ui/openwrt-nss-edma openwrt
cd openwrt
cp feeds.conf.default feeds.conf
echo "src-git nss https://github.com/JuliusBairaktaris/nss-packages.git;edma-nss" >> feeds.conf
./scripts/feeds update -a && ./scripts/feeds install -a
B=../Qualcommax_NSS_Builder/devices
cat "$B/common/config" "$B/ipq807x-1g/config" > .config
make defconfig && make -j"$(nproc)"
```

> [!IMPORTANT]
> `nss-edma-rework` 分支会周期性 rebase（历史重写）。更新检查outs时用
> `git fetch origin && git reset --hard origin/nss-edma-rework`，
> 并重建 feeds（`rm -rf feeds/nss package/feeds/nss` 后重新 update/install），
> 普通 `git pull` 会失败或产生坏合并。

## 致谢

- **[Ansuel (Christian Marangi)](https://github.com/Ansuel)** — 本栈所依赖的 [EDMA rework](https://github.com/openwrt/openwrt/pull/22381)
- **[JuliusBairaktaris](https://github.com/JuliusBairaktaris)** — 上游 builder 与 nss-edma 树的原作者，本 fork 的一切基础
- **[qosmio](https://github.com/qosmio)** — NSS 开发与 Wi-Fi offload 补丁谱系
- **OpenWrt 社区** — [IPQ807x NSS Build 帖](https://forum.openwrt.org/t/qualcommax-nss-build/148529)

## License

[GPL-2.0](LICENSE)，与 OpenWrt 一致。
