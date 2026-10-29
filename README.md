# Ubuntu Linux for Rodin — 移植路线图

状态：讨论稿 v0.1（2026-10-30，待评审）
设备：Redmi Turbo 4（国行）/ POCO X7 Pro，代号 **rodin**
SoC：MediaTek MT6899（天玑 8400-Ultra），Mali-G720 MC7（r49p1），UFS 存储，A/B 双槽

---

## 0. 已定决策（2026-10-30 讨论）

| 决策点 | 结论 |
|---|---|
| 内核基线 | **直接在 upstream v7.3（org/linux）上前向移植** vendor 栈；不做 6.18 中间启动产物 |
| rootfs 落位 | **userdata 整盘独占**（ext4 起步）；super 完全不动，保留随时回刷 HyperOS/LineageOS 的能力 |
| 目标形态 | **Ubuntu Desktop rootfs（26.04 LTS，GNOME）**，首启动即桌面 rootfs |
| 6.18 栈角色 | 保留在 omajili-manbu 名下不动；作为**已点亮的行为基线 / 对照 oracle**，不在 org 内重复建设 |

## 1. 设备底账（依据 MT6899_Android_scatter + super_parts，本地 `E:\work\hyperos\rodin_images_OS3.0.304.0.WOJCNXM_16.0\images`）

### 1.1 与 Ubuntu 方案相关的分区

| 分区 | 每槽大小 | Ubuntu 方案 |
|---|---|---|
| boot_a/b | 64 MiB | 7.3 内核 Image.lz4 + 空 GKI ramdisk（boot header v4）。6.18 基线 Image.lz4 ≈ 44 MB，上限 60 MB |
| vendor_boot_a/b | 64 MiB | dtb + **Ubuntu initramfs**（vendor_ramdisk 形式打包） |
| dtbo_a/b | 8 MiB | 面板 overlay；LK 按 panel id 选 idx（`androidboot.dtbo_idx`），需与 LK 约定保持一致 |
| init_boot_a/b | 8 MiB | Android 通用 ramdisk，Ubuntu 不使用，保留原样 |
| vbmeta / vbmeta_system / vbmeta_vendor | 1 MiB ×3 | 刷 `--disable-verity --disable-verification` 版本 |
| super | 11.25 GiB | **不动**。动态分区现状：product 4.9G / vendor 2.0G / system_ext 936M / odm 862M / system 880M / mi_ext 64M / 两个 dlkm |
| userdata | scatter 标 12 GiB（最小变体下限） | **Ubuntu rootfs**（ext4，首启动 resize 填满）。真机实际大小以 `blockdev --getsize64` 实测为准 |
| pstore | 133 MiB | ramoops 持久内核日志——7.3 调试通道之一 |
| modem | 200 MiB ×2 | 保留（本路线图内不驱动基带） |

固件分区全部保留不动：connsys_wifi/bt/gnss、audio_dsp、scp、sspm、mcupm、gpueb、vcp、apusys、mvpu_algo、ccu、dpm、pi_img、tee、gz、spmfw、mcf_ota。这些是后续 WiFi/BT/GPU/DSP 各阶段的固件来源（运行时由 /lib/firmware 加载或分区直读，方案见对应里程碑）。

### 1.2 启动链（Ubuntu 方案）

```
BootROM → preloader → LK(已解锁，rodin_lk_unlock_kit)
  → boot_a(Image.lz4) + vendor_boot_a(dtb + initramfs) + dtbo(idx)
  → Linux 7.3 → initramfs 挂 /dev/disk/by-partlabel/userdata → switch_root
  → systemd → GDM/GNOME
```

槽位策略：**Ubuntu 常驻 _a**，_b 保留 stock boot/vendor_boot 作为恢复通道（lk 已双槽写入解锁版）。

### 1.3 关键上游事实（2026-10-30 核查）

- upstream v7.3 的 `arch/arm64/boot/dts/mediatek/` **没有 mt6899**（mt68xx 系只有 mt6858/mt6878/mt6893 pinfunc）→ pinctrl、clk、PMIC、DTS 全部走前移，无上游可白嫖。
- v7.3 已有 panthor 驱动（Mali CSF 世代）；对 G720 (r49p1) 的实际支持状态需在 M5 专项验证。

## 2. 现有资产与角色

| 仓库 | 角色 |
|---|---|
| `omajili-manbu/Xiaomi_Rodin_Kernel_Enhance@bsp-rodin-c-rebase` | 6.18 已点亮内核树（android17-6.18-lts 基线）——**oracle + DT/vendor 源** |
| `omajili-manbu/MTK_kernel_device_modules@bsp-rodin-v-oss` | 设备模块前移源（569 模块名：538 内建 / 31 丢载 / 0 blob） |
| `omajili-manbu/MTK_kernel_modules@bsp-rodin-v-oss` | 技术 kmod 前移源（mali r49p1、conninfra、udc、mtkcam 等） |
| `omajili-manbu/rodin_lk_unlock_kit` | 解锁闭环（lk 303/304，注意 anti-rollback 警告） |
| `Ubuntu-linux-for-rodin/linux`（v7.3 裸导入，1 commit） | **目标内核基线**，rodin 工作全部落这里 |
| `Ubuntu-linux-for-rodin/initramfs`（仅 LICENSE） | 待建设：stage-1 |
| LineageOS 23.2 + OrangeFox(rodin) | 恢复/对照环境 |

三仓 rebase 点配对快照（2026-10-16）：kernel `52a8e4b4498a` · devmod `bc2dfc01+122` · kmod `991e9cd48ac7`。从 6.18 树前移任何代码时以此为准。

## 3. 目标仓库布局（提议，待确认）

```
Ubuntu-linux-for-rodin/
├── linux        v7.3 + rodin patch series（DT、vendor 前移、配置）
├── initramfs    stage-1：busybox 静态、按 partlabel 找根、rescue shell
├── build        (新建) mkbootimg 打包 / fastboot 刷写脚本 / CI
├── rootfs       (新建) Ubuntu 26.04 Desktop rootfs 构建器 + 固件提取脚本
└── docs         (新建或用 wiki) 分区底账 / 刷机指南 / 功能状态矩阵
```

固件 blob（connsys、mali_csffw 等）**只进构建产物，不进 git**——rootfs 仓提供从本地镜像/真机提取的脚本。

## 4. 里程碑

### M0 工作台就绪
- org/linux 建 `rodin` 工作分支；定补丁组织方式（见 §7 待确认）
- 交叉工具链：LLVM=1 全量构建，确认 7.3 要求的 clang 版本（6.18 用的 r596125 是否满足，不满足则升级）
- 刷机链路冒烟：用已验证的 6.18 Image 走一遍 fastboot 刷 boot/vendor_boot/vbmeta → 确认解锁机、刷写、恢复流程畅通（不产生新代码，纯粹验证工具链）
- **验收**：手动刷写/恢复全流程演练一遍无误

### M1 7.3 内核点亮
- 内容：mt6899 dtsi + rodin dts + panel dtbo 前移；pinctrl / clk / PMIC / GPIO / UFS / UART 前移到 7.3；gki_defconfig(6.18) 起步 + Ubuntu fragment（Android 组件 binder/ashmem 默认关闭，Waydroid 未来再说）
- 调试通道（本里程碑最大不确定性）：pstore/ramoops + USB gadget（mtu3 dual-role + configfs NCM/serial）双通道；UART 是否可用见 §7
- initramfs 最小版：busybox、挂根、rescue shell
- **验收**：7.3 内核启动到 initramfs shell，pstore 能取回完整 dmesg，USB 网络口可用
- 风险：vendor 驱动 6.18→7.3 API churn（IOMMU/SMI、drm、of_graph、sysctl、sched 等）逐件对账——沿用 6.6→6.18 的方法，每件一个 commit 说明

### M2 Ubuntu Desktop rootfs 启动
- 内容：rootfs 构建器产出 26.04 LTS arm64 Desktop 根（debootstrap + 定制层，支持 headless 调试模式）；img2simg 后 `fastboot flash userdata`；首启动 resize 填满；固件提取脚本先把 connsys/gpu 固件放进 /lib/firmware（暂不验证功能）
- **验收**：USB NCM 下 SSH 进入完整 Ubuntu，systemd 到 login，apt 可用
- 风险：Desktop 首启动变量多——构建器保留 headless profile 兜底

### M3 显示
- 内容：MTK DDP/DSI DRM + 面板驱动前移（最大单项前移工程）；先内核 console 上屏（fbcon/DRM），`modetest` 验证，再 Wayland/compositor 起来（llvmpipe 软渲）
- **验收**：GNOME 桌面以软件渲染上屏
- 风险：面板参数/时序对不上时与 6.18 树逐寄存器对照

### M4 触摸 → 桌面形态达成
- 内容：vendor 触控驱动前移，GNOME 交互可用
- **验收**：手指可操作完整桌面——**目标形态（Desktop rootfs）在此达成**

### M5 GPU 加速
- 内容：panthor 对 G720 (r49p1) 支持验证 → mali_csffw.bin 提取加载 → Mesa panvk（版本闸门）→ GLES/Vulkan
- **验收**：glmark2/es2gears 硬件加速运行
- 风险：panvk 对 G720 成熟度未知；回退方案 = 维持 llvmpipe，或评估 vendor mali_kbase 路线（用户态难落地，优先级最低）

### M6 无线（WiFi/BT）
- 内容：优先尝试上游 mt76（MT7961 系）+ 提取 connsys 固件；不通则回退 vendor conninfra 前移；BT 走 hciuart/conninfra
- **验收**：WiFi 稳定吞吐 + 蓝牙配对
- 风险：集成式 connsys 的固件/驱动配对关系，需在 6.18 oracle 上先记录版本组合

### M7 电源
- 内容：电量计/充电（PMIC vendor 驱动前移）、thermal、cpufreq/EAS
- **验收**：GNOME 电量图标正确、充电可控、高温不炸
- 风险：充电策略涉及安全，先用保守参数

### M8 音频
- 内容：MTK ASoC machine driver 前移 + audio_dsp 固件与 topology 参数提取
- **验收**：扬声器 + 耳机输出、内录可用
- 风险：**全路线最难啃的闭源面**，DSP 路由参数可能要从 6.18 运行态 dump

### M9 suspend 与打磨
- 内容：suspend/resume、亮度/音量键、传感器（加速度/光感）、GNOME 电源策略
- **验收**：合盖休眠唤醒稳定，日常可用

### Stretch（不在本路线图承诺范围）
modem/通话/短信、相机 mtkcam、NFC、GNSS、AVF/pvmfw、Waydroid

## 5. 7.3 前向移植方法论

1. **oracle 对照法**：每个子系统动手前，先在已点亮的 6.18 树上记录行为基线（寄存器/日志/功能表现），7.3 上复现到一致才算完成
2. **补丁纪律**：vendor 改动全部集中在 rodin patch series，不散改上游文件；为顺移 7.4+ 留余地
3. **逐件对账**：6.18→7.3 每个 API 变化一个独立 commit + commit message 说明依据（沿用 6.6→6.18 的既有实践）
4. **配置策略**：6.18 gki_defconfig 为起点 + `ubuntu.fragment`；Rust/ashmem/binder 默认关闭减变量

## 6. 风险登记

| # | 风险 | 对策 |
|---|---|---|
| 1 | 无 UART 时首启动不可见 | pstore + USB gadget 双通道在 M1 前落实；rootfs headless 模式兜底 |
| 2 | boot 64 MiB 容量吃紧（44 MB 起步，7.3 只会更大） | 优先调压缩参数；超限则把非启动关键驱动改 =m 放 vendor_boot ramdisk |
| 3 | 7.3 前向移植与 bring-up 耦合 | 严格 oracle 对照法，M1 拆小步 |
| 4 | G720 主线 GPU 支持不成熟 | M5 设闸门，llvmpipe 常备 |
| 5 | 音频闭源 DSP | 放最后（M8），预留运行态 dump 方案 |
| 6 | 变砖 | _b 槽保留 stock；OrangeFox + 本地全套镜像可恢复；遵守解锁 kit 的 anti-rollback 警告 |

## 7. 待确认问题

1. **UART 调试条件**：手上是否有可接 rodin 串口的手段（测试点/工装）？决定 M1 调试策略优先级
2. **真机存储配置**：256GB 还是 512GB？（确认 userdata 实际大小与 rootfs 预算）
3. **Ubuntu 版本**：26.04 LTS 确认？（24.04 亦可作为备选，软件源更保守）
4. **仓库布局**：§3 新建 build / rootfs / docs 三个仓是否同意？
5. **补丁组织**：org/linux 用独立 `rodin` 分支直接堆 commit，还是 main 之上维护 patch series（quilt/git format-patch）？
6. **6.18 oracle 的访问方式**：调试时真机怎么切回 6.18（切 _b 槽？还是另备一台可刷机器）？
