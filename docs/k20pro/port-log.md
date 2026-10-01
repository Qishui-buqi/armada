# Redmi K20 Pro (`raphael`, SM8150) 移植日志

本目录记录把 Armada 移植到 Redmi K20 Pro / Mi 9T Pro（Qualcomm Snapdragon 855，`SM8150`，aarch64）的工作。

> 本文件路径对应仓库内 `docs/k20pro/port-log.md`，用于开启 fork 内部的 PR 以触发 CI 构建。

---

## 状态

| 阶段 | 状态 |
|---|---|
| P0 情报与启动链实测 | ✅ 完成（见下"启动链结论"） |
| P1 基线复现（CI 构建官方镜像） | ⏳ 进行中（本 PR 即为第一次触发） |
| P2 内核与 DTB（raphael） | ⬜ 未开始 |
| P3 引导镜像与 rootfs 落地 | ⬜ 未开始 |
| P4 真机点亮（L1） | ⬜ 未开始 |
| P5 设备层与可用性（L2） | ⬜ 未开始 |

---

## 启动链结论（已实测）

Armada 的引导链本质是「**自己生成一个 Android boot.img，放到引导器读得到的位置，再由 ostree 切根**」：

1. 内核 = kernel.org 原生 tarball（`7.2.6`）+ `patches/series` + 自带板级 DTS（`packages/kernel/dts/`）
2. `bootc-image-builder` 产出 3 分区磁盘：p1 vfat ESP（label `ARMADA`）/ p2 `/boot` / p3 `/`（btrfs，最小 10 GiB）
3. `post_process/make-bootimg.sh`：`kernel = gzip(Image) + 追加全部 supported-dtbs`，`mkbootimg` → 写到 **p1 的 `/KERNEL`**
4. `post_process/finalize-armada-image.sh`：把 `EFI/` 改名 `EFI.disabled`，让 ABL 落到 `/KERNEL`
5. 运行时：ABL → `/KERNEL` → 内核 → dracut initramfs → `ostree-prepare-root` 按 `ostree=` / `root=UUID` 切根
6. 更新：`armada-bootimg-update` 重新生成 `/KERNEL`（保留 `KERNEL.BAK` 回滚位）

关键：ostree 配置为 **`sysroot.bootloader=none`** —— 启动阶段**不依赖 ESP / systemd-boot**。

---

## K20 Pro 需要改动的清单

| # | 文件 | 改动 |
|---|---|---|
| 1 | `packages/kernel/dts/sm8150-xiaomi-raphael.dts` | 新增（来自上游 SM8150 社区树，962 行，含三星 `ams639rq08` 面板、Goodix `gt9886` 触摸、`a640_zap` GPU、IPA、adsp/cdsp/mpss/slpi、UFS、USB-C、wcn3990 WiFi/BT） |
| 2 | `packages/kernel/config/armada-kernel.config.overrides` | 加 `CONFIG_PINCTRL_SM8150=y`；确认 `CONFIG_ATH10K` / `ATH10K_SNOC`（wcn3990） |
| 3 | `system_files/usr/lib/armada/supported-dtbs` | 加 `sm8150-xiaomi-raphael` |
| 4 | `system_files/usr/lib/armada/bootimg-args` | 按设备取几何参数：K20 Pro 需 `pagesize 4096`、`base 0`、`ramdisk_offset 0` |
| 5 | `system_files/usr/lib/armada/devices/xiaomi-raphael.conf` | 新增设备档案（`ARMADA_SOC_CLASS=SM8150` 等） |
| 6 | `system_files/usr/libexec/armada/device-env` | 加 DT model 匹配 `"Xiaomi Redmi K20 Pro"`；加 `SM8150` CPU 拓扑分支 |
| 7 | `system_files/usr/libexec/armada/armada-bootimg-update` | 落盘目标由 `ESP/KERNEL` 改为手机 `boot` 分区，保 OTA 可用 |
| 8 | fstab / 安装逻辑 | 去掉 `/boot/efi` 依赖（手机无 ESP） |
| 9 | `post_process/make-bootimg.sh` | 新增 phone 目标：产出可 `fastboot flash boot` 的 boot.img |

### 已知约束

- K20 Pro **无 microSD 卡槽**、无手柄、小米不允许替换 ABL → **只能走 fastboot 刷 `boot.img` + rootfs 落内部存储**。
- 引导几何必须改用 raphael 的参数（pmOS `deviceinfo` 实测值），不能沿用 Armada 给 ROCKNIX ABL 的 `ARMADA_BOOTIMG_ARGS`。
- 交付目标为 **L1（点亮）+ L2（桌面可用）**；**不承诺 Game Mode / Steam**（手机无手柄、无风扇）。
- 任何真机操作前必须备份 `boot` / `dtbo` / `vbmeta` / `abl_a` / `abl_b` 与分区表，并确认 EDL 救援路径。

---

## 构建方式

本 fork 使用项目自带的 **PR 工作流**（`.github/workflows/pr.yml`）构建，它运行在 GitHub 原生 ARM64 runner（`ubuntu-24.04-arm`）上，且**不需要任何密钥**：

```
push 分支 → 开 fork 内部 PR → CI 自动组装镜像并产出 armada-*.img.gz 产物
```

> 不要使用 `build.yml`（强制要求 `SIGNING_SECRET`）与 `build-disk.yml`（会去拉 fork 命名空间下不存在的 ghcr 镜像）。

---

## P2 变更集（本分支 `k20pro-p2-dts`）

### 加入的文件

| 文件 | 内容 |
|---|---|
| `packages/kernel/dts/sm8150-xiaomi-raphael.dts` | 板级设备树，**与上游逐字节一致**（sha256 `9cfcfe92…`，28546 B / 1086 行），来自 `soc/qualcomm-sm8150/linux` 分支 `sm8150/6.17` |
| `packages/kernel/dts/sm8150-xiaomi-raphael.dts.patch` | 删掉 `&ipa` 覆盖块（该 label 只存在于特性分支 `andrew/6.16-ipa`，主分支无法解析） |
| `packages/kernel/dts/sm8150-pmic-labels.patch` | 给主线的 `pm8150.dtsi` / `pm8150b.dtsi` 补上 raphael 引用的 `pm8150_rtc` / `pm8150_sdam_2` / `pm8150b_charger` / `pm8150b_fg` 及其 ADC 通道（纯增量） |
| `system_files/usr/lib/armada/devices/xiaomi-raphael.conf` | 设备档案（`ARMADA_SOC_CLASS=SM8150`，竖屏 `normal`） |
| `system_files/usr/lib/armada/supported-dtbs` | +1 行 `sm8150-xiaomi-raphael`（与 `dts/` **成对**修改） |
| `system_files/usr/libexec/armada/device-env` | 新增 DT model 匹配 `"Xiaomi Redmi K20 Pro"` 与 `SM8150` CPU 拓扑分支 |
| `packages/kernel/config/armada-kernel.config.overrides` | 见下 |

### 配置增量与理由（均在 7.2.6 源码里核对过）

```
CONFIG_SM_GCC_8150=y          # ← 关键：arm64 defconfig 里没有 8150/8250 的 GCC，
                              #   不补上就无法 bring up UART 与各消费方时钟
CONFIG_SM_VIDEOCC_8150=y
CONFIG_SM_CAMCC_8150=m
CONFIG_PINCTRL_SM8150=y       # 符号在 drivers/pinctrl/qcom/Kconfig.msm
CONFIG_INTERCONNECT_QCOM_SM8150=y
CONFIG_DRM_PANEL_SAMSUNG_AMS639RQ08=y   # 面板驱动已上游，但不在 defconfig
```

- **无需**为 WiFi 加配置：`CONFIG_ATH10K=m` 与 `ATH10K_SNOC=m` 已在 arm64 defconfig 中。
- **无需**为显示时钟加符号：SM8150 的 dispcc 由 `dispcc-sm8250.c` 提供（`qcom,sm8150-dispcc`），即已有的 `CONFIG_SM_DISPCC_8250`。
- `SM_GPUCC_8150` 已在 defconfig 中 `=y`。

### 已知缺口（不阻塞编译，但影响功能）

1. **触摸**：`goodix,gt9886` 在主线 **与社区树** 的 `goodix.c` 里都**没有**匹配项。→ 需要给驱动加 compatible 的小补丁，才能有触摸。
2. **IPA / 移动数据**：本变更集删掉了 `&ipa` 段（见上）。`CONFIG_QCOM_IPA=m` 在 defconfig 里是有的，缺的是 SoC 侧 DT 节点（只在特性分支上）。
3. **boot.img 几何**：Armada 的 `bootimg-args` 是给 ROCKNIX ABL 调优的（`pagesize 2048`、`ramdisk_offset 0x06000000`），而 raphael 需要 `pagesize 4096`、`base 0`、`ramdisk_offset 0` → 属 P3，本分支未改。
4. **不 vendor 社区 dtsi**：社区树的 `sm8150.dtsi` 相对主线 7.2.6 **少了 40 行**（ICE 加密、UFS ICE 时钟、DP 像素时钟等），搬过来会造成回退，故只做纯增量补丁。

### 本地验证（文本级，已通过）

把两个补丁应用到**主线 v7.2.6 的 dtsi 集合**（10 个 `sm8150*`/`pm8150*` 文件，LF 归一）后重做 label 解析：

| 指标 | 打补丁前 | 打补丁后 |
|---|---|---|
| dtsi 侧定义的 label | 553 | **557**（+4，正是我们补的 PMIC labels） |
| DTS 引用 | 86 | 85（删掉 `&ipa` 段少 1 个） |
| **无法解析的引用** | **5** | **0** ✅ |

- `pm8150_rtc` / `pm8150_sdam_2` / `pm8150b_charger` / `pm8150b_fg` 各出现 **1 处定义**；
- `&ipa {` 已确认消失；
- **85 个引用全部可解析** —— 文本级等价于"能编译"。真正的 kernel build 仍以 CI 为准。

> 两个补丁本身也已用 `git apply --check` 在临时仓库里验证可**精确应用**（`-F0` 语义，exit 0）。
