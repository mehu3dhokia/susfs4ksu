## 引言 ##
- 一个用于 KernelSU 的内核补丁和用户空间模块，用于隐藏 root 权限。
- 用户空间工具 `ksu_susfs` 和 ksu 模块需要基于 **susfs 补丁的内核** 才能运行。

# 警告 #
- 此代码仅用于实验目的，可能损坏系统或导致性能下降，**您已被明确警告！**

## 兼容性 ##
- susfs 内核补丁可能因内核版本不同（甚至相同版本）而异，您可能需要为自己的内核创建定制补丁。

## 补丁指南（仅适用于 GKI 内核且基于官方 Google 源码构建）
**- 前置条件 -**
1. 所有 susfs 补丁均基于 **weishu 维护的官方 KernelSU（含 release tag）**。请使用 **release tag** 克隆其仓库，并确保本 susfs 分支也使用 **release tag** 或包含 **"Bump version to vX.X.X"** 的提交，以获得最佳补丁效果。
2. 只要启用 `KSU_SUSFS_HAS_MAGIC_MOUNT` 功能，SUSFS 即可支持 Magisk Mount (Magic Mount) 的 AUTO_ADD_ 特性。

**- 应用 SUSFS 补丁 -**
1. 严格遵循 KSU 官方构建指南：`https://kernelsu.org/guide/how-to-build.html`，内核根目录为 `$KERNEL_REPO/common`，KernelSU 需克隆至 `$KERNEL_REPO`（**务必使用 tag 版本克隆**）。
2. 克隆本 susfs 分支时使用 **release tag** 或包含 **"Bump version to vX.X.X"** 的提交（通常更稳定）。
3. 运行 `cp ./kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch $KERNEL_REPO/KernelSU/`
4. 运行 `cp ./kernel_patches/50_add_susfs_in_kernel-<内核版本>.patch $KERNEL_REPO/common/`
5. 运行 `cp ./kernel_patches/fs/* $KERNEL_REPO/common/fs/`
6. 运行 `cp ./kernel_patches/include/linux/* $KERNEL_REPO/common/include/linux/`
7. 运行 `cd $KERNEL_REPO/KernelSU` 后执行 `patch -p1 < 10_enable_susfs_for_ksu.patch`
8. 运行 `cd $KERNEL_REPO/common` 后执行 `patch -p1 < 50_add_susfs_in_kernel.patch`（**若补丁失败需手动修复**）。
9. 如需支持其他 KSU 管理器变体，需在 `KernelSU/kernel/apk_sign.c` 的 `ksu_is_manager_apk()` 函数中添加其哈希值与长度。
10. 内核构建前必须启用 `CONFIG_KSU` 和 `CONFIG_KSU_SUSFS`。其他 SUSFS 功能默认可能关闭，可通过 `menuconfig`、`kernel defconfig` 或修改 `$KernelSU_repo/kernel/Kconfig` 中 `config KSU_SUSFS_` 的 `default [y|n]` 选项启用。
11. 若内核已应用 **KSU non-kprobe hook 补丁**，则必须 **禁用** `CONFIG_KSU_SUSFS_SUS_SU` 选项。
12. 若 KernelSU 管理器使用 Magic Mount，需启用 **`KSU_SUSFS_HAS_MAGIC_MOUNT`** 选项以支持 AUTO_ADD_ 特性。
13. 对于 **Android 14 及更高版本的 GKI 内核**（基于 Google 源码构建），必须删除文件：
   - `$KERNEL_REPO/common/android/abi_gki_protected_exports_aarch64`
   - `$KERNEL_REPO/common/android/abi_gki_protected_exports_x86_64`
   否则 WiFi 等模块将失效（或直接移除所有相关文件）。
14. 刷入新编译的 GKI boot.img 前：
   - 需在 `$KERNEL_REPO/build/kernel/build_utils.sh` 的 `build_gki_boot_images()` 函数中修正或硬编码 `local spl_date` 以匹配设备当前的安全补丁版本。
   - 或使用 magiskboot 解包/重打包内核至原厂 boot.img。
15. 构建并刷入内核。
16. 编译错误请参考下方 **[已知编译问题]** 章节。
17. 其他构建技巧请参考 **[其他构建技巧]** 章节。

## 构建 ksu_susfs 用户空间工具 ##
1. 运行 `./build_ksu_susfs_tool.sh` 构建 `ksu_susfs` 工具，ARM64/ARM 二进制文件将复制至 `ksu_module_susfs/tools/`。
2. 可将编译后的 `ksu_susfs` 推送至 `/data/adb/ksu/bin/`，以便在 adb root shell、Termux root shell 或自定义 KSU 模块中直接运行。

## 构建 sus_su 用户空间工具（已弃用）
**--重要提示--**
- `sus_su` 工具现已弃用，因部分新小米设备存在名为 "mrmd" 的 root 检测服务（由 init 进程启动）。由于 overlayfs 挂载的 sus_su 无法为 init 进程卸载，故会被检测到（除非有更优卸载方案）。

**--模式 1 指南（已弃用）--**
- `sus_su` 是通过向 susfs FIFO 驱动发送请求获取 root shell 的可执行文件，**仅适用于启用 kprobe hook 的 KSU**（kprobe 禁用时不可用）。
- 仅 KSU 管理器授权 root 的应用可执行 'su' 命令。
- 为最佳兼容性，需 overlayfs 支持第三方应用执行 'su' 获取 root shell。
- 详见模块模板的 `service.sh`。

1. 运行 `./build_sus_su_tool.sh` 构建 sus_su，ARM64/ARM 二进制文件将复制至 `ksu_module_susfs/tools/`。
2. 在 `service.sh` 中取消注释 `#enable_sus_su` 以启用 sus_su。
3. 运行 `./build_ksu_module.sh` 构建模块并重新刷入。

**--模式 2 指南--**
- 直接运行 `ksu_susfs sus_su 2` 禁用核心 kprobe hooks 并启用 su 的内联 hooks。

## 构建 susfs4ksu 模块 ##
- 此 ksu 模块仅为功能演示模板。
- 安装时会复制 `ksu_susfs` 和 `sus_su` 工具至 `/data/adb/ksu/bin/`。

1. `ksu_susfs` 可在任意阶段脚本运行（`post-fs-data.sh`/`services.sh`/`boot-completed.sh`）。
2. 运行 `./build_ksu_module.sh` 构建 SUSFS KSU 模块。

## ksu_susfs 用法及支持特性 ##
- 在 root shell 中运行 `ksu_susfs` 查看详细用法。
- 应用 susfs 补丁后，支持的特性详见 `$KernelSU_repo/kernel/Kconfig`。

## 其他构建技巧 ##
- **移除内核版本字符串中的 "-dirty"**：
  - 修改 `$KERNEL_ROOT/scripts/setlocalversion`，将所有 `printf '%s' -dirty` 替换为 `printf '%s' ''`
- **硬编码内核版本字符串**：
  - 修改 `$KERNEL_ROOT/scripts/setlocalversion`，将最后一行 `echo "$res"` 替换为（示例）：`echo "-android13-01-gb123456789012-ab12345678"`
- **硬编码内核版本信息**：
  - 修改 `$KERNEL_ROOT/scripts/mkcompile_h`，将行 `UTS_VERSION="$(echo $UTS_VERSION $CONFIG_FLAGS $TIMESTAMP | cut -b -$UTS_LEN)"` 替换为（示例）：`UTS_VERSION="#1 SMP PREEMPT Mon Jan 1 18:00:00 UTC 2024"`
- **硬编码 /proc/version 显示的内核编译信息**：
  - 修改 `$KERNEL_ROOT/scripts/mkcompile_h`，在行 `UTS_VERSION="$(echo $UTS_VERSION $CONFIG_FLAGS $TIMESTAMP | cut -b -$UTS_LEN)"` 后添加（示例）：
    ```bash
    LINUX_COMPILE_BY=build-user
    LINUX_COMPILE_HOST=build-host
    ```
- **伪装 /proc/config.gz 为原厂配置**：
  1. 确保设备运行原厂 ROM 和内核。
  2. 通过 adb shell 或 root shell 从设备拉取原厂 `/proc/config.gz`。
  3. 使用 `gunzip` 解压，复制为 `$KERNEL_ROOT/arch/arm64/configs/stock_defconfig`。
  4. 修改 `$KERNEL_ROOT/kernel/Makefile`。
  5. 将行 `$(obj)/config_data: $(KCONFIG_CONFIG) FORCE` 替换为：`$(obj)/config_data: arch/arm64/configs/stock_defconfig FORCE`

## 已知编译问题 ##
1. **错误：'struct yyyyyyyy' 中无成员 'android_kabi_reservedx'**
   - 因内核版本 <4.19 的结构体通常不含 `u64 android_kabi_reservedx;`，某些 4.19≤内核≤5.4 的 GKI 内核也可能缺失此成员（部分自定义内核已禁用）。
   - **解决方案**：手动在对应结构体定义末尾添加成员（如 `u64 android_kabi_reserved1;`）。可参考本仓库 `kernel-4.14`/`kernel-4.9` 分支的补丁差异。

## 其他已知问题 ##
- **部分文件管理器无法正常显示添加到 `sus_path` 的 '/sdcard' 子目录**：
  1. 确保文件管理器应用已被 KSU 授予 root 权限（因 `sus_path` 仅对无 root 权限的进程 UID 生效）。
  2. **强烈不建议**将 '/sdcard' 或 '/storage/emulated/0' 子目录加入 `sus_path`。因文件管理器可能通过 Android API（调用系统媒体服务如 Google Provider）检索文件列表，此时调用 UID 会被视为无 root 权限而隐藏内容。除非应用先获取 root 权限再直接列出文件（不使用 Android API）。

## 致谢 ##
- KernelSU 官方：https://github.com/tiann/KernelSU
- KernelSU 分支：https://github.com/5ec1cff/KernelSU
- @Kartatz：原始提交灵感（https://github.com/Dominium-Apum/kernel_xiaomi_chime/pull/1/commits/74f8d4ecacd343432bb8137b7e7fbe3fd9fef189）

## Telegram 联系 ##
- @simonpunk

## 支持作者 ##
- PayPal：kingjeffkimo@yahoo.com.tw
- BTC：bc1qgkwvsfln02463zpjf7z6tds8xnpeykggtgk4kw
