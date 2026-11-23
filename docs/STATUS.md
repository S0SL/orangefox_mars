# STATUS — 现状、修复史、待办（接手文档）

> 最后更新：v4.9 实测通过后整理。
> 旧的阶段报告已移入 [docs/history/](history/)，构建排障编年史在
> [BUILD-STATUS.md](BUILD-STATUS.md)，构建步骤在 [BUILD-ON-SERVER.md](BUILD-ON-SERVER.md)。

---

## 1. 当前状态

**v4.9 = 可用版本。** 产物（GitHub `recovery-test` pre-release）：

| 文件 | 大小 | sha256 |
| --- | --- | --- |
| `boot-v4.9.img` | 201 326 592 B | `0567218e977f45c51267291a04c20557c12f594cec6d943db5753dc8d127b4b1` |
| `boot-v4.9.img.gz` | 52 949 394 B | `946075bb30cb0e7c2dc1c34d5dd3041ee34a6ea6e2ed0d92e728a49b1e98fd00` |

已实测通过的能力：

| 能力 | 证据（v4.9 实测日志） |
| --- | --- |
| **/data 解密**（FBE + metadata 加密 + 硬件包裹密钥） | `GateKeeper status ok` → `User 0 Decrypted Successfully!` → `Data successfully decrypted`；`/data` = f2fs rw、`/sdcard` 可访问、217 GB 可用 |
| **ADB 与 MTP 共存**、sideload | `init.recovery.usb.rc` 生效；`init.svc.adbd=running`；`sys.usb.controller=a600000.dwc3` |
| **电量显示** | `power_supply` 有 `battery/usb/wireless`，`capacity=17`（health HAL 工作） |
| **默认中文 / 黑色主题 / 最低亮度** | `tw_language=zh_CN`；`/twres/themes/style.xml` 由 `styles/Black.xml` 复制；`TW_DEFAULT_BRIGHTNESS=40`（编译进 `board ninja`，各 21 处） |
| **刷 zip、分区备份/恢复** | `/tmp/recovery.log`：`Process ended with no errors`；分区表 `Can_Be_Backed_Up` 等标志正常 |
| **解密后服务健康** | `init_svc`：0 个 restarting；`keystore2`、`health-hal-2-1`、`adbd`、`hwservicemanager` 均 running |
| **触摸/显示/充电模块** | `lsmod`：`fts_touch_spi`、`xiaomi_touch`、`msm_drm`、`qti_battery_charger_main`、`adsp_loader_dlkm` 等 |

测试方式见 README 的 “Try it without flashing”；**不要**在没有备份的情况下
`fastboot flash boot`（那会让手机每次开机都进 recovery，见 §4）。

---

## 2. 已完成的修复（按"现象 → 根因 → 修法"）

每一层都对应一次实测失败；详细日志与推导见对应文档（`docs/history/` 为阶段报告，
`BUILD-STATUS.md` 为排障编年史）。提交信息里带 `v4.x:` 前缀，可用
`git log --oneline --grep 'v4.9'` 定位。

| # | 现象 | 根因 | 修法（落点） |
| --- | --- | --- | --- |
| 1 | 刷入后**黑屏** | 内核与安装的 ROM 内核 vermagic 不匹配；且 ramdisk 用 LZMA，而该内核只带 `CONFIG_RD_GZIP`/`LZ4` | 用 LOS 原厂 `Image`（`scripts/fetch-kernel.sh`）+ `BOARD_RAMDISK_USE_LZMA := false`（`BoardConfig.mk`） |
| 2 | recovery 里**完全没有 USB**（无 adb/MTP，sideload 卡死） | 本构建系统走 RBC 的 board-config 路径，把 `BOARD_KERNEL_CMDLINE` 从 boot header 里剥掉 → `sys.usb.controller` 为空；gadget 又缺 `mtp.gs0`/`os_desc` | ① `BOARD_MKBOOTIMG_ARGS += --cmdline "$(BOARD_KERNEL_CMDLINE) twrpfastboot=1"`；② `setprop sys.usb.controller a600000.dwc3` 兜底；③ 新增 `recovery/root/init.recovery.usb.rc`（QTI configfs gadget，与 vayu 逐字节相同）；核对：`cat /proc/cmdline`、`getprop sys.usb.config` |
| 3 | 解密相关服务**根本不存在** | `qcom_decrypt` 的 post-install 在本构建系统失效，没人 `import /init.recovery.qcom_decrypt.rc` | `recovery/root/init.recovery.qcom.rc` 里写死 `import`（`qseecomd`/`keymaster-4-1-qti`/`gatekeeper-1-0-qti` 才存在） |
| 4 | 状态栏时钟落进**挖孔**、UI 比例不对 | 主题是 1080 宽的 1920 基线，y 缩放用 `(fb_h+TW_H_OFFSET)/OF_SCREEN_H`，`OF_SCREEN_H=3200` 让横竖比例差 37%；`OF_STATUS_INDENT_LEFT=48` < 挖孔宽 208 | `OF_SCREEN_H := 2400`、`OF_STATUS_H := 137`（ROM 的 `status_bar_height_portrait`）、`OF_STATUS_INDENT_LEFT := 216`、`OF_HIDE_NOTCH := 1`、`OF_CLOCK_POS := 1`（`fox_mars.mk`） |
| 5 | **logd 崩溃**（因此早期看不到 logcat） | `system/logging/logd/main.cpp:77` 的 `set_sched_policy(0, SP_BACKGROUND)` 需要 `/etc/task_profiles.json`，ramdisk 里没有 | `device.mk` 的 `PRODUCT_COPY_FILES` 补 `task_profiles.json` + `init.recovery.qcom.rc` 建 `cpu/blkio` 的 `background`/`foreground` cgroup 目录 |
| 6 | keystore2 拿不到 KeyMint；厂商 HAL 注册被拒 | `libhidlbase` 的 VINTF 强制检查：接口未在 manifest 声明就 `registerAsService` 失败 | `recovery/root/vendor/etc/vintf/manifest.xml`：`android.hardware.keymaster@4.0/@4.1::IKeymasterDevice`、`@1.0::IGatekeeper`、`@2.0/@2.1::IHealth` |
| 7 | keystore2 日志 `Skipping keymaster compat, this system is AIDL only` | 该消息的真实含义是"连不上 hwservicemanager"：缺 `android.hidl.manager@1.2::IServiceManager/default`（框架 manifest 缺失时 `getTransport` 回退到 device manifest） | 同一 manifest 里声明 `android.hidl.manager@1.2` |
| 8 | keystore2 注册 AIDL 服务失败（`UNKNOWN_ERROR`） | servicemanager 要求 VINTF 稳定 AIDL 服务先声明；而 ramdisk 缺 `/system/etc/vintf/manifest.xml` 基文件 → fragment 从未被合并 | 新增 framework 基文件（v4.4）并在 device manifest 也写一份 AIDL `android.system.keystore2.IKeystoreService/default`（v4.9 起基文件可省） |
| 9 | gatekeeper 校验失败 `verify_legacy` → `ret: -45`，密码进不去 | ① 随机 challenge 补丁打在了**不会执行**的 legacy 分支；② synthetic-password 槽位优先找 `/data/system_de/0/spblob`；③ **真正原因**：ROM 的 `vendor.prop` 有 `vendor.gatekeeper.disable_spu=true`，recovery 没有 → HAL 去初始化 SPU（recovery 里不可用）→ 退回 legacy 实现 → 该 TA 不支持 legacy verify | ① `apply-patches.sh` 在**真正执行**的两处 `verify()` 传随机 challenge、spblob 优先 `/metadata/password_slots`；② `setprop vendor.gatekeeper.disable_spu true`（`init.recovery.qcom.rc` 的 `on init`） |
| 10 | `/firmware` 与 `/vendor/firmware_mnt` 都是空目录（SPU/PMIC 固件取不到） | 旧 rc 用 `by-name/modem${ro.boot.slot_suffix}`，而 `fastboot boot` 下该属性为空 → 节点不存在（真机是 `modem_a`）；vfat 也不允许二次挂载 | 新增 `recovery/root/sbin/mars-fw-mount.sh`（oneshot 服务，`on fs` 启动）：`getprop` → cmdline → `_a/_b` 探测 slot、列节点、vfat/ext4 双尝试、`/vendor/firmware_mnt` 用**可写目录 + 软链**、全过程写 `/tmp/mars-fw-mount.log` |
| 11 | ramdisk 脚本 `bad substitution` 直接退出 | 脚本里用了 bash 风格的 `${ro.boot.slot_suffix}`，recovery 是 Android **mksh** | 全部改 `getprop`；并新增 `scripts/check-shell.sh`（`sh -n` + 点号属性展开检查），接入 `build-local.sh` 的 `== 0. shell syntax lint ==` 守卫 |
| 12 | 默认值不生效 / 构建不可复现 | 设置无法持久化（`/data/recovery/Fox` 不可用）；AOSP 侧补丁只存在于构建机 | 默认值：`TW_DEFAULT_BRIGHTNESS := 40`（**必须在 `BoardConfig.mk`**，它在产品配置之后解析）、`TW_DEFAULT_LANGUAGE := zh_CN` + 自带 `twres/languages/zh_CN.xml`、`on init` 里 `copy styles/Black.xml → style.xml`；可复现：`scripts/apply-patches.sh`（幂等）+ `build-local.sh` 的 `== 0b. build-tree patches ==` 守卫 |

---

## 3. 待办 / 下一步（v5）

> **v5 = 一次构建** 即可包含已就绪的第 1 项，其余按优先级排。

### 3.1 GUI 拖拽节奏（补丁已就绪，只需出包）

* **为什么**：`gui/gui.cpp` 的 `loopTimer()` 用
  `long timeout = got_event ? 500000000 : (1.0 / TW_FRAMERATE * 1000000000);`，
  而拖拽只在预算到期时由 `handleDrag()` 下发 → **手指在滑动时预算恒为 500 ms**，
  拖拽 ≈ 2 次/秒（TWRP 上游同样如此）。表现：图案/滑块/拖动"不灵敏"，点击不受影响。
* **改哪里**：`scripts/apply-patches.sh` 已把该行改为始终使用帧预算（幂等），
  v5 构建会自动应用；几何值（`OF_SCREEN_H` 等）**不要**因此再动。
* **怎么验证**：解密页输入 PIN、拖动亮度滑块、滚动列表，应 1:1 跟手；
  `TW_FRAMERATE` 只影响空闲帧预算/动画跳帧，可用 `cat /sys/class/drm/*/modes`
  确认真实面板率后决定是否设成 60。

### 3.2 `AB_OTA_UPDATER` + boot HAL（解锁刷 ROM / vendor 类 zip）

* **为什么**：现在没有 `AB_OTA_UPDATER`，`Prepare_Super_Volume()` 用不带后缀的
  分区名去找 dm 设备，而设备映射是 `system_a`/`product_a` → 日志
  `unable to update logical partition`，改写 logical partition 的 zip 刷不了；
  同时 `android.hardware.boot@1.0::IBootControl` 缺失（无 slot 控制）。
* **改哪里**：`BoardConfig.mk` 里 `AB_OTA_UPDATER := true` + `AB_OTA_PARTITIONS`
  （必须在 OrangeFox 的赋值之前，使其成为 no-op）；`device.mk` 增加
  `update_engine_sideload` 与 `android.hardware.boot@1.0/1.1/1.2` 服务及对应 blob
  （缺件会让构建直接失败，这也是当初搁置的原因）。
* **怎么验证**：`/system/bin/update_engine_sideload` 存在；TWRP 刷 ROM zip 无
  `unable to update logical partition`；`bootctl get-slot` 可读。

### 3.3 keystore2 GC 噪音

* **为什么**：解密后 `keymaster-4-1-qti` 已 stopped，而 keystore2 仍在跑 GC，
  每秒多次 `deleteKey transaction failed: 'DEAD_OBJECT'` + `gc.rs:139 Error trying to
  delete blob entry`（会话内无实际影响，`keystore.crash_count=0`）。
* **改哪里**：让 keymaster 服务常驻（去掉 oneshot 或在 GC 前停止），或接受噪音。
* **怎么验证**：`logcat | grep -c keystore2.*DEAD_OBJECT` 应为 0。

### 3.4 vendor 模块目录

* **为什么**：`/tmp/recovery.log` 有 `Unable to open module directory: /vendor/lib/modules.
  Skipping`（recovery 没挂真实 vendor 分区）；目前依赖 vendor_boot ramdisk 里的模块。
* **改哪里**：把 vendor 分区只读挂到 `/mnt/vendor`，并只对 `/vendor/lib/modules` 做软链
  ——**切勿**整体覆盖 `/vendor`，那里放着本项目的 TEE blob。
* **怎么验证**：`lsmod` 含 ROM 的完整模块集；日志不再出现该行。

### 3.5 device manifest 的 `target-level`

* **为什么**：`hwservicemanager: Cannot filter framework manifest HALs properly because
  target FCM version is unspecified in the device manifest`。
* **改哪里**：`recovery/root/vendor/etc/vintf/manifest.xml` 的 `<manifest>` 加
  `target-level`（按该 vendor 的 FCM 级别填，需先核对，填错可能拒 HAL）。
* **怎么验证**：该 warn 消失，且 keymaster/gatekeeper 仍能注册。

### 3.6 可选

* **SELinux enforcing**：现在 permissive。若要做，需要补 `recovery` 访问
  `/data`（`vold_data_file`/`system_data_root_file`/`app_data_file`/`adb_data_file`/
  `property_data_file`/`fingerprint_vendor_data_file`）、`ueventd → firmware_file`、
  `health → netlink_kobject_uevent_socket` 等规则（固件挂载的 `context=` 已在 v5 修复里加好）。
* **haptics**：`TW_NO_HAPTICS := true`（sm8350 AIDL vibrator HAL 不在 manifest 里）。
* **`ipa_fws.mdt` PIL 失败**：与 recovery 无关的固件缺失（IPA 卸载），可不处理。

---

## 4. 已知限制

1. **没有 recovery 分区**：`fastboot flash boot` 后每次开机都进 recovery（recovery-as-boot）。
   回 Android 需重刷 ROM 的 `boot.img`。测试请用 `fastboot boot`（不写分区）。
2. **/data 解密的依赖**：手机 TEE（keymaster/gatekeeper TA）、`/metadata` 完好、
   与 vendor 同源的 QSEE blob，以及 `vendor.gatekeeper.disable_spu=true`
   （ROM 自带该属性，recovery 现在也设）。换 ROM/换 vendor 需要重做 blob 来源与 VINTF 声明。
3. **SELinux permissive**：enforcing 需补 sepolicy（见 3.6）。
4. **AB_OTA 关闭**：改写 logical partition 的 zip / ROM 不支持（见 3.2）；
   `system` 文件级备份也因 ROM 的 `Can_Be_Backed_Up=false` 被禁用。
5. **无 haptics**：`TW_NO_HAPTICS := true`。
6. **触摸固件来源**：`firmware/st_fts_k2*.ftb` 来自 OrangeFox venus 树（v4.9 实测触摸正常，
   但如遇异常可从 mars 真机 `/vendor/firmware/` 替换）。
7. **签名**：AVB 测试密钥，仅限已解锁 bootloader。
8. **构建机**：需要一台真实服务器（AOSP 树放不进容器），见 `BUILD-ON-SERVER.md`。

---

## 5. 接手须知

### 5.1 构建

```sh
git clone https://github.com/S0SL/orangefox_mars.git ~/orangefox-mars
~/orangefox-mars/scripts/build-local.sh --fox-dir ~/fox_14.1              # 首次（sync + 内核 + 编译）
~/orangefox-mars/scripts/build-local.sh --fox-dir ~/fox_14.1 --skip-sync  # 之后
```

* `MARS_JOBS=12`（24 核 / 22 GB 机型上比 16 更快；`mka` 不认 `MAKE_FLAGS`，脚本会显式传 `-j`）。
* `--skip-sync` 要求 `~/fox_14.1` 已经完整同步过（`~/.mars-sync-complete` 存在），**不要**重跑 sync。
* 构建前两个守卫会自动跑：`== 0. shell syntax lint ==`（`check-shell.sh`）与
  `== 0b. build-tree patches ==`（`apply-patches.sh`）；设备树静态检查用 `scripts/check-tree.sh`。
* 产物：`out/target/product/mars/boot.img`（job = `bootimage`）。

### 5.2 发版

* 构建在本机，**产物通过 GitHub Release 分发**（不要 scp 大文件）；
  `scripts/publish-release.sh` 面向 CI 产物，手工发版用 `gh release upload` 或 API。
* 测试永远先 `fastboot boot boot.img`（不写分区），确认能进 UI/解密后再考虑 flash。

### 5.3 排障入口

| 想知道 | 看 |
| --- | --- |
| 某个历史失败是怎么定位的 | `docs/BUILD-STATUS.md`（编年史，按现象索引） |
| 某个 blob 从哪来 | `docs/PROVENANCE.md` |
| 构建机/依赖/计时 | `docs/BUILD-ON-SERVER.md` |
| 备份与救砖 | `docs/BACKUP-GUIDE.md` |
| 阶段报告（v4 / v4.6 / phase2 / fstab 对比） | `docs/history/` |
| 内核侧（触摸模块）准备但未推送 | `docs/kernel-prep/` |

**观察点时最有用的几条命令**（recovery 内 adb）：

```sh
getprop init.svc.keystore2; getprop init.svc.logd; getprop vendor.gatekeeper.disable_spu
cat /tmp/recovery.log            # TWRP 自己的决策（解密、备份、刷 zip）
cat /tmp/mars-fw-mount.log       # 固件分区挂载与 SPU 固件名
logcat -d | grep -iE "keystore|keymint|gatekeeper|vintf|SPU"
```
