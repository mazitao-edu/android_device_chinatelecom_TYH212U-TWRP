# Tianyi1Hao2022 TYH212U (ARES) - TWRP 修复版

基于 [Xpsoted/android_device_chinatelecom_ARES-TWRP](https://github.com/Xpsoted/android_device_chinatelecom_ARES-TWRP) 修复版，针对天翼一号 2022 **臻情版** (展锐 UMS9620_2H10)。

## 修复了什么

原作者 README 承认有 3 个未解决问题，本修复版针对性修复 + 额外发现 2 个隐藏 bug：

### 1. ✅ 开机报"无权限挂载 Vendor"（AVB 验证失败）
- `BoardConfig.mk`: `BOARD_AVB_MAKE_VBMETA_IMAGE_ARGS` 由 `--flags 0` 改为 `--flags 2`（禁用 hashtree + verification）
- `recovery/root/first_stage_ramdisk/fstab.ums9620_2h10`: 删除 `avb=vbmeta_xxx` 强制验证（直接根因）

### 2. ✅ Fastbootd 无法启动
- `system.prop`: 添加 `ro.boot.virtual_ab=true`（展锐 ums9620 是 Virtual A/B，缺这个 fastbootd 直接退出）
- `device.mk`: 移除 minimal manifest 里没源码的 `bootctrl.ums9620`（导致编译失败）
- `device.mk`: 移除不工作的 `android.hardware.fastboot@1.0-impl-mock`

### 3. ✅ 解密半残 - 找到真正根因
**原作者 init.rc 启动的是不存在的服务**！
- 原代码: `vendor.keymint-beanpod` → `/vendor/bin/hw/android.hardware.security.keymint@1.0-service.beanpod`（**此文件不存在**）
- 实际 vendor/bin/hw/ 下只有: `android.hardware.keymaster@4.1-unisoc.service`
- 修复: `init.recovery.ums9620_2h10.rc` 改为启动 `vendor.keymaster-unisoc`，使用实际存在的二进制
- 额外: `on post-fs-data` 创建 `/data/vendor/keymaster` 工作目录

### 4. ✅ 臻情版 kernel 不兼容（作者明确警告"臻情需要换内核"）
- 作者 `prebuilt/kernel` MD5: `bb0e07d9...`（不是原厂提取的，可能是源码编译）
- 本版 `prebuilt/kernel` MD5: `744190f1...`（**从用户原厂 boot.img 提取**，100% 兼容）
- `prebuilt/dtb.img` MD5 一致（`b030eec6...`），无需替换但已同步

### 5. ✅ userdata 文件系统类型不匹配
- 原代码 `BoardConfig.mk`: `BOARD_USERDATAIMAGE_FILE_SYSTEM_TYPE := ext4`（正确）
- 但 `recovery.fstab` 里 /data 写成了 f2fs（作者从其他设备复制模板的错误）
- 实际设备: ext4
- 修复: `recovery.fstab` 里 /data 从 f2fs 改为 ext4，去掉 f2fs 特有选项（inline_xattr/inline_data/fsync_mode）

### 6. ✅ build fingerprint 错误
- 原代码: `UNISOC/ums9620_2h10_ctcc/...`（通用名）
- 修复: `ChinaTelecom/TYH212U/ARES:11/RP1A.201005.001/1697029118:user/release-keys`（正确品牌名）

## 推荐使用方式：fastboot boot 临时启动（不破坏原系统）

```bash
fastboot boot boot.img
# 重启即回原系统
```

## 编译方法

用 [kinguser981/TWRP-Recovery-Builder-2024](https://github.com/kinguser981/TWRP-Recovery-Builder-2024) GitHub Actions 编译：

| 参数 | 值 |
|---|---|
| MANIFEST_BRANCH | `twrp-12.1` |
| DEVICE_TREE | 你的 fork URL |
| DEVICE_TREE_BRANCH | `main` |
| DEVICE_PATH | `device/chinatelecom/ARES` |
| DEVICE_NAME | `ARES` |
| BUILD_TARGET | `boot` |

## 硬件信息

- 设备：天翼一号 2022 臻情版 (TYH212U)
- 代号：ARES
- 芯片：展锐 UMS9620_2H10
- 原厂系统：Android 11 (RP1A.201005.001)
- prebuilt/kernel 来源：用户原厂 boot.img（2023-10-11 版本，fingerprint 1697029118）
- 加密：FBE v2 (aes-256-xts:aes-256-cts) + metadata encryption
- userdata：ext4
- 分区：Virtual A/B + 动态分区

## 已知限制

- prebuilt/kernel 是 2023-10-11 版本（fingerprint 1697029118），用户已验证此 kernel 通过面具注入在当前系统正常工作，版本兼容性已确认
- 解密需要正确输入锁屏密码（FBE 加密）
- 如果编译后解密仍失败，检查 keymaster@4.1-unisoc.service 是否能正常启动（adb shell logcat | grep keymaster）
