# 内核仓库侧的准备（**未推送、未应用**）

本目录包含对 `S0SL/android_kernel_xiaomi_sm8350` 的**预备改动**：让内核构建顺带产出
recovery 启动触摸与屏幕所需的内核模块。**这里的东西一个都没有推送，内核仓库完全未被改动**
（只在本地检出上做过 dry-run，并已逐字节还原 —— `git status` 为空，
对补丁 `git apply --check` 通过）。

方案在 recovery 侧的另一半见 [../RAMDISK-MODULES.md](../RAMDISK-MODULES.md)。

## 为什么需要它

两条实测结论（都来自真实产物，详见 [../RAMDISK-MODULES.md](../RAMDISK-MODULES.md) §1.1）：

* **ROM 的 `/vendor/lib/modules` 永远加载不进我们预编译的 `Image`** ——
  `CONFIG_LOCALVERSION_AUTO=y` 把构建 commit 编进 `vermagic`
  （原厂 LineageOS 23.2 是 `5.4.302-qgki-g7ede20c8692e`，我们的 BakaSU `Image` 是
  `5.4.302-qgki-g797c093f5b52`），而且 `CONFIG_MODVERSIONS=y` 且
  `# CONFIG_MODULE_FORCE_LOAD is not set`；
* **当前内核构建根本不产出 mars 的触摸驱动**：`vendor/xiaomi_QGKI.config` 里
  `# CONFIG_TOUCHSCREEN_ST is not set` / `# CONFIG_TOUCHSCREEN_FTS is not set`，
  也从未启用 `CONFIG_TOUCHSCREEN_XIAOMI_TOUCHFEATURE`。

因此用 `fastboot boot boot.img` 测试时（第一张镜像就是这种测法），触摸/显示只能来自
**与这张 `Image` 同一次构建**产出的模块。

## 改动内容

| 文件 | 改动 |
| --- | --- |
| `arch/arm64/configs/vendor/mars_recovery_touch.config` | **新增**，两个符号：`CONFIG_TOUCHSCREEN_ST_FTS_V521_SPI=m`、`CONFIG_TOUCHSCREEN_XIAOMI_TOUCHFEATURE=m` |
| `ci/build-kernel.sh` | 合并新 fragment；`MAKE_TARGET` 默认由 `Image` 改为 `Image modules` |
| `ci/package-modules.sh` | **新增**：`modules_install` + 产出 `dist/kernel-modules-<kver>.tar.gz`，若缺 `fts_touch_spi.ko`/`xiaomi_touch.ko`/`msm_drm.ko` 直接硬失败 |
| `.github/workflows/build-kernel.yml` | 执行打包步骤，并把 `dist/kernel-modules-*.tar.gz` 与现有资产一起上传 |

### 实测副作用：内置内核部分为零

用 `ci/build-kernel.sh` 相同的合并链（基础 `vendor/lahaine-qgki_defconfig`，再叠加
`vendor/debugfs.config` + `vendor/xiaomi_QGKI.config`，最后加新 fragment，每步后
`olddefconfig`）在 `O=/tmp/kout` 下跑过 —— 内核源码树没有任何改动。加入 fragment 前后
`.config` 的**完整**差异是：

```diff
-# CONFIG_TOUCHSCREEN_ST_FTS_V521_SPI is not set
+CONFIG_TOUCHSCREEN_ST_FTS_V521_SPI=m
-# CONFIG_TOUCHSCREEN_XIAOMI_TOUCHFEATURE is not set
+CONFIG_TOUCHSCREEN_XIAOMI_TOUCHFEATURE=m
```

没有任何 `=y` 符号变化，也没有触发 `select`：两个 Kconfig 都是普通 `tristate`，
依赖 `I2C` / `INPUT_TOUCHSCREEN`（已满足）。这两个驱动目前缺失、由 ROM 提供；改成 `=m`
只是让**我们的**构建也额外产出两个 `.ko`，`Image` 的内置部分不受影响。

## 如何应用（需批准后）

```sh
git clone https://github.com/S0SL/android_kernel_xiaomi_sm8350
cd android_kernel_xiaomi_sm8350

# 方式 A —— 用补丁
git am /path/to/0001-mars-touch-modules.patch        # 或 git apply 后手动提交

# 方式 B —— 用脚本（幂等；--revert 可撤销）
/path/to/docs/kernel-prep/apply-kernel-prep.sh --target .
git diff --stat && git add -A && git commit
```

然后推送。现有的内核 CI（`.github/workflows/build-kernel.yml`）会构建 `Image modules`，
并把 `kernel-modules-<kver>.tar.gz` 与 AnyKernel3 zip、可 fastboot 刷入的 `boot.img`
一起上传。

## 如何验证这次构建

```sh
# 内核 CI 跑完后，在产物上：
tar tzf kernel-modules-*.tar.gz | grep -E 'fts_touch_spi\.ko|xiaomi_touch\.ko|msm_drm\.ko'
tar xzf kernel-modules-*.tar.gz
strings fts_touch_spi.ko | grep vermagic      # 必须与 Image 的版本串一致
strings Image | grep -m1 'Linux version'
```

两个版本串必须完全相同 —— 这正是"模块能被同一次构建的 `Image` 加载"的判据。

## 本仓库侧后续（按设计尚未编写）

一旦 tarball 存在：

1. `scripts/fetch-modules.sh`：把 tarball 下载到内核 `Image` 旁边，校验 `.ko` 数量与
   `vermagic` 串；
2. CI：对解压后的树执行 `depmod -b <staging> <kver>`；
3. 把 `.ko` **以及** `modules.dep` / `modules.alias` / `modules.softdep` 复制进
   `recovery/root/lib/modules/` —— **故意不放** `modules.load` 与
   `modules.load.recovery`，这样 first-stage init 永远不会去加载它们，TWRP 的容错加载器
   在"recovery 跑在 ROM 内核上"（OrangeFox 安装器路径）时还能回退到 ROM 的模块。
   见 [../RAMDISK-MODULES.md](../RAMDISK-MODULES.md) §3.1 与顶层 README 的警告。

本仓库的 `scripts/fetch-modules.sh` 已实现上述 1–3 步
（默认 `--stage-into recovery/root/lib/modules`；它拒绝创建任何 load list）。
它还没有被 CI 调用，所以在内核 release 出现之前不会有任何变化。届时工作流改动只有两行：

```yaml
      - name: Fetch matching kernel modules
        run: |
          set -euo pipefail
          ./scripts/fetch-kernel.sh "$KERNEL_URL" prebuilt/Image
          ./scripts/fetch-modules.sh "$MODULES_URL"          # stages into recovery/root/lib/modules

      - name: Install device tree into device/xiaomi/mars
        # 不变：rsync 会把 recovery/root/lib/modules/ 一起复制进构建树
```

`depmod` 索引来自 tarball 本身（内核的 `ci/package-modules.sh` 执行 `modules_install`，
其中会调用 `depmod`），所以 recovery CI 不需要 `depmod`；但如果 tarball 被人手工重打包，
请在 staging 之前用 `depmod -b <staging> <kver>` 重新生成。

## 两个必须说清的风险

* **绝对不要新增 `modules.load.recovery`。** first-stage init 对第一个加载失败的模块直接
  `LOG(FATAL)`，所以只要这个列表与"当前实际运行的内核"不匹配，开机就砖 —— 包括安装器
  路径（此时内核是 ROM 的）。这就是上面"不放 load list"设计的原因。
* **绝对不要混用不同构建的模块。** `CONFIG_LOCALVERSION_AUTO=y` 会把构建 commit 编进
  `vermagic`，所以 `Image` 与 `kernel-modules-*.tar.gz` 必须来自**同一次内核 release**，
  `fetch-modules.sh` 正是在检查这一点。
