# 魅族 21 KernelSU 临时 Root 记录

[English](README.md) | [中文](README_ZH.md)

这个仓库保存的是魅族 21 通过 GhostLock 适配 KernelSU 临时 root 时用到的偏移文件、APK、提取工具和复现记录。这个方案不需要解锁 Bootloader，也不需要刷入 `boot.img`。

## 设备信息

- 型号：MEIZU 21
- 设备代号：meizu21
- SoC：Qualcomm SM8650
- Android：14
- 系统指纹：`meizu/meizu_21_CN/meizu21:14/UKQ1.230917.001/1733904967:user/release-keys`
- 增量版本：`1733904967`
- 内核版本：`6.1.79-android14-11-maybe-dirty`
- 提取时当前槽位：`_a`
- 固件包：Flyme 11.2.1.0A

## 文件说明

- `offsets/offsets-meizu21.json`
  - GhostLock App 里直接导入的偏移文件。
- `release/GhostLock-release.apk`
  - 本次测试使用的 GhostLock APK。
- `ghostlock-kernel-table/6.1.79-android14-11-maybe-dirty/offsets.h`
  - 使用 `ghostlock-extract --register` 生成的内置偏移表。如果后续要重新编译 GhostLock，可以把这份表并入源码。
- `tools/windows/ghostlock-extract.exe`
  - 本次在 Windows 上编译出的 GhostLock 偏移提取器。
- `SHA256SUMS.txt`
  - 已保留文件和源镜像的 SHA256 校验值。

仓库没有提交完整 OTA、`payload.bin`、`boot.img`、`vendor_boot.img` 等大文件。它们体积太大，也不是日常导入 GhostLock 必需的文件；README 里只记录提取方式和关键哈希。

## 适配过程

先从匹配当前系统版本的 OTA 中拿到 `payload.bin`。这个 OTA 里包含了 GhostLock 分析需要的关键分区：

- `boot`
- `init_boot`
- `vendor_boot`
- `xbl_config`

使用 `payload-dumper-go` 提取镜像：

```powershell
.\payload-dumper-go.exe -p boot,init_boot,vendor_boot,xbl_config -o .\extracted .\payload.bin
```

然后从 YuKongA/ghostlock-app 编译 `tools/extract_rs` 里的提取器。本次是在 Windows 上用便携 Rust 和 MinGW 编译完成的。

生成可导入 JSON 的命令：

```powershell
.\tools\extract_rs\target\release\ghostlock-extract.exe `
  <firmware-workdir>\extracted\boot.img `
  --xbl-config <firmware-workdir>\extracted\xbl_config.img `
  --format json `
  --out <firmware-workdir>\offsets-meizu21.json
```

提取器从 `boot.img` 中恢复了 kallsyms 和 BTF，并在写出偏移前检查了 GhostLock 依赖的漏洞原语。关键输出如下：

```text
info: kallsyms layout pre-6.4 recovered (96641 symbols)
info: (CVE-2026-43499 primitive present (remove_waiter@0xf8d8d4 still uses current)
info: pselect chain __arm64_sys_pselect6->core_sys_select ... shift=1
info: nf_logger loggers=0x1e82968 nfulnl_logger=0x1e82a28 ulog=1 slot=0x1e82970
info: sizeof(mm_struct)=0x3C0 (MM_STRUCT_SZ=0x500 in src/core/common.h)
```

生成的 JSON 关键字段：

```text
release: 6.1.79-android14-11-maybe-dirty
compact_waiter: 1
pselect_waiter_shift: 1
kernel_phys_load: 0xa8000000
```

同时也把这台设备的偏移注册进本地 GhostLock 源码，生成了内置表：

```powershell
.\tools\extract_rs\target\release\ghostlock-extract.exe `
  <firmware-workdir>\extracted\boot.img `
  --xbl-config <firmware-workdir>\extracted\xbl_config.img `
  --register
```

## 使用方法

先确认手机当前运行的内核版本必须完全一致：

```powershell
adb shell uname -r
```

期望输出：

```text
6.1.79-android14-11-maybe-dirty
```

把偏移文件推到手机：

```powershell
adb push .\offsets\offsets-meizu21.json /sdcard/Download/offsets-meizu21.json
```

打开 GhostLock App，选择导入 `offsets-meizu21.json`，然后在 App 中执行临时 root。这个流程走的是 GhostLock 临时 root 路线，不需要刷 `boot.img`，也不需要解锁 BL。

## 注意事项

提取器有一条布局警告：

```text
warning: waiter fits at the last usable word (shift=3); wake_state falls outside the copied fd_set and relies on the kernel zero-initialising it
```

实测这台魅族 21 已经可以拿到 root，但这类 race 型临时 root 和系统负载、时机有关，失败时可以多尝试几次。

本适配只对应 `6.1.79-android14-11-maybe-dirty`。如果系统升级后 `adb shell uname -r` 发生变化，需要重新用对应版本 OTA 的 `boot.img` 和 `xbl_config.img` 提取偏移。

上游项目：https://github.com/YuKongA/ghostlock-app
