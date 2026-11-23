# 历史文档（已被取代）

这里的文件保留**过程记录**，不是当前操作指南。当前结论见
[../STATUS.md](../STATUS.md)，构建步骤见 [../BUILD-ON-SERVER.md](../BUILD-ON-SERVER.md)，
排障编年史见 [../BUILD-STATUS.md](../BUILD-STATUS.md)。

| 文件 | 是什么 | 被什么取代 |
| --- | --- | --- |
| [V4-REPORT.md](V4-REPORT.md) | v4 的交付报告（`/data` FBE 解密 + USB 修复 + 挖孔几何） | [../STATUS.md](../STATUS.md) §2 |
| [V4.6-PLAN.md](V4.6-PLAN.md) | v4.6 的离线计划（gatekeeper 链路分析） | [../STATUS.md](../STATUS.md) §2 第 9 条 |
| [REBUILD-V3.md](REBUILD-V3.md) | 服务器扩容后重建 v3 的一次性清单 | 已被 `scripts/build-local.sh` 流程取代 |
| [PHASE2-CRYPTO.md](PHASE2-CRYPTO.md) | phase 2（解密 `/data`）的调研与计划 | 已实现，见 [../STATUS.md](../STATUS.md) |
| [FSTAB-DIFF.md](FSTAB-DIFF.md) | TWRP 与 LineageOS 的 `recovery.fstab`/crypto 差异分析 | 结论已落入设备树的 `recovery/root/system/etc/recovery.fstab` |
| [KNOWN_ISSUES.md](KNOWN_ISSUES.md) | phase 1 时期的已知问题清单（I1–I19 / R1–R5） | [../STATUS.md](../STATUS.md) §4（当前有效的部分已并入） |

> 这些文件里的“待修/未实现”描述的是**当时**的状态（例如 KNOWN_ISSUES 说
> “解密不在范围内”、亮度/几何值是借来的），不要据此判断现状。
