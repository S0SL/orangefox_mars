# 小米 11 Pro（`mars`）OrangeFox recovery 设备树

本仓库用于构建 **OrangeFox Recovery**（基础：`fox_14.1`，即 AOSP 14 + OrangeFox R12），
目标设备为**小米 11 Pro**，代号 **`mars`**，SoC **高通 sm8350（骁龙 888）**，
**A/B 分区**、**没有独立 recovery 分区**。

recovery 被构建进 **`boot.img`**（`BOARD_USES_RECOVERY_AS_BOOT`）。本仓库里的任何脚本
**都不会自行向手机写入任何东西**。

> **当前状态：v4.9，可用。**
> 输入锁屏 PIN 后可以解密 `/data`（FBE + 硬件包裹密钥 + metadata 加密），ADB 与 MTP 同时可用，
> 电量/语言/主题默认值已内置，刷 zip 与分区备份/恢复均正常。
> 完整状态、修复史与后续工作见 **[docs/STATUS.md](docs/STATUS.md)**（接手文档）。
> 最近一次实测镜像：`recovery-test` 预发布里的 `boot-v4.9`
> （`sha256 0567218e977f45c51267291a04c20557c12f594cec6d943db5753dc8d127b4b1`），
> 稳定版发布见
> [recovery-v4.9-stable](https://github.com/S0SL/twrp_mars/releases/tag/recovery-v4.9-stable)。

---

## 1. 先试，不写分区

```sh
curl -L -O https://github.com/S0SL/twrp_mars/releases/download/recovery-test/boot-v4.9.img.gz
gunzip boot-v4.9.img.gz
fastboot boot boot-v4.9.img          # 载入内存运行；不写入任何分区
```

`fastboot flash boot boot-v4.9.img` 会让手机**每次开机都进入 recovery** —— 这是"没有独立
recovery 分区"的必然结果（见 [已知限制](docs/STATUS.md)）。要回 Android，请重刷 ROM 自带的
`boot.img`。

## 2. 构建

需要一台真实的构建服务器（AOSP 源码树放不进容器）：

* **完整步骤：** [docs/BUILD-ON-SERVER.md](docs/BUILD-ON-SERVER.md)
* 已经同步过源码树的机器上，简版：

  ```sh
  git clone https://github.com/S0SL/twrp_mars.git ~/twrp-mars
  ~/twrp-mars/scripts/build-local.sh --fox-dir ~/fox_14.1            # 首次
  ~/twrp-mars/scripts/build-local.sh --fox-dir ~/fox_14.1 --skip-sync  # 之后
  ```

  构建目标是 `bootimage`，产物为 `out/target/product/mars/boot.img`。
  在 24 核 / 22 GB 的机器上 `MARS_JOBS=12` 最快（16 线程反而更慢）。

构建脚本在编译开始前会跑两道守卫：

| 守卫 | 脚本 | 防的是什么 |
| --- | --- | --- |
| `== 0. shell syntax lint ==` | `scripts/check-shell.sh` | 打进 ramdisk 的 shell 脚本里的 bash 专有写法（recovery 跑的是 Android **mksh**；v4.7 就是因为 `${ro.boot.slot_suffix}` 直接 `bad substitution` 退出） |
| `== 0b. build-tree patches ==` | `scripts/apply-patches.sh` | 重新应用本设备需要、但又不能放进设备树的 AOSP 侧补丁（FBE 凭据流程、GUI 拖拽节奏） |

设备树自身的静态检查：`./scripts/check-tree.sh`（必需文件、fstab/flags 语法、makefile 结构、
代号一致性）与 `./scripts/check-shell.sh`。

## 3. 目录结构

```
BoardConfig.mk device.mk fox_mars.mk twrp_mars.mk AndroidProducts.mk Android.mk
vendorsetup.sh board-info.txt manifest.xml modules.load.recovery
firmware/                       打进 ramdisk 的触摸固件
prebuilt/                       预编译内核 Image（不提交）
recovery/root/                  进入 recovery ramdisk 的文件：
  init.recovery.qcom.rc           挂载与属性默认值（见 docs/STATUS.md）
  init.recovery.usb.rc            QTI configfs gadget（adb + MTP + sideload）
  sbin/mars-fw-mount.sh           挂载固件分区供 TEE 使用
  system/etc/{recovery.fstab,twrp.flags}
  system/bin/, vendor/lib64/      QSEE/keymaster/gatekeeper 相关 blob（见 docs/PROVENANCE.md）
  twres/languages/zh_CN.xml       简体中文语言包
  vendor/etc/vintf/manifest.xml   厂商 HAL 需要的 VINTF 声明
scripts/                        同步 / 构建 / 打补丁 / 检查 / 发版脚本
patches/                        构建树补丁的说明（由 apply-patches.sh 应用）
docs/                           见下
```

## 4. 文档

| 文件 | 内容 |
| --- | --- |
| **[docs/STATUS.md](docs/STATUS.md)** | **接手文档** —— 当前能做什么、每一处修复的原因、剩余工作（v5）、已知限制 |
| [docs/BUILD-ON-SERVER.md](docs/BUILD-ON-SERVER.md) | 完整构建指南、耗时、本项目踩过的每一个坑 |
| [docs/BACKUP-GUIDE.md](docs/BACKUP-GUIDE.md) | 备份 `/data`（Android 内 root）与分区（recovery 内）的方法与恢复流程 |
| [docs/PROVENANCE.md](docs/PROVENANCE.md) | 每个 blob 的来源与不确定点 |
| [docs/RAMDISK-MODULES.md](docs/RAMDISK-MODULES.md) | 把内核模块（触摸/显示）打进 ramdisk 的方法 |
| [docs/BUILD-STATUS.md](docs/BUILD-STATUS.md) | 排障编年史（"为什么会有这些守卫"） |
| [docs/kernel-prep/](docs/kernel-prep/) | 已准备但未推送的内核侧改动（触摸模块） |
| [docs/history/](docs/history/) | 被取代的阶段报告与计划（v4、v4.6、phase 2、fstab 对比、phase 1 已知问题） |

## 5. 已知限制（简述）

* **没有 recovery 分区**：刷入 `boot.img` 后每次开机都进 recovery，而不是 Android。
* **`/data` 解密依赖**：手机 TEE，以及厂商属性 `vendor.gatekeeper.disable_spu=true`
  （recovery 已设置；stock ROM 自带 —— 原因见 [STATUS.md](docs/STATUS.md)）。
* **SELinux 为 permissive**：要转 enforcing 需补一套 sepolicy。
* **`AB_OTA_UPDATER` 未开启**：改写 logical partition（`system`/`vendor`/…）的 zip 暂不支持，
  见 STATUS.md 的 v5 清单。
* 镜像使用 **AVB 测试密钥**，仅适用于已解锁 bootloader。

上述每一条的原因与证据都在 **[docs/STATUS.md](docs/STATUS.md)** 里。
