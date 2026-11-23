# 来源追溯（Provenance）——每个文件从哪来、哪些还不确定

编写设备树时克隆下来对照的上游仓库：

| 简称 | 仓库 | 分支 |
| --- | --- | --- |
| **LOS** | `LineageOS/android_device_xiaomi_mars` | `lineage-23.2` |
| **LOS-common** | `LineageOS/android_device_xiaomi_sm8350-common` | `lineage-23.2` |
| **TW** | `TeamWin/android_device_xiaomi_mars` | `android-11` |
| **TW-common** | `TeamWin/android_device_xiaomi_sm8350-common` | `android-11` |
| **NEB** | `nebrassy/device_xiaomi_mars-TWRP` | `android-11` |
| **FOX-KOTAH** | `kotah81/OrangeFox_device_xiaomi_venus` | `tiramisu` |
| **FOX-KOTAH-C** | `kotah81/OrangeFox_device_xiaomi_sm8350-common` | `tiramisu` |
| **FOX-VAYU** | `OrangeFox/device/vayu` | `fox_14.1` |
| **FOX-SYNC** | `gitlab.com/OrangeFox/sync` | `master` |
| **FOX-VENDOR** | `gitlab.com/OrangeFox/vendor/recovery` | `main` |
| **A14-MK** | `nebrassy/android_build` | `android-14`（fox_14.1 使用的 `build/make` 分支） |

## 1. 文件清单

| 本仓库文件 | 来源 | 备注 |
| --- | --- | --- |
| `BoardConfig.mk` | **新写** | 分区大小/cmdline/架构取自 LOS-common；recovery-as-boot 与 `TW_*` 取自 TW-common 与 FOX-KOTAH-C；AVB 与 `BUILD_BROKEN_*` 取自 FOX-VAYU + FOX-KOTAH-C；预编译内核段落参照 FOX-VAYU |
| `device.mk` | **新写** | `TARGET_RECOVERY_DEVICE_MODULES` / `RECOVERY_LIBRARY_SOURCE_FILES` / `TW_LOAD_VENDOR_MODULES` 取自 FOX-KOTAH-C；触摸固件拷贝取自 FOX-KOTAH；crypto 段落来自 TW-common/FOX-KOTAH-C（**v4 起已启用**） |
| `twrp_mars.mk` | **新写**，结构参照 **FOX-VAYU** 的 `twrp_vayu.mk` | `PRODUCT_NAME := twrp_mars`，继承 `base.mk` + `core_64_bit.mk` + `vendor/twrp/config/common.mk` + `device.mk` + `fox_mars.mk` |
| `fox_mars.mk` | **新写**，选项集参照 **FOX-VAYU** 的 `fox_vayu.mk` | 屏幕几何按 mars 的 1440×3200 调整（见 U4 与状态说明） |
| `AndroidProducts.mk` | **新写**，格式参照 **FOX-VAYU** + **FOX-KOTAH** | `COMMON_LUNCH_CHOICES := twrp_mars-ap2a-eng twrp_mars-ap2a-userdebug` —— 三段式，见 U11 |
| `Android.mk` | **TW**（裁剪） | 2021 版用 `all-subdir-makefiles`；已去掉，避免 `scripts/`、`docs/` 被解析 |
| `vendorsetup.sh` | **FOX-KOTAH**（改写） | 与 FOX-VAYU 相同的 `fox_get_target_device` 结构，`FDEVICE=mars`，适配 1440×3200 |
| `board-info.txt` | **新写** | `require board=mars` |
| `manifest.xml` | **TW** | mars 的 HIDL manifest。**保留但未引用**（`ODM_MANIFEST_FILES` 未设置） |
| `modules.load.recovery` | **FOX-KOTAH** | 仅作参考清单：`BOARD_VENDOR_RAMDISK_RECOVERY_KERNEL_MODULES_LOAD` 未设置（内核是预编译的） |
| `firmware/st_fts_k2.ftb`、`st_fts_k2_htp.ftb` | **FOX-KOTAH** | FTS 触摸屏固件，见 U1 |
| `recovery/root/lib/firmware/st_fts_k2*.ftb` | 上面两个的副本 | 打进 recovery ramdisk 的 `/lib/firmware/`，让触摸不依赖 `TARGET_COPY_OUT_VENDOR_RAMDISK` |
| `recovery/root/system/etc/recovery.fstab` | **新写**，参照 **TW-common** + **FOX-KOTAH-C** + **LOS-common** | 见 `history/FSTAB-DIFF.md`；这就是 `TARGET_RECOVERY_FSTAB` |
| `recovery/root/system/etc/twrp.flags` | **新写**，参照 **FOX-KOTAH-C** + **TW-common** + **FOX-VAYU** | mars 适配：去掉 `/recovery` 与 MicroSD，加入 `/vendor_boot` |
| `recovery/root/ueventd.qcom.rc` | **TW** | 原样取自 `twrp/recovery/root/ueventd.qcom.rc` |
| `recovery/root/init.recovery.qcom.rc` | **TW-common** + **LOS-common** | 并集：TW-common 的 USB/`by-name` 段 + LOS-common 的 ADSP 固件挂载；v4 起另行加入解密相关 import 与默认属性 |
| `recovery/root/init.recovery.usb.rc` | **FOX-VAYU** | 与 vayu 逐字节相同（QTI configfs gadget，adb+MTP+sideload 都依赖它） |
| `recovery/root/sbin/mars-fw-mount.sh` | **新写**（v4.7–v4.9） | 探测 slot、挂载固件分区、诊断日志 |
| `recovery/root/system/bin/`、`vendor/lib64/` 下的 QSEE/keymaster/gatekeeper blob | **TheMuppets/proprietary_vendor_xiaomi**（`lineage-23.2`，`mars/`、`sm8350-common/`） | 与设备 ROM 同源；逐个文件的说明见 git 历史里的 `docs/history/V4-REPORT.md` |
| `scripts/fetch-kernel.sh` | **新写** | 风格参照内核仓库的 `ci/make-bootimg.sh` |
| `scripts/fox-sync.sh` | **新写** | 封装 `FOX-SYNC` 的 `orangefox_sync.sh --branch 14.1` |
| `prebuilt/Image` | **不提交** | 由 `fetch-kernel.sh` 从内核项目的 release 下载 |

## 2. 值得记录的设计决定

**D1 —— 自包含设备树，不依赖 `sm8350-common`。**
`device/xiaomi/mars` 不 `include device/xiaomi/sm8350-common/*`，recovery 需要的东西
全部内置，做法与 `FOX-VAYU` 一致（后者直接在 `fox_14.1` 上构建、没有公共树）。
好处：构建只需取一个仓库，也不存在把 Android 16 公共树混进 Android 14 构建系统的风险
（LOS-common 是 `lineage-23.2`，在这里不能用）。

**D2 —— `BOARD_USES_RECOVERY_AS_BOOT := true`，不用 vendor_boot recovery。**
mars 没有 recovery 分区，要求产出可刷的 `boot.img`。开启该开关后，构建系统的
`recoveryimage` 目标输出 `boot.img` 而不是 `recovery.img`
（`build/make/core/Makefile`：`ifeq ($(BOARD_USES_RECOVERY_AS_BOOT),true)` →
`$(INSTALLED_BOOTIMAGE_TARGET): $(recoveryimage-deps)`）。
`BOARD_MOVE_RECOVERY_RESOURCES_TO_VENDOR_BOOT` 显式为 `false`，避免 `OrangeFox_A14.sh`
把本设备误判成 vendor_boot-recovery 设备。

**D3 —— 不打包 dtb。**
`BOARD_INCLUDE_DTB_IN_BOOTIMG` 故意不设；一旦设置，`recoveryimage-deps` 会依赖
`$(INSTALLED_DTBIMAGE_TARGET)`（`dtb.img`），没有真实 `BOARD_PREBUILT_DTBIMAGE_DIR` 就会失败。

**D4 —— recovery-as-boot 路径下 OrangeFox 选中 `boot.img`。**
`OrangeFox_A14.sh` 的判定是
`if FOX_VENDOR_BOOT_RECOVERY=1 || BOARD_INCLUDE_RECOVERY_RAMDISK_IN_VENDOR_BOOT=true || BOARD_MOVE_RECOVERY_RESOURCES_TO_VENDOR_BOOT=true || -n "$INSTALLED_VENDOR_BOOTIMAGE_TARGET"`。
recovery-as-boot 的 Makefile 钩子（`ORANGEFOX_CALLING_CARD=3.0`）**不传**
`INSTALLED_VENDOR_BOOTIMAGE_TARGET`，D2 又让另外三项为假，于是
`COMPILED_IMAGE_FILE="boot.img"`。

**D5 —— crypto 现在默认开启（历史：phase 1 曾关闭）。**
早期为降低风险设过 `TW_INCLUDE_CRYPTO := false`；v4 起改为开启，并把 QCOM
keymaster / `qseecomd` / `rpmb` 等 blob 一并放进 `recovery/root/vendor/lib64/`
（v4.9 实测可解密）。当前开关见 `device.mk` 与 `docs/STATUS.md`。

**D6 —— 仓库根目录就是设备树。**
与 `repo`/LineageOS 的设备树布局一致，可以直接克隆到 `device/xiaomi/mars`。
构建脚本 rsync 时会排除 `.git/`、`.github/`、`scripts/`、`docs/`、`dist/`。

## 3. 不确定项（U）

| 编号 | 项 | 为什么不确定 | 如何确认（现状） |
| --- | --- | --- | --- |
| **U1** | `firmware/st_fts_k2*.ftb` 触摸固件 | 取自 **venus** 的 OrangeFox 树；mars 与 venus 同属 FTS 触摸家族但面板固件可能不同 | v4.9 实测触摸正常（`fts_touch_spi`/`xiaomi_touch` 已加载）；如遇异常，从 mars 真机 `adb pull /vendor/firmware/` 对比替换 |
| **U2** | 运行期内核模块 | 内核是预编译的 ⇒ `TW_LOAD_VENDOR_MODULES` 从**已安装 ROM 的** `/vendor/lib/modules` insmod；只有 vermagic 匹配才有效 | v4.9 实测 `lsmod` 里有 `xiaomi_touch`/`fts_touch_spi`/`msm_drm`/`qti_battery_charger_main`；长期方案见 `RAMDISK-MODULES.md` |
| **U3** | 亮度/帧率的取值 | 早期是从 FOX-VAYU（不同面板）照抄的 | 已定：`TW_MAX_BRIGHTNESS := 4095`（同平台值）、`TW_DEFAULT_BRIGHTNESS := 40`（`BoardConfig.mk`，0 = 熄屏）、`TW_FRAMERATE := 120`（只影响空闲帧预算与动画跳帧）。可用 `cat /sys/class/drm/*/modes` 核对真实面板率 |
| **U4** | GUI 几何（`OF_SCREEN_H` / `OF_STATUS_H` 等） | 早期无 mars 参照 | 已定：`OF_SCREEN_H := 2400`（虚拟画布 1080×2400，与面板宽高比一致）、`OF_STATUS_H := 137`、`OF_STATUS_INDENT_LEFT := 216`、`OF_HIDE_NOTCH := 1`、`OF_CLOCK_POS := 1` |
| **U5** | `TW_NO_HAPTICS := true` | phase 1 的简化：sm8350 的 AIDL vibrator HAL 不在本 manifest | 仍未做（见 STATUS 的 v5 可选清单） |
| **U6** | `twrp.flags` 里的 `/cust` 分区 | 取自 2021 年的 sm8350 flags，未确认每个 mars SKU 都有 | 设备上 `ls /dev/block/by-name/` 核对；缺失时 TWRP 只会显示为不可用，无害 |
| **U7** | AVB 设置（`--flags 3`、sha256_rsa2048 测试密钥） | 照抄 FOX-VAYU / LOS-common；设备已解锁，只影响产物里的 hash footer | 不阻塞；刷写报错再回看 |
| **U8** | 未设置 `AB_OTA_UPDATER` | 故意不设；设成 `true` 后 A14 要求 `AB_OTA_PARTITIONS`（`Makefile`：`$(error AB_OTA_PARTITIONS must be defined when using AB_OTA_UPDATER)`） | 仍是 v5 待办（见 STATUS 3.2） |
| **U9** | fox_14.1 同步后的磁盘占用 | `nebrassy` 的 twrp-14 manifest 接近完整 AOSP 14，另有 `remove-minimal.xml` 裁剪；裁剪后大小未精确测 | 已按 200 GB 预算；首次构建前先 `df -h` |
| **U10** | 两个从 **FOX-KOTAH** 抄来的选项在 fox_14.1 里不存在 | `OF_NO_HAPTICS` 与 `OF_IGNORE_LOGICAL_MOUNT_ERRORS` 在 OrangeFox 的 recovery/vendor 树里没有任何地方读取 | 已从 `fox_mars.mk` 移除；振动用真实开关 `TW_NO_HAPTICS := true`（`bootable/recovery/Android.mk` 里 `ifeq ($(TW_NO_HAPTICS), true)`）；`check-tree.sh` 会防止它们被加回来 |
| **U11** | lunch 里的 `<release>` 名 | fox_14.1 的 `lunch()` 要求三段式，而这棵树不发布 release 名；`ap2a` 由实测确定 | 已定：`twrp_mars-ap2a-eng`；`build-local.sh` 会自动探测 |

## 4. 为什么 fox_14.1 的同步和"工单里的假设"不一样

```
$ git ls-remote --heads https://gitlab.com/OrangeFox/sync.git
53a303ecfb622c516082d3e61dbaa7d9f02f0120  refs/heads/master
```

只有 `master`。`fox_14.1` 是**legacy** 分支，由 `orangefox_sync.sh`（`do_fox_141()`）处理，
`--branch 14.1` 时会设置：

```
MIN_MANIFEST="https://github.com/nebrassy/platform_manifest_twrp_aosp.git"
FOX_BRANCH="fox_14.1"   TWRP_BRANCH="twrp-14"
DEVICE_BRANCH="android-14"   TW_DEVICE_BRANCH="android-14.1"
```

然后按顺序执行：

1. `repo init --depth=1 -u $MIN_MANIFEST -b twrp-14`
2. `repo sync --force-sync -c -j$(nproc --all) --no-clone-bundle --no-tags`
3. 给 `build/make`（`patch-manifest-fox_14.1.diff`）、`system/vold`、
   `.repo/manifests/remove-minimal.xml`、`system/update_engine`、`vendor/twrp` 打补丁
4. 克隆 `device/qcom/common`（TW `android-14.1`）、`device/qcom/twrp-common`（TW `android-14`）
5. 克隆 `external/se_omapi`（OrangeFox `fox_14.1`）
6. 用 `OrangeFox/bootable/Recovery`（`fox_14.1`）替换 `bootable/recovery`
7. 用 `OrangeFox/vendor/recovery`（`main`）替换 `vendor/recovery`

任一步失败即中止，且**不会**跑测试构建（`WorkNow()` 里的 `test_build` 被注释掉了）。

`fox_16.0` 不同：它用完整的 OrangeFox manifest
（`gitlab.com/OrangeFox/Manifest.git -b fox_16.0`）走普通 `repo init`/`repo sync`，
不再做 legacy 打补丁。以后若切到 16.0，把 `scripts/fox-sync.sh` 换成那两条命令即可。
