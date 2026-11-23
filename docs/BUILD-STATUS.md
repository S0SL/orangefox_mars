# 排障编年史（BUILD-STATUS）

本文件按时间顺序记录每一次失败：**当时的现象 → 定位过程 → 根因 → 修法**。
目的不是"报告进度"，而是让下次遇到相似现象时能快速对照。

> **当前结论不在这里。** 现状、修复后的完整清单、待办与限制见 [STATUS.md](STATUS.md)；
> 构建步骤见 [BUILD-ON-SERVER.md](BUILD-ON-SERVER.md)。
> 云端 CI 时代（`.github/workflows/build-recovery.yml`）的完整英文原文在 git 历史里
> （该工作流后来已删除，构建全部改为本地服务器）。

---

## 云端 CI 阶段

### 第 1 轮 —— 构建失败，定位并修复两处

运行：`37761738123`。

* **打通的部分**：修正后的同步路线被确认可用 —— `scripts/fox-sync.sh` →
  `orangefox_sync.sh --branch 14.1` 同步完整棵树，并成功给 `build/make`、`system/vold`、
  `remove-minimal.xml`、`system/update_engine` 打上补丁。
* **失败 1**（直接原因）：
  ```
  build/envsetup.sh: line 21: TOP: unbound variable
  ```
  在 `set -u` 下 source AOSP 的 `envsetup.sh` 会炸。修法：source 前 `set +u`，之后恢复。
* **失败 2**：上游 `orangefox_sync.sh` 到错误目录找 `vendor/twrp` 补丁，且只告警：
  ```
  ./orangefox_sync.sh: line 248: .../sync/patch-vendor-twrp-fox_14.1.diff: No such file or directory
  -- Error! Failed to patch the twrp-14 vendor/twrp !
  ```
  修法：`fox-sync.sh` 预先把补丁放到正确位置，并**校验** `vendor/twrp/config/BoardConfigSoong.mk`
  确实包含 `orangefox_soong.mk`。

### 第 2、3、4 轮

| 轮次 | 结论 |
| --- | --- |
| 2 | 故意取消（带了前两项修复，但缺 display 库守卫；继续跑也过不去） |
| 3 | 同上，取消 |
| 4 | 走到第 12 步；apt 依赖拆成三层后从"卡 10 分钟"降到 **约 50 秒** |

* **失败 3**：`lunch` 要求**三段式**，`twrp_mars-eng` 永远不合法：
  ```
  including device/xiaomi/mars/vendorsetup.sh
  Invalid lunch combo: twrp_mars-eng
  ```
  修法：`twrp_mars-ap2a-eng`（三个阶段：`<product>-<release>-<variant>`）。

### 第 5 轮 —— 又发现两个原因

运行 `37769569262`。前 12 步全部通过（同步约 19 分钟），编译 30 秒即失败。

* **原因 4**：release token 是 `ap2a`，不是 `bp2a`。
* **原因 5**：探测时把每次尝试的输出丢进 `/dev/null`，导致有效 release 的真实报错被丢弃。
  改为捕获并打印每次失败尝试的尾部日志。

### 第 6 轮 —— 卡在 `lunch`，原因 6

运行 `37773718753`。

* **原因 6**：本构建系统对 `KATI_obsolete_var` 的**任何**赋值都是硬错误
  （`build/make/core/config.mk:174` 的 `$(KATI_obsolete_var BOARD_BUILD_SYSTEM_ROOT_IMAGE)`）。
  修法：清掉过时变量，并把 148 个名字整理成 `scripts/obsolete-build-vars.txt`，
  由 `check-tree.sh` 守住。

### 第 6/7 轮无法轮询时的离线工作（12:17 UTC 起）

构建容器完全失去出网能力，CI 无法轮询；期间只在本地准备了内核侧与模块侧的方案
（不改变 CI 行为，见 `docs/kernel-prep/`、`docs/RAMDISK-MODULES.md`）。

### GitHub 第 9 轮（`37780545560`）—— 构建开始 15 秒后被 runner 杀掉

第 7 次构建尝试，**第一次 `lunch` 成功**；随后因 runner 资源限制被终止。
这一轮之后彻底放弃云端 runner，转向自有服务器。

---

## 自有服务器阶段（`scripts/build-local.sh`）

**没有刷过任何设备，没有改过内核仓库，没有触发云端构建。**

### 服务器构建 1 —— 反复失败，逐个定位

**环境实测**：Ubuntu 22.04.5、内核 6.8.0-40、x86_64。

**阶段耗时**：apt 依赖（三层）**9 分 33 秒**（第一层 7 分 00 秒、第二层 i386 6 分 33 秒、
第三层 repo 启动器约 1 秒）。

* **故障 S1 —— 国际链路整段死掉约 15 分钟**：对所有 `github.com`、
  `android.googlesource.com`、`1.1.1.1` 的连接都能完成 TCP 握手，但传输 **0 字节**；
  而 `raw.githubusercontent.com`、`gitlab.com` 和国内镜像仍可用。
  处理：入口脚本对吃网络的部分做循环重试。
* **故障 S2 —— `repo` 静默丢掉 `--depth=1`**，把 1 GB 的拉取变成 20 GB：
  ```
  error: Unable to fully sync the tree
  ```
  根因：`repo` 只对"新"项目用 `--depth=1`；目录已存在的项目会拉**完整历史**。
  修法：入口脚本检测"只有失败 `tmp_pack_*` 的项目目录"→ 删除 → 重试恢复浅克隆；
  并在两次尝试间清理残留 `tmp_pack_*`（某次清出 54 GB）。
* **故障 S3 —— 自伤**：低速守卫在镜像还在打包时就触发
  （`curl 28 Operation too slow`）。修法：守卫只在真正拉取阶段生效，并先做心跳判断。
* **随后服务器不可达**：SSH 端点停止响应；探测显示 TCP 可连但**不返回 banner**、
  几秒后被对端干净关闭 —— 典型的"隧道后端掉线"。
* **故障 S4 —— `lunch` 静默死亡**：构建第一次走到 `mka` 就停住，且没有错误行。
  修法：在 `lunch` 前后加显式状态检查与日志落盘。
* **S5 —— 误报"内存守卫把构建锁死"**：守卫暂停时只打印一行，而 Soong 分析阶段
  长时间只显示 `[99% ...] cp .../soong_build`。修法：守卫补心跳输出，避免误判。
* **内存**：`soong_build` 峰值约 15.0 GB RSS（机器 15.6 GB），原 2 GB swap 100% 使用、
  约 120 MB/s 交换流量 → 增加 8 GB `/swapfile-mars`（`fallocate` + `mkswap` + `swapon`，
  不写 `/etc/fstab`）。
* **S6 —— 设备树自己设了 OrangeFox 已废弃的变量 `OF_AB_DEVICE`**，
  kati 阶段直接失败。修法：移除。
* **S7 —— 改名成 `FOX_AB_DEVICE` 后撞上 `AB_OTA_UPDATER` 只读**：
  ```
  bootable/recovery/orangefox.mk:170: error: cannot assign to readonly variable: AB_OTA_UPDATER
  ```
  这也是 `AB_OTA_UPDATER` 至今未开启的历史原因（见 STATUS 的 v5 待办）。

### v2 —— 构建并交付；黑屏根因定位并修复

v1（`fastboot boot`）能进 recovery（adb/`twrpfastboot` 起来了）但**屏幕不亮**。

* **原因 A（主因）**：预编译内核永远无法接受 ROM 的模块 ——
  `CONFIG_LOCALVERSION_AUTO=y` 把构建 commit 编进 `vermagic`
  （ROM：`5.4.302-qgki-g7ede20c8692e`；我们的 BakaSU：`5.4.302-qgki-g797c093f5b52`），
  且 `CONFIG_MODVERSIONS=y`、`# CONFIG_MODULE_FORCE_LOAD is not set`。
  v2 改用 LineageOS 原厂 `Image`。
* **原因 B（次因）**：v1 的 ramdisk 缺 `libdisplayconfig.qti.so` 等三个 CAF 显示库
  （且 `vendor/lib64` 为空）→ v2 从用户自己的 mars 真机拉取后打入 ramdisk。
* 交付镜像的实测：
  ```
  $ magiskboot unpack boot-v2.img
  KERNEL_SZ [39428608]  RAMDISK_SZ [18814764]  RAMDISK_FMT [lzma]  CMDLINE [twrpfastboot=1]
  ```
  —— 这一行同时暴露了下一个问题（LZMA + cmdline 只有 `twrpfastboot=1`）。

### v3 —— 黑屏真正原因是 LZMA ramdisk

**一句话根因**：recovery ramdisk 用 LZMA-alone 压缩，而 mars 的内核
`# CONFIG_RD_LZMA is not set`，内核根本无法解包 initramfs：`/init` 从未执行，
于是没有 `adbd`，于是"黑屏 + 无 adb"。

**证据链**：v1 与 v2 的 `Image` 不同、ramdisk 相同，两者都黑屏；
换回原厂 `Image` 后仍黑屏 → 变量只剩 ramdisk。

**修法**：`fox_mars.mk` 不再设置 `OF_USE_LZMA_COMPRESSION`，并固定
`BOARD_RAMDISK_USE_LZMA := false`；`check-tree.sh` 会在约 10 秒内发现回归。

**内核来源告警**：`dist/BUILD-INFO.txt` 过去打印的是 `$KERNEL_URL`（"将会下载"的 URL），
v2 的日志因此显示 BakaSU 而实际装的是原厂 `Image`。现已改为打印**实际**内核的
版本串与来源。

### v4 —— FBE 解密开启 + recovery USB 修复 + 挖孔/几何校准

```
v3  boot.img  sha256 d4eb1c6f092f762d0832b6159be119527ba8514292b35afd7de20d528c135584  (192 MiB，能启动，无 /data)
v4  boot.img  sha256 40d74db0ff5ee5efbfe5e84151738f012dec74c2cf2137c046df570de0853657
```

这一轮的细节（USB gadget 缺 MTP、cmdline 被 RBC 剥掉、解密 rc 未被 import、
logd 缺 `task_profiles.json`、厂商 HAL 缺 VINTF 声明、keystore2 的
`android.hidl.manager@1.2` 与 AIDL 注册、gatekeeper `-45` 与
`vendor.gatekeeper.disable_spu=true`、固件分区挂载、`getprop` 与 shell lint、默认值）
已按"现象→根因→修法"整理进 **[STATUS.md](STATUS.md) §2**，不再在这里重复。

### v4.9 —— 实测可用

`/data` 解密成功（`Data successfully decrypted`）、ADB 与 MTP 共存、电量正常、
中文/黑主题/最低亮度内置、刷 zip 与分区备份可用。完整证据见
[STATUS.md](STATUS.md) §1。
