# 在自己的服务器上构建 mars recovery

这是现在主要的构建方式：一台你自己控制的普通 Ubuntu 机器，而不是 GitHub 的 runner。
（`.github/workflows/` 里的云端工作流已经删除：每一轮要 40–70 分钟、受磁盘限制，而且
从来没有跑到完整编译；本地服务器更快、更好迭代。）

`scripts/build-local.sh` 实现了下面所有步骤；本文说明它做什么、需要什么、要多久，
以及坏了怎么办。

---

## 1. 服务器需要什么

| 资源 | 最低 | 推荐 | 原因 |
| --- | --- | --- | --- |
| 系统 | Ubuntu 22.04 x86_64 | Ubuntu 24.04 x86_64 | 该源码树按 22.04+ 构建；其它发行版也能用，但下面的 apt 命令是按 Ubuntu 写的 |
| CPU | 4 核 | 24 核 | Android 14 树；`-j$(nproc)` 上限 16 |
| 内存 | 16 GB | 22 GB + swap | Soong + ninja 链接；8 GB 用 `-j8` 会 OOM（改 `--jobs 4`） |
| 磁盘 | 150 GB 可用 | **200 GB 以上** | 实测：`repo sync` + 缓存后占 118 GB，`out/` 还没算 |
| 网络 | 约 40 GB 下载 | — | 裁剪过的 Android 14 manifest 的 `repo sync` |
| 首次构建耗时 | 约 2 小时 | 60–90 分钟 | 同步 20–35 分钟 + 编译 30–60 分钟 |

这里没有任何 CI 专属的东西：不用 Docker、不用 GitHub token。唯一的"密钥"是构建本身
就要用的 AVB 测试签名（`external/avb/test/data/testkey_rsa2048.pem`、
`vendor/recovery/security/miui`）。

## 2. 一次性准备

```sh
# 2.1 构建依赖（与 CI 安装的完全一致）
sudo apt-get update
sudo apt-get install -y \
  bc bison build-essential ccache cpio curl flex git git-lfs gnupg gperf \
  imagemagick libelf-dev libssl-dev libxml2 libxml2-utils lzop pngcrush \
  rsync schedtool squashfs-tools unzip xsltproc zip zlib1g-dev zstd lz4 \
  python3 patchelf openjdk-17-jdk-headless

# 32 位 multilib（少数工具需要；部分包装不上也不致命）
sudo dpkg --add-architecture i386 && sudo apt-get update
sudo apt-get install -y gcc-multilib g++-multilib libc6-dev-i386 \
  lib32z1-dev lib32readline-dev lib32ncurses5-dev libncurses5-dev

# 2.2 `repo`：apt 里的版本太旧，用 Google 的启动器
mkdir -p ~/bin
curl -fsSL https://storage.googleapis.com/git-repo-downloads/repo -o ~/bin/repo
chmod a+x ~/bin/repo
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc && export PATH="$HOME/bin:$PATH"
repo --version

# 2.3 git 身份（部分工程需要）与 ccache
git config --global user.name  "mars builder"
git config --global user.email "builder@localhost"
git config --global color.ui false
ccache -M 50G
```

`scripts/build-local.sh` 会逐项检查这些，缺什么就直接打印对应的 `apt-get` 命令。

## 3. 构建——简版

```sh
# 3.1 取本设备树（放哪都行；脚本会把它复制进 AOSP 树）
git clone https://github.com/S0SL/orangefox_mars.git ~/orangefox_mars

# 3.2 构建。首次：同步（20-35 分钟）+ 取内核 + 编译（30-60 分钟）
~/orangefox_mars/scripts/build-local.sh --fox-dir ~/fox_14.1
```

所有产物落在 `~/orangefox_mars/dist/`：

| 文件 | 是什么 |
| --- | --- |
| `boot.img` | **recovery 本体** —— 内核 + OrangeFox recovery ramdisk（mars 没有 recovery 分区，所以是 recovery-as-boot） |
| `recovery.img` | 同一 ramdisk 的副产物；**不要**在 mars 上刷它 |
| `OrangeFox-*.zip` | 可刷入的安装包（如果 OrangeFox 脚本生成了的话） |
| `BUILD-INFO.txt` | 用什么构建出什么，附带 `sha256sum` |

本仓库里没有任何东西会写设备。想试的时候：

```sh
adb reboot bootloader
fastboot boot boot.img          # 只启动一次，不写任何分区
# 确认能用之后，才 fastboot flash boot boot.img
```

常用参数：`--jobs N`、`--skip-sync`（树已同步）、`--skip-kernel`（Image 已就位）、
`--clean`（先删 `out/`）、`--no-ccache`、`--ccache-dir DIR`、
`--build-type Beta|Stable`、`--kernel-url URL`、`--releases "ap2a"`
（lunch 的 release 名，见 §6）。

## 4. 构建——手动版

如果你想自己一步步来，或者单独调试某一步，下面就是 `scripts/build-local.sh` 实际执行的命令：

```sh
export FOX_DIR="$HOME/fox_14.1"
export PATH="$HOME/bin:$PATH"

# 4.1 同步 OrangeFox fox_14.1（20-35 分钟，约 35 GB）
~/orangefox_mars/scripts/fox-sync.sh "$FOX_DIR"
#   - 克隆 OrangeFox/sync 并执行 ./orangefox_sync.sh --branch 14.1
#   - 绕过上游的一个 bug（vendor/twrp 补丁在错误目录里找），并**校验**补丁确实打上了
#   - 树不完整时拒绝继续

# 4.2 把本设备树放到构建系统期望的位置
mkdir -p "$FOX_DIR/device/xiaomi/mars"
rsync -a --exclude '.git/' --exclude '.github/' --exclude 'scripts/' \
      --exclude 'docs/' --exclude 'dist/' \
      ~/orangefox_mars/ "$FOX_DIR/device/xiaomi/mars/"

# 4.3 取预编译内核 Image（这里不编译内核）
~/orangefox_mars/scripts/fetch-kernel.sh \
    "https://github.com/S0SL/android_kernel_xiaomi_sm8350/releases/download/boot-img-LOS23.2-20260930-bakasu/anykernel3-BakaSU-susfs-v2.3.0-mars.zip" \
    "$FOX_DIR/device/xiaomi/mars/prebuilt/Image"

# 4.4 构建
cd "$FOX_DIR"
export LC_ALL=C FOX_BUILD_DEVICE=mars FOX_BUILD_TYPE=Unofficial
export ALLOW_MISSING_DEPENDENCIES=true
export USE_CCACHE=1 CCACHE_DIR="$FOX_DIR/.ccache"

source build/envsetup.sh            # 注意：不要在 `set -u` 下执行
lunch twrp_mars-ap2a-eng            # 三段式：<product>-<release>-<variant>
mka recoveryimage                   # -> out/target/product/mars/boot.img

# 4.5 收集
ls -l out/target/product/mars/{boot.img,recovery.img}
```

**ccache 是最有效的加速手段。** 在 2.3 配置、在 4.4 导出；缓存热了以后，设备树小改动
的重建只需要几分钟而不是约 40 分钟。`CCACHE_DIR` 放快盘、给 50 GB，`ccache -s` 看命中率。

## 5. 耗时（实测）

| 阶段 | 观测值 | 备注 |
| --- | --- | --- |
| 装 apt 依赖 | 约 50 秒 | 按层拆开装（一次性全装曾在云端卡住 10 分钟以上） |
| `repo sync` | **19 分钟**（缓存热）/ **35 分钟**（冷） | 来自 CI 日志第 1 轮与第 8 轮 |
| `.repo` 缓存保存 | 约 5 分钟 | 仅 CI；本地服务器直接保留树 |
| `fetch-kernel.sh` | 几秒 | 约 20 MB 的 AnyKernel3 压缩包 |
| `mka recoveryimage` | 30–60 分钟（估计） | 这是只含 recovery 的最小构建 |
| **首次构建合计** | **约 60–100 分钟** | 另外约 40 GB 下载 |

磁盘（CI runner，共 146 GB）：同步前 36 GB 已用，**同步 + 缓存后 118 GB**，
编译开始时剩 28 GB。建议按 200 GB 预算，尤其是还要开 ccache 时。

## 6. 排障——本项目真实遇到过的每一个失败

| 日志里的现象 | 原因 | 处理 |
| --- | --- | --- |
| `build/envsetup.sh: line 21: TOP: unbound variable` | 在 `set -u` 下 `source build/envsetup.sh` | 先 `set +u` 再 source，然后恢复 `set -u`；`build-local.sh` 已经这么做 |
| `patch-vendor-twrp-fox_14.1.diff: No such file or directory` + `Error! Failed to patch the twrp-14 vendor/twrp` | 上游 `orangefox_sync.sh` 在错误目录找补丁，且只告警 | `scripts/fox-sync.sh` 预先放好补丁，然后校验 `vendor/twrp/config/BoardConfigSoong.mk` 里确实包含了 `orangefox_soong.mk` |
| `Invalid lunch combo: twrp_mars-eng` / `Valid combos must be of the form <product>-<release>-<variant>` | 这套 build/make 分支要求三段式 | 用 `twrp_mars-ap2a-eng`（或直接跑 `build-local.sh`，它会自动探测） |
| `No release config found for TARGET_RELEASE: bp2a. Available releases are: ap2a` | release 名猜错了 | 从 `ls build/release/release_configs/` 取；本 manifest 是 `ap2a` |
| `device/xiaomi/mars/BoardConfig.mk:124: error: BOARD_BUILD_SYSTEM_ROOT_IMAGE is obsolete` | Kati 对 `KATI_obsolete_var` 的任何赋值都是硬错误 | 设备树里已修；`scripts/check-tree.sh` 会在它复发时直接失败（148 个过时变量名在 `scripts/obsolete-build-vars.txt`） |
| `** Don't have a product spec for: 'twrp_mars'` | 几乎总是**连带反应**：更早的 `dumpvars` 失败 | 往上翻找第一个真正的错误，不要追这条 |
| `vendor/qcom/opensource/commonsys-intf/display is not in the manifest`（警告） | 该 CAF 工程不在此 manifest 里 | 设计上无害；若屏幕不亮，把三个 `.so` 放进 `recovery/root/vendor/lib64/`（见 `docs/history/KNOWN_ISSUES.md` I16） |
| `No space left on device` | 磁盘 | 腾空间或搬树；建议 200 GB |
| 构建 OOM / `virtual memory exhausted` | 内存 | `--jobs 4` 或更低 |
| recovery 里触摸或屏幕不工作 | 内核模块不在 ramdisk 里，且 ROM 的模块与我们的内核不匹配 | 在内核改动落地前是预期现象，见 `docs/RAMDISK-MODULES.md`；ROM 的 `.ko` 永远加载不进我们的预编译 Image（vermagic 里含构建 commit） |

把树交给构建之前，可以先做**不依赖 AOSP** 的静态检查（文件齐全、fstab/flags 可解析、
lunch 三段式、无过时构建变量、shell 语法）：

```sh
~/orangefox_mars/scripts/check-tree.sh
```

## 7. 这条路径里**故意没有**的东西

* **没有内核编译** —— 内核是预编译的（`TARGET_PREBUILT_KERNEL`）。内核仓库侧那处"顺带
  发布触摸/显示模块"的改动已准备好、做过 dry-run，但**没有推送**：见 `docs/kernel-prep/`。
* **没有 CI 缓存与制品上传** —— 那些只在已删除的云端工作流里；`scripts/fox-sync.sh`
  是纯构建逻辑，原样可复用。
* **本仓库任何脚本都不会刷机。**

> 注意：`/data` 解密**已经实现**（v4.9 实测可用），不再属于"范围外"。
> 早期文档里"phase 2、不在范围内"的说法已过时，当前状态见 `docs/STATUS.md`。

## 8. 构建成功之后

1. 把 `dist/BUILD-INFO.txt` 与镜像放在一起 —— 它记录了内核 URL 和 `sha256sum`。
2. 先 `fastboot boot dist/boot.img`；能起来再 flash。
3. 观察 recovery 的 `/tmp/recovery.log`（或 `adb shell cat /tmp/recovery.log`）里与模块
   加载有关的行：`Checking directory: /lib/modules`、`Modules Loaded: N`
   —— 这是"触摸/显示能不能起来"的直接测量。

---

## 9. 无人值守构建（systemd：开机自启、自愈）

首次构建是 1.5–2.5 小时的同步 + 编译，而 SSH 会话是整条链路里最不可靠的一环。因此把构建
做成 **systemd 服务**：掉线、重启、网络抖动都不影响，且不需要有人登录。

仓库里两个文件：

| 文件 | 安装为 | 作用 |
| --- | --- | --- |
| `scripts/mars-build-run.sh` | `/usr/local/bin/mars-build-run.sh` | 幂等、自愈的入口 |
| `scripts/mars-build.service` | `/etc/systemd/system/mars-build.service` | unit（`WantedBy=multi-user.target`、`Restart=on-failure`） |

### 安装（不需要网络）

把两个文件以任意方式拷到机器上（拷到 `~/` 即可；`git pull` 后它们也在
`~/orangefox-mars/scripts/`），然后执行四行：

```sh
sudo install -m 755 ~/mars-build-run.sh /usr/local/bin/mars-build-run.sh
sudo install -m 644 ~/mars-build.service /etc/systemd/system/mars-build.service
sudo systemctl daemon-reload
sudo systemctl enable --now mars-build
```

之后日常命令就一行：

```sh
sudo systemctl enable --now mars-build     # 开机自启 + 立刻开始
systemctl status mars-build                # 现在在干什么
tail -f ~/logs/systemd-build.log           # 跟踪构建日志
```

安装不需要网络，unit 也不会从 GitHub 拉任何东西 —— 完全用磁盘上已有的树。

### 入口保证什么

* **幂等**：`flock` 保证同时只有一个实例；`dist/boot.img` 已存在就直接退出
  （`MARS_FORCE=1` 强制重建）；同步 + 打补丁阶段成功后写 `~/.mars-sync-complete`，
  之后每次启动都直接进入编译。
* **自愈**：吃网络的部分循环重试（12 次同步尝试，每次之前先修复一直失败的项目），
  `Restart=on-failure` + `RestartSec=60` 让 systemd 重试失败的运行
  （`StartLimitBurst=20` 防止无限空转）。成功构建退出码 0，不再重试。
* **内存守护**：watchdog 线程在 `MemAvailable` 低于 `MARS_MEM_LOW_MB` 时把重下载
  `SIGSTOP` 暂停，回到 `MARS_MEM_HIGH_MB` 以上再 `SIGCONT`，避免一次全历史打包把
  `sshd` 或隧道客户端一起拖死；退出时会恢复所有被暂停的进程。
* **修复了这台机器真实遇到的故障**：`repo` 只对"新"项目用 `--depth=1`，对目录已存在的
  项目会**静默拉完整历史**（10–20 GB 的 pack，正是反复失败的原因）。入口发现项目目录里
  只有失败的 `tmp_pack_*` 时会删掉它、让重试重新走浅克隆，并在两次尝试之间清理残留的
  `tmp_pack_*`（某次清出 54 GB）。详见 `docs/BUILD-STATUS.md` 的 "Server build 1"。

### 可调项（都可选，通过 unit 的 `Environment=` 设置）

| 变量 | 默认 | 含义 |
| --- | --- | --- |
| `MARS_HOME` | `/home/user` | AOSP 树与设备树所在目录 |
| `MARS_JOBS` | `$(nproc)`，上限 16 | `mka` 并行度 |
| `MARS_MEM_LOW_MB` | `2000` | `MemAvailable` 低于此值时暂停下载 |
| `MARS_MEM_HIGH_MB` | `4000` | 高于此值时恢复 |
| `FOX_BUILD_TYPE` | `Unofficial` | 传给 `FOX_BUILD_TYPE` |
| `MARS_FORCE` | *(未设置)* | `1` 表示即使 `dist/boot.img` 存在也重建 |

### 日志与产物

| 路径 | 内容 |
| --- | --- |
| `~/logs/systemd-build.log` | unit 的全部输出（journal 里也有） |
| `~/orangefox-mars/dist/boot.img` | **产物** |
| `~/orangefox-mars/dist/BUILD-INFO.txt` | 用什么构建出什么，附 `sha256sum` |

### 机器断电/被中断之后

不需要特殊处理：unit 是 `enabled` 的，重启即重新开始；同步从 `~/.repo` 续传、不会重新下载；
`~/.mars-sync-complete` 会跳过已完成阶段。但被中断的 checkout 可能留下过期 git 锁，症状是
`fatal: unable to create '.../.git/index.lock': File exists` 且 repo 报
`Checking out local projects failed`。清理：

```sh
find ~/fox_14.1 -name index.lock -mmin +3 -print -delete
```
