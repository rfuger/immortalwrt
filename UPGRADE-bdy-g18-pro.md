# 跟随上游版本升级（bdy-g18-pro 分支）

上游 ImmortalWrt 发布新版本（如 `v25.12.3`，或大版本 `v26.x`）后，把本移植分支挪到新基座并编译新固件的完整步骤。

## 前提

- 本地构建树的 remote 布局：`origin` = 官方 `immortalwrt/immortalwrt`，`rfuger` = 本 fork（`git remote -v` 可核对）
- 移植分支 `bdy-g18-pro` 跟踪 `rfuger/bdy-g18-pro`
- 记下当前基座 tag（本分支目前是 `v25.12.2`）

## 标准流程（维护版升级，如 25.12.2 → 25.12.3）

```bash
# 1. 拉取官方新 tag
git fetch origin --tags
git tag | grep v25.12.3          # 确认拿到了

# 2. 把移植提交整体挪到新基座（<旧基座> 换成上次用的 tag）
git checkout bdy-g18-pro
git rebase --onto v25.12.3 v25.12.2 bdy-g18-pro

# 3. 更新 feeds 与配置
./scripts/feeds update -a && ./scripts/feeds install -a
make defconfig                   # 新版新增选项自动补默认值，旧 .config 保留

# 4. 编译
make -j$(nproc)

# 5. 验证产物（见下方检查单）

# 6. 推分支 + 挂 Release
git push -f rfuger bdy-g18-pro
gh release create bdy-g18-pro-25.12.3-r1 --target bdy-g18-pro \
  --title "BDY G18-PRO ImmortalWrt 25.12.3 (r1)" \
  bin/targets/rockchip/armv8/immortalwrt-rockchip-armv8-bdy_g18-pro-squashfs-sysupgrade.img.gz
```

维护版通常**零冲突**（不动 rockchip/generic 相关文件），rebase 几秒完成。

## rebase 干净 ≠ 编译一定通过：三个真正的脆弱点

| 改动 | 冲突发生在 | 处理方式 |
|---|---|---|
| `rtl8367b.c`（+520 行，直接改 `target/linux/generic/files/` 里的文件） | **rebase 时**（上游若更新该文件） | 手工解冲突：我们的改动集中在 `realtek,extif0/1/2` 九单元格解析、SGMII/HSGMII 模式、`rtl8367S_led_*` 函数、of_match `realtek,rtl8367s` 四处 |
| `713` dtb 注册补丁 | **编译时**（内核补丁集版本变了 → 对 `arch/arm64/boot/dts/rockchip/Makefile` 的行号漂移） | 编译日志 `grep 'Hunk FAILED'` 发现后：删旧补丁，在新内核源码里把 `rk3568-bdy-g18-pro.dtb` 行加到 `rk3568-9tripod-x3568-v4.dtb` 之后，用 `git diff` 重新生成补丁 |
| `714` xpcs fixed-link 补丁 | **编译时**（官方 `710` 补丁内容变了） | 打开新 710，把"fixed-link 时跳过 Clause-37 AN"的改动重新叠加，重新生成补丁 |

## 特殊情形：内核大版本变化（如 6.12 → 6.18，通常发生在大版本）

- `target/linux/rockchip/patches-6.12/` 目录名随内核版本变（如 `patches-6.18/`），**713/714 要移过去并按新内核重做**；
- 先检查官方 `710`（dwmac-rk SGMII）与 `711`（rk3568 xpcs）是否已被上游内核吸收——吸收了就直接删掉这两个补丁，DTS 里相关节点可能也要跟着调；
- DTS 本身跨内核版本高度稳定，一般原样可用；
- uboot 版本也可能升级（如 2025.10 → 2026.x）：`package/boot/uboot-rockchip/patches/901` 若 apply 失败，按新 uboot 的 dts/defconfig 重做板级补丁（defconfig 偶尔要补新引入的符号）。

## 刷写

- **推荐**：LuCI「系统 → 备份/升级」或 `sysupgrade -n <image>`——新旧镜像分区表一致时逐分区写入，uboot 区不动；
- 整盘刷：Maskrom/Loader 下 `rkdeveloptool wl 0 <image>`（本分支 uboot 已真机验证可整盘启动）。

## 升级后验证检查单

- [ ] `make` 日志无 `Hunk FAILED` / `does not apply`
- [ ] `bin/targets/rockchip/armv8/` 出现新的 `bdy_g18-pro` 镜像
- [ ] `immortalwrt-rockchip-armv8.manifest` 含 `swconfig` 与 `kmod-switch-rtl8367b`
- [ ] 真机刷入后 `swconfig dev switch0 show` 有 `enable_vlan: 1` 且 port 0/6/7 link up
- [ ] LAN 口接 PC 能拿到 DHCP（默认 `192.168.1.1`）
