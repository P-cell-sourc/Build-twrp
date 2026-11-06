# TWRP 16 Builder

针对 **TWRP 官方最新 `twrp-16.0` 分支**（[TWRP-Test/platform_manifest_twrp_aosp](https://github.com/TWRP-Test/platform_manifest_twrp_aosp)）的 GitHub Actions 自动构建器，由 [P-cell-sourc/RO_Twrp](https://github.com/P-cell-sourc/RO_Twrp)（OrangeFox Builder for PD2415）复刻改造而来，**默认设备：vivo PD2415（iQOO Z11）**。

## 与 RO_Twrp 的差异

| 项目 | RO_Twrp（原版） | 本仓库（TWRP 16 版） |
|---|---|---|
| 源码 | OrangeFox fox_14.1 | TWRP 官方 `twrp-16.0`（可选 `twrp-14` / `twrp-12.1`） |
| 同步方式 | pinned manifest + 手工补丁 | 官方 manifest 开箱即用，直接 `repo sync` |
| 补丁链 | patches/（build/make、vendor/twrp） | 无需（TWRP 无此问题） |
| fox_env / OrangeFox 打包 | 需要 | 已移除 |
| 保留的工程化设计 | ✅ | ✅ 全部保留 |

**保留的健壮性设计**（继承自 RO_Twrp）：

- 12 GiB swap 自动补齐（`setup-swap.sh`，只补差额）
- 磁盘/内存容量双重自检（同步前 75 GiB、删除 `.repo` 后 35 GiB）
- ccache 编译缓存（跨运行复用，key 按分支+机型区分）
- `repo sync` 失败自愈重试（4 次，自动清除损坏项目的 `.git`）
- boot/vendor_boot 镜像地址静态校验（`check-boot-offsets.sh`，防 32 位偏移溢出在最后一步才炸，白跑几十分钟）。**已针对 PD2415 增强**：支持 `BOARD_MKBOOTIMG_ARGS += --ramdisk_offset $(BOARD_RAMDISK_OFFSET)` 的变量引用与 `+=` 多行追加写法（原版只认字面量，遇 PD2415 这类设备树会误判失败）
- 逐目标构建日志 + 60 秒心跳输出（防 CI 判超时）
- 产物校验：recovery ramdisk 载荷比对、mkbootfs 静默失败检测、unpack_bootimg header 复核
- 失败自动诊断（5 段式：目标结果 / FAILED 目标 / 错误关键字 / 产物清单 / BoardConfig 摘要）
- 独立 `release` job 发布 GitHub Release（含 sha256sums）

## 使用方法

1. 把本仓库内容推到你自己的 GitHub 仓库（或直接 Fork/Use this template）
2. 仓库 **Settings → Actions → General → Workflow permissions** 选 **Read and write permissions**
3. **Actions → TWRP 16 Build → Run workflow**，参数默认值即为 vivo PD2415，直接运行
4. 约 1~2 小时后到 Releases 下载镜像

### 关键参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `MANIFEST_BRANCH` | `twrp-16.0` | 也可选 `twrp-14`、`twrp-12.1` |
| `DEVICE_TREE_URL` | P-cell-sourc/twrp_device_vivo_PD2415_twrp | 换机型改这里 |
| `DEVICE_PATH` | `device/vivo/PD2415` | 必须与设备树 `BoardConfig.mk` 的 `DEVICE_PATH` 一致 |
| `DEVICE_NAME` | `PD2415` | 与 `twrp_PD2415.mk` 的 `PRODUCT_DEVICE` 一致 |
| `MAKEFILE_NAME` | `twrp_PD2415` | 产品 makefile 文件名（不带 `.mk`） |
| `BUILD_TARGET` | `recoveryimage` | 可空格分隔多个目标；recovery 在 vendor_boot 的机型用 `vendorbootimage` |
| `PUBLISH` | `true` | 成功后自动发 Release |

### PD2415 特别说明

- 设备树自带 `prebuilt/kernel`、`dtb.img`、`dtbo.img`，无需内核内编。
- 设备树 `BoardConfig.mk` 写了 `TARGET_NO_RECOVERY := false`，故默认 `BUILD_TARGET=recoveryimage`。若编译成功但刷入无效（该机型实为 recovery 打进 vendor_boot 的虚拟 A/B 设备，参考 long36708 树的说法），把 `BUILD_TARGET` 改为 `vendorbootimage` 重跑。
- 解密失败（系统为 Android 16）时，确认源分支为 `twrp-16.0`；若编译失败再退回 `twrp-14`。

## 文件结构

```
.github/workflows/twrp-build.yml   # 主工作流
.github/scripts/setup-swap.sh      # swap 补齐脚本
.github/scripts/check-boot-offsets.sh  # 镜像地址静态校验
scripts/convert.sh                 # *.dependencies → local_manifests 转换
```

## 致谢

- [P-cell-sourc/RO_Twrp](https://github.com/P-cell-sourc/RO_Twrp) — 工程化设计来源
- [kmiit/twrp_device_oplus_sm87xx](https://github.com/kmiit/twrp_device_oplus_sm87xx) — twrp-16.0 可行性参照
- [azwhikaru/Action-TWRP-Builder](https://github.com/azwhikaru/Action-TWRP-Builder) — 原始 Actions 构建器
- TeamWin / TWRP-Test — twrp-16.0 官方源码
