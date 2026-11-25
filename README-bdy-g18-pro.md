# BDY G18-PRO (RK3568) on ImmortalWrt 25.12.2

本分支基于官方 tag `v25.12.2`，将 **BDY G18 AX3000** 路由器（原仓库 [zhoufuli/immortalwrt-rk356x](https://github.com/zhoufuli/immortalwrt-rk356x)，24.10 系）移植到 ImmortalWrt 25.12.2，全部改动已在真机验证通过。

## 设备规格

| 项 | 值 |
|---|---|
| SoC | Rockchip RK3568（8GB DDR4，eMMC 启动） |
| 交换机 | Realtek RTL8367S：4×GE LAN + 1×GE WAN |
| 交换机 CPU 口 | **双口**：port6 = SGMII → gmac1（`fe010000`，eth0），port7 = RGMII → gmac0（`fe2a0000`，eth1） |
| WiFi | MT7615 已物理移除，镜像不含 mt7615 包 |
| 默认网络 | LAN = `br-lan`（eth0，192.168.1.1，DHCP），WAN = eth1；VLAN1 = 用户口0-3+CPU6，VLAN2 = WAN4+CPU7 |

## 编译

```bash
git clone -b bdy-g18-pro-25.12.2 https://github.com/rfuger/immortalwrt
cd immortalwrt
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig   # Target System → Rockchip → ARMv8，选 BDY G18-PRO（可同时选 Firefly ROC-RK3568-PC）
make -j$(nproc)
```

产物在 `bin/targets/rockchip/armv8/`，镜像名含 `bdy_g18-pro`。

## 刷写

- **从现有 ImmortalWrt/OpenWrt 升级（推荐）**：LuCI「系统 → 备份/升级」刷入 `*-sysupgrade.img.gz`，或命令行 `sysupgrade -n <image>`。新旧镜像分区表一致时只逐分区写入，**不会覆盖 uboot 区**。
- **整盘刷**：Maskrom（按住 recovery 上电）或 Loader 模式下 `rkdeveloptool wl 0 <image>`。本分支的 uboot 构建已真机验证可整盘启动。

刷完后 LAN 口即插即用：开机 preinit 会自动配置交换机 VLAN。

## 分支改动（4 个提交，基于 v25.12.2）

1. `uboot-rockchip: add bdy-g18-pro-rk3568 board` — 板级 dts/u-boot.dtsi/defconfig 经 `patches/901` 注入 + Makefile 注册
2. `rockchip: rtl8367b: add RTL8367S dual-CPU port support` — 官方驱动合入 vendor 扩展：`realtek,extif0/1/2` 九单元格、SGMII(mode 11)/HSGMII(mode 12)、RTL8367S LED 初始化、of_match `realtek,rtl8367s`
3. `rockchip: add BDY G18-PRO device tree and kernel patches` — 自包含 DTS（`target/linux/rockchip/files/`）+ `713` dtb 注册 + `714` xpcs fixed-link 跳过 Clause-37 自协商（修复官方 710 补丁在 fixed-link 下 SGMII 链路异常）
4. `rockchip: armv8: add bdy_g18-pro image and network setup` — 设备定义、02_network 交换机 VLAN、IRQ 亲和（eth0→core3、eth1→core2）

## 排障教训（同类板子通用）

- **交换机用户口不通（link up 但 ping 不通）**：25.12 的 netifd 已移除 asf(swconfig) handler，`config switch` 段不再被 netifd 处理；官方机制改为 preinit 脚本仅在 `/sbin/swconfig` 存在时配置 VLAN。**凡用 `ucidef_add_switch` 的板子，`DEVICE_PACKAGES` 必须包含用户态 `swconfig` 包。**
- **uboot 进 Maskrom**：本板 eMMC 在 uboot dts 里 include `*-u-boot.dtsi`（HS400 时序 + datastrobe pinctrl）会导致 SPL eMMC 初始化失败；901 补丁保留该 dtsi 文件但**不 include**，与原厂行为一致。
- **SGMII fixed-link**：官方 710 补丁无条件开启 xpcs Clause-37 自协商，fixed-link 场景需按 714 跳过。

## 跟随官方升级

```bash
git fetch origin
git rebase --onto v<新版本> v25.12.2 bdy-g18-pro-25.12.2
```

冲突点通常只有三处：713（内核 Makefile 行号漂移）、714（官方 710 变化）、rtl8367b.c（上游驱动更新）。DTS、uboot 板级、mk/base-files 均为独立文件，自动带过。
