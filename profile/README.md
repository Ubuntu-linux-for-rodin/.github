## Hi there 👋
# Ubuntu for Rodin

把 **Ubuntu Desktop** 完整带到 **Redmi Turbo 4（国行）/ POCO X7 Pro**（代号 **rodin**）上：上游 Linux 内核 + 原生 systemd/GNOME 用户态，不依赖 Android 框架。

- SoC：MediaTek MT6899（天玑 8400-Ultra），Mali-G720 MC7（r49p1），UFS
- 目标形态：Ubuntu Desktop 26.04 LTS（GNOME），首启动即桌面 rootfs
- 内核路线：upstream v7.3 直线前向移植 vendor 驱动栈，不走中间基线

## 仓库

| 仓库 | 内容 |
|---|---|
| [`linux`](https://github.com/Ubuntu-linux-for-rodin/linux) | v7.3 上游基线 + rodin 前向移植（DTS、vendor 驱动、配置），在 `linux-7.3.y` 上直接推进 |
| [`initramfs`](https://github.com/Ubuntu-linux-for-rodin/initramfs) | stage-1：busybox 静态、按 partlabel 找根、rescue shell |
| `build` | mkbootimg 打包 / fastboot 刷写脚本 / CI |
| `rootfs` | Ubuntu 26.04 Desktop rootfs 构建器 + 固件提取脚本 |
| `docs` | 刷机指南 / 功能状态矩阵 |

固件 blob（connsys、mali_csffw 等）只进构建产物、不进 git——`rootfs` 仓提供从官方 ROM 或真机提取的脚本。

## 设计决策（2026-10-30）

| 决策点 | 结论 |
|---|---|
| 内核基线 | 直接在 upstream v7.3 前向移植，不做 6.18 中间启动产物 |
| 提交方式 | `linux-7.3.y` 直接堆 commit，不引入 patch series 管理层 |
| rootfs 落位 | userdata 整盘独占（ext4）；super 完全不动，保留回刷 HyperOS/LineageOS 的能力 |
| 机型适配 | 256GB / 512GB 双机型：rootfs 镜像定容烧录，首启动扩容填满 userdata |
| dtbo / 固件分区 | **只读不可刷写**（有启动校验）：base DT 适配原厂 dtbo overlay；固件从分区只读加载或用 ROM 内文件 |
| 目标形态 | Ubuntu Desktop 26.04 LTS（GNOME） |
| 调试通道 | 无 UART：ramoops/pstore + USB gadget（NCM 网络 + ACM 串口），M3 起屏幕成为第三通道 |
| 槽位分工 | _a = Ubuntu 试验槽；_b = 6.18 oracle 槽，`fastboot set_active` 切换对照 |
| 解锁 | 设备已解锁为前置条件，不在本项目范围内 |

## 设备底账

依据官方 ROM（HyperOS 3.0.304.0.WOJCNXM）的 MT6899_Android_scatter。A/B 双槽，无独立 recovery。

### 与 Ubuntu 方案相关的分区

| 分区 | 每槽大小 | Ubuntu 方案 |
|---|---|---|
| boot_a/b | 64 MiB | 7.3 内核 Image.lz4 + 空 GKI ramdisk（boot header v4）。6.18 基线 Image.lz4 ≈ 44 MB，上限 60 MB |
| vendor_boot_a/b | 64 MiB | dtb + **Ubuntu initramfs**（vendor_ramdisk 形式打包） |
| dtbo_a/b | 8 MiB | **原样保留，不可刷写**（有启动校验）。LK 仍按 panel id 选取原厂 overlay 合并后传内核——base DTB 必须兼容原厂 overlay 的覆盖目标（节点路径/label 对齐 vendor 基线） |
| init_boot_a/b | 8 MiB | Android 通用 ramdisk，Ubuntu 不使用，保留原样 |
| vbmeta ×3 | 1 MiB | 刷 `--disable-verity --disable-verification` 版本 |
| super | 11.25 GiB | **不动**（动态分区现状：product 4.9G / vendor 2.0G / system_ext 936M / odm 862M / system 880M / mi_ext 64M / 两个 dlkm） |
| userdata | 随机型 | **Ubuntu rootfs**（ext4，首启动扩容填满） |
| pstore | 133 MiB | ramoops 持久内核日志——主要调试通道之一 |
| modem | 200 MiB ×2 | 保留（本路线图内不驱动基带） |

固件分区（connsys_wifi/bt/gnss、audio_dsp、scp、sspm、mcupm、gpueb、vcp、apusys、mvpu_algo、ccu、dpm、pi_img、tee、gz、spmfw、mcf_ota）**只读不可刷写**——各加载方有启动/运行校验，任何情况下不写这些分区。它们同时是 WiFi/BT/GPU/DSP 各阶段的固件来源：由启动链加载的（modem/tee 等）保持原样即可；运行时加载的（connsys、audio_dsp、mali 等）走两条路——vendor 驱动按分区名只读加载（保持原行为），或改从 /lib/firmware 加载（文件从官方 ROM 提取）。

### 启动链（Ubuntu 方案）

```
BootROM → preloader → LK（已解锁）
  → boot_a(Image.lz4) + vendor_boot_a(dtb + initramfs) + 原厂 dtbo（不可替换，LK 合并）
  → Linux 7.3 → initramfs 挂 /dev/disk/by-partlabel/userdata → switch_root
  → systemd → GDM/GNOME
```

### 上游事实（2026-10-30 核查）

- upstream v7.3 的 `arch/arm64/boot/dts/mediatek/` **没有 mt6899**（mt68xx 系只有 mt6858/mt6878/mt6893 pinfunc）→ pinctrl、clk、PMIC、DTS 全部走前移。
- v7.3 已有 panthor 驱动（Mali CSF 世代）；对 G720 (r49p1) 的实际支持状态需在 M5 专项验证。

## 资产与角色

| 仓库 | 角色 |
|---|---|
| [`omajili-manbu/Xiaomi_Rodin_Kernel_Enhance@bsp-rodin-c-rebase`](https://github.com/omajili-manbu/Xiaomi_Rodin_Kernel_Enhance) | 6.18 已点亮内核树（android17-6.18-lts 基线）——**oracle + DT/vendor 源**，驻真机 _b 槽 |
| [`omajili-manbu/MTK_kernel_device_modules@bsp-rodin-v-oss`](https://github.com/omajili-manbu/MTK_kernel_device_modules) | 设备模块前移源（569 模块名：538 内建 / 31 丢载 / 0 blob） |
| [`omajili-manbu/MTK_kernel_modules@bsp-rodin-v-oss`](https://github.com/omajili-manbu/MTK_kernel_modules) | 技术 kmod 前移源（mali r49p1、conninfra、udc、mtkcam 等） |
| `Ubuntu-linux-for-rodin/linux` | **目标内核基线**（v7.3 裸导入 + rodin 工作） |
| `Ubuntu-linux-for-rodin/initramfs` | stage-1（建设中） |
| LineageOS 23.2 / OrangeFox(rodin) | 恢复 / 对照环境 |

三仓 rebase 点配对快照（2026-10-16）：kernel `52a8e4b4498a` · devmod `bc2dfc01+122` · kmod `991e9cd48ac7`。从 6.18 树前移任何代码时以此为准。

## 路线图

| 里程碑 | 内容 | 状态 |
|---|---|---|
| M0 | 工作台就绪 + oracle 槽位整理 | ⬜ |
| M1 | 7.3 内核点亮 | ⬜ |
| M2 | Ubuntu Desktop rootfs 启动（SSH） | ⬜ |
| M3 | 显示 | ⬜ |
| M4 | 触摸 → **桌面形态达成** | ⬜ |
| M5 | GPU 加速 | ⬜ |
| M6 | 无线（WiFi/BT） | ⬜ |
| M7 | 电源 | ⬜ |
| M8 | 音频 | ⬜ |
| M9 | suspend 与打磨 | ⬜ |

### M0 工作台就绪
- org 仓初始化：linux 建 `linux-7.3.y` rodin 工作、initramfs 骨架、新建 build / rootfs / docs
- 交叉工具链：LLVM=1 全量构建，确认 7.3 要求的 clang 版本（6.18 用的 r596125 是否满足，不满足则升级）
- 槽位整理：_b 灌入 6.18 oracle 栈并验证 `fastboot set_active` 双向切换可靠
- 刷机链路冒烟：用已验证的 6.18 Image 走一遍 fastboot 刷 boot / vendor_boot / vbmeta，确认刷写与恢复流程畅通
- **验收**：刷写/切槽/恢复全流程演练无误

### M1 7.3 内核点亮
- mt6899 dtsi + rodin dts 前移；**base DT 与原厂 dtbo overlay 对齐适配**（dtbo 不可刷写，LK 仍按 panel id 合并原厂 overlay）；pinctrl / clk / PMIC / GPIO / UFS / UART 前移到 7.3；gki_defconfig(6.18) 起步 + Ubuntu fragment（binder/ashmem 默认关闭）
- 调试通道（无 UART，已确认）：
  - **ramoops/pstore**：DT 前移时携带 MTK 预留内存 + ramoops 节点，console-ramoops 取回早期 dmesg；pstore 分区作持久后备
  - **USB gadget**：mtu3 dual-role + configfs，NCM 网口（SSH/dmesg）+ ACM 串口（console）
  - 屏幕通道 M3 后补上
- initramfs 最小版：busybox、挂根、rescue shell
- **验收**：7.3 内核启动到 initramfs shell，pstore 能取回完整 dmesg，USB 网络口可用
- 风险：vendor 驱动 6.18→7.3 API churn（IOMMU/SMI、drm、of_graph、sysctl、sched 等）逐件对账

### M2 Ubuntu Desktop rootfs 启动
- rootfs 构建器产出 26.04 LTS arm64 Desktop 根（debootstrap + 定制层，支持 headless 调试模式）；img2simg 后 `fastboot flash userdata`；首启动扩容填满 userdata（256/512 双机型自适应）；connsys/gpu 固件（从官方 ROM 提取或分区只读加载）先放进 /lib/firmware（暂不验证功能，绝不刷写固件分区）
- **验收**：USB NCM 下 SSH 进入完整 Ubuntu，systemd 到 login，apt 可用

### M3 显示
- MTK DDP/DSI DRM + 面板驱动前移（最大单项前移工程）；面板 overlay 来自原厂 dtbo（不可改），参数调整只落在 base DT / 驱动侧；先内核 console 上屏（fbcon/DRM），`modetest` 验证，再 Wayland/compositor 起来（llvmpipe 软渲）
- **验收**：GNOME 桌面以软件渲染上屏
- 风险：面板参数/时序对不上时与 6.18 oracle 逐寄存器对照

### M4 触摸 → 桌面形态达成
- vendor 触控驱动前移，GNOME 交互可用
- **验收**：手指可操作完整桌面——**目标形态（Desktop rootfs）在此达成**

### M5 GPU 加速
- panthor 对 G720 (r49p1) 支持验证 → mali_csffw.bin 提取加载 → Mesa panvk（版本闸门）→ GLES/Vulkan
- **验收**：glmark2/es2gears 硬件加速运行
- 风险：panvk 对 G720 成熟度未知；回退方案 = 维持 llvmpipe，或评估 vendor mali_kbase 路线（用户态难落地，优先级最低）

### M6 无线（WiFi/BT）
- 优先尝试上游 mt76（MT7961 系）+ connsys 固件从 ROM 提取入 /lib/firmware（分区只读不动）；不通则回退 vendor conninfra 前移（其按分区只读加载固件的路径保持）；BT 走 hciuart/conninfra
- **验收**：WiFi 稳定吞吐 + 蓝牙配对
- 风险：集成式 connsys 的固件/驱动配对关系，先在 6.18 oracle 上记录版本组合

### M7 电源
- 电量计/充电（PMIC vendor 驱动前移）、thermal、cpufreq/EAS
- **验收**：GNOME 电量图标正确、充电可控、高温不炸
- 风险：充电策略涉及安全，先用保守参数

### M8 音频
- MTK ASoC machine driver 前移 + audio_dsp 固件与 topology 参数提取
- **验收**：扬声器 + 耳机输出、内录可用
- 风险：**全路线最难啃的闭源面**，DSP 路由参数可能要从 6.18 运行态 dump

### M9 suspend 与打磨
- suspend/resume、亮度/音量键、传感器（加速度/光感）、GNOME 电源策略
- **验收**：休眠唤醒稳定，日常可用

### Stretch（不在本路线图承诺范围）
modem/通话/短信、相机 mtkcam、NFC、GNSS、AVF/pvmfw、Waydroid

## 前向移植方法论

1. **oracle 对照法**：每个子系统动手前，先在 _b 槽的 6.18 环境上记录行为基线（寄存器/日志/功能表现），7.3 上复现到一致才算完成
2. **提交纪律**：在 `linux-7.3.y` 直接堆 commit；每个前移/每件 API 对账一个独立 commit，message 说明依据，保证可 revert/可 bisect
3. **配置策略**：6.18 gki_defconfig 为起点 + `ubuntu.fragment`；Rust/ashmem/binder 默认关闭减变量

## 风险登记

| # | 风险 | 对策 |
|---|---|---|
| 1 | 无 UART，早期启动不可见 | ramoops/pstore + USB gadget 双通道在 M1 前落实；rootfs headless 模式兜底 |
| 2 | boot 64 MiB 容量吃紧（44 MB 起步，7.3 只会更大） | 优先调压缩参数；超限则把非启动关键驱动改 =m 放 vendor_boot ramdisk |
| 3 | 7.3 前向移植与 bring-up 耦合 | 严格 oracle 对照法，M1 拆小步 |
| 4 | G720 主线 GPU 支持不成熟 | M5 设闸门，llvmpipe 常备 |
| 5 | 音频闭源 DSP | 放最后（M8），预留运行态 dump 方案 |
| 6 | 双槽均为自编译内核，无 stock boot 兜底 | 完整官方 ROM 线刷恢复；刷写前先验证线刷链路可用 |
| 7 | dtbo / 固件分区不可刷写 | base DT 对齐原厂 overlay；固件一律只读加载（分区直读或 /lib/firmware） |

## 许可

内核与 vendor 前移遵循 GPL-2.0（派生自小米 GPL OSS 发布）；initramfs 与构建脚本 MIT。固件 blob 归属各自权利人，仅以提取脚本 + 构建产物形式分发，不入库。
