# Xiaomi 14 (SM8650) Gunyah VMID 修复内核

## 项目说明

本项目基于 [ABK (AnyBase Kernel)](https://github.com/xingguangcuican6666/ABK) 的完整构建流程，为小米14 (SM8650 / 骁龙8 Gen 3) 编译带有 **Gunyah VMID 修复** 的 GKI 内核。

## 问题描述

骁龙8 Gen 3 (SM8650) 的 Gunyah 虚拟机管理程序存在 VMID 映射问题：
- SM8650 上 RM（Resource Manager）运行在独立 VM，VMID 大于 `QCOM_SCM_MAX_MANAGED_VMID (0x3F)`
- 原始内核代码硬编码 HLOS VMID，导致 SCM 调用拒绝，虚拟机无法启动
- 表现为 DroidVM 等虚拟化应用报错：`No such device (os error 19)` 或 `Out of memory (os error 12)`

> **注意**：骁龙8 Elite (SM8750) 及以后的芯片已重构 Gunyah 代码，不存在此问题。

## 快速开始

### 直接使用本仓库（推荐）

1. **Fork 本仓库**
2. 进入 **Actions** 选项卡，启用 GitHub Actions
3. 选择 **kernel-custom** 工作流，点击 **Run workflow**
4. 参数保持默认即可（`use_gunyah=true`）
5. 等待构建完成（约1-2小时），下载 Artifacts 中的内核包
6. 构建完成后，在 Release 页面下载：
   - **AnyKernel3 zip**（内核，刷入 boot 分区）
   - **配套 KernelSU Manager APK**（安装到手机，需与内核的 KernelSU 版本匹配）
7. **确保 init_boot（ramdisk）干净**：不能带 Magisk 修补，否则 KernelSU 会报"Magisk 冲突"并禁用所有模块（清理方法见 FAQ）
8. 刷入方式：
   - **TWRP**：直接刷入 AnyKernel3 zip
   - **fastboot**：`fastboot flash boot 内核.img`（需先提取 boot.img）
9. 重启后打开 KernelSU Manager，启用需要的模块（如 gh-hugepage-reserve）

### 构建参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `android_version` | android14 | Android 版本 |
| `kernel_version` | 6.1 | 内核版本 |
| `sub_level` | 138 | 内核子版本号 |
| `os_patch_level` | 2025-06 | 安全补丁级别 |
| `kernelsu_variant` | Official | KernelSU 变体 |
| `kernelsu_branch` | Stable(标准) | KernelSU 分支 |
| `use_gunyah` | true | 启用 Gunyah VMID 修复 |
| `use_ntsync` | false | 启用 NTsync 补丁 |
| `cancel_susfs` | true | 禁用 SUSFS |
| `virtualization_support` | 678 | DroidSpaces 虚拟化槽位 |

## 内核包含的修复

### Gunyah VMID 修复（核心）

在内核编译时修改 `gunyah_qcom.c` 中的 VMID 处理逻辑：
- 通过 `qcom_scm_map_vmid()` 查询真实 VMID 映射
- 当 VMID 超过 `QCOM_SCM_MAX_MANAGED_VMID (0x3F)` 时，使用正确的 SCM 调用路径
- 使用 `DEFINE_MUTEX` 保护 VMID 查询的并发安全

### KernelSU

集成 [KernelSU](https://github.com/tiann/KernelSU)，提供内核级 root 支持。

### DroidSpaces 虚拟化支持（可选）

通过 `virtualization_support` 参数启用 [DroidSpaces](https://github.com/ravindu644/Droidspaces-OSS) 补丁，增强虚拟化能力。

## kvcalloc 模块（突破 2GB 内存限制）

除了内核内置的 VMID 修复外，还需要 **kvcalloc 模块** 来突破单 VM 2GB 内存限制。

该模块通过 kprobe 劫持 Gunyah 驱动的内存分配函数，将 `kcalloc` 替换为 `kvcalloc`，解决大内存 VM 创建时的 OOM 问题。

**下载地址**：[`modules/gunyah_kvcalloc_fix.zip`](modules/gunyah_kvcalloc_fix.zip)

**安装方法**：通过 KernelSU/Magisk Manager 刷入 ZIP，重启即可。详见 [`modules/README.md`](modules/README.md)。

> **注意**：该模块由 **秋秋**（QQ: 3487467850）编译，仅适用于 `6.1.138` 内核版本。

## DroidVM 使用注意（骁龙 8 Gen 3 专属）

使用 DroidVM 时，请务必注意以下两点：

### 1. 关闭"预分配大页"（mthp）

DroidVM 的 **"预分配大页"（Prepare Lend mTHP）** 选项（编辑 VM → 虚拟机选项，默认"分块 chunked"）在骁龙 8 Gen 3 上会导致**启动虚拟机时设备卡死并重启（内核 panic）**。

**解决办法**：编辑 VM → 虚拟机选项 → **"预分配大页"改为"禁用"（Disabled）**。

> **原因**：8 Gen 3 的 Gunyah 固件不接受 crosvm 用 THP collapse 生成的 2MB 大页做 lend，hypervisor 处理时触发异常导致整机复位。8 Elite（SM8750）不受影响。

### 2. init_boot 必须干净（无 Magisk 修补）

init_boot（ramdisk）**不能包含 Magisk 修补**（特征：存在 `.backup/.magisk`、`overlay.d/sbin/magisk*.xz`），否则 KernelSU 会检测到 Magisk 并**禁用所有模块**（提示"与 Magisk 冲突"）。

**解决办法**：刷入干净的 init_boot。清理命令（在已 root 的电脑/设备上）：

```bash
# 从设备导出当前 init_boot，用 magiskboot 清理后刷回
magiskboot unpack init_boot.img
magiskboot cpio ramdisk.cpio restore
magiskboot repack init_boot.img
# 得到 clean 的 new-boot.img，fastboot flash init_boot new-boot.img
```

## 已知限制

- 需要同时刷入 **kvcalloc 模块** 才能突破单 VM 2GB 内存限制
- 模块需要与内核版本匹配，内核更新后需要重新编译模块

## 常见问题（FAQ）

### Q: 启动 DroidVM 时设备卡住并重启
- **原因**：DroidVM 的"预分配大页"（mthp）在 8 Gen 3 上触发 Gunyah hypervisor 异常
- **解决**：DroidVM 编辑 VM → 虚拟机选项 → "预分配大页"改为**禁用**

### Q: KernelSU 显示"与 Magisk 冲突"，所有模块不可用
- **原因**：init_boot（ramdisk）被 Magisk 修补过（存在 `.backup/.magisk`、`overlay.d/sbin/magisk*.xz`），KernelSU 检测到 Magisk 后自动禁用模块
- **解决**：刷入干净的 init_boot（见上文"init_boot 必须干净"，用 `magiskboot cpio ramdisk.cpio restore` 清理）

### Q: 虚拟机报 `No such device (os error 19)` 或 `Out of memory (os error 12)`
- **原因**：内核缺少 Gunyah VMID 修复，SCM 内存共享调用被拒绝
- **解决**：刷入本仓库构建的内核（构建时启用 `use_gunyah=true`）

### Q: 单 VM 内存超过 2GB 报错
- **原因**：Gunyah 大内存分配走 `kcalloc`，需要大块连续物理内存，超出后 OOM
- **解决**：刷入 [kvcalloc 模块](modules/gunyah_kvcalloc_fix.zip)（需与 6.1.138 内核匹配）

## 项目结构

```
.github/
  workflows/
    build.yml          # 核心构建流程（基于 ABK，含 Gunyah VMID 修复）
    kernel-custom.yml   # 自定义构建触发器
    get-manager.yml     # KernelSU Manager 下载
  scripts/
    resolve-ksu-ref.sh  # KernelSU 分支解析
    download-manager-from-actions.sh  # Manager APK 下载
config/
    config              # stock defconfig
    zram.config         # zram 配置
modules/
    gunyah_kvcalloc_fix.zip   # kvcalloc 修复模块（KernelSU/Magisk ZIP）
    gunyah_kvcalloc_mod.ko    # kvcalloc 修复模块（独立 .ko 文件）
    README.md                 # 模块说明
```

## 相关链接

- [ABK 项目](https://github.com/xingguangcuican6666/ABK)
- [DroidVM](https://github.com/Droid-VM/DroidVM)
- [DroidVM Wiki - SM8650 已知问题](https://droidvm.github.io/en/wiki/troubleshooting/common-issues.html)
- [KernelSU](https://github.com/tiann/KernelSU)
- [gh-hugepage-reserve 模块](https://github.com/Droid-VM/gh-hugepage-reserve)

## 致谢

- **秋秋** (QQ: 3487467850) — 编译 kvcalloc 修复模块，突破单 VM 2GB 内存限制
- [ABK](https://github.com/xingguangcuican6666/ABK) — 内核构建流程
- [DroidVM](https://github.com/Droid-VM) — 虚拟化平台

## 许可证

本项目基于 GPL-2.0 许可证开源。
