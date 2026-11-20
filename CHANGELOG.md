# 更新日志

本仓库基于上游 [OPPO-Realme_kernel_4.14_MT6853](https://github.com/sanba0519/OPPO-Realme_kernel_4.14_MT6853)。
以下为本 fork 相比上游 `main` 的增量改动，按主题整理。

---

## 最近更新

### 🔁 SUSFS 策略调整：遵循上游 sanba0519

SUSFS 补丁不再在本仓库内维护，回归上游模式：

- **撤回 vendored 2.3.0 适配版**（revert `469ee5bb6`、`05045d952`、`a7cb1dd9a`）
  - 删除仓库内 `Patches/susfs_patch_to_4.14.patch`
  - workflow 的 SUSFS 段恢复为上游原始逻辑
- **SUSFS 补丁来源**：CI 开启 SUSFS 开关时实时从上游
  `sanba0519/Kernel-patch@main` 获取 `susfs_patch_to_4.14.patch`
  （当前内容为 **SUSFS v2.3.0** 含上游编译修复，dry-run 21/21 hunks 干净通过）
- **与上游 workflow 的唯一差异**：clone URL 使用 `${{ github.repository }}`
  （fork 自举必需，保证 CI 编译的是本 fork 的提交，与 SUSFS 无关）

---

## 传感器 vendor 消毒（过 Miscellaneous Check 5）

风控 SDK 会统计 vendor 含 `AOSP`（不区分大小写）的传感器数量，>3 判定为模拟器/云手机。本仓库从**内核**和**用户态**两层封堵。

- **内核侧**（`52ba5be29`）
  - 新增 `drivers/misc/mediatek/sensor/2.0/sensor_vendor_sanitize.h`（`vendor_contains_aosp()` + `sensor_vendor_sanitize()`）
  - `mtk_nanohub.c`：固件上报透传点 + 驱动注册点消毒
  - `sensorhub/sensor_list.c`：SCP 共享内存透传点消毒
  - 任何 vendor 含 `aosp`（大小写不敏感）一律等长改写为 `mtk`，兼容 clang/gcc，两个后端都覆盖

- **用户态脚本 `sensor_vendor_fix.sh`**（`c9d853db3`）
  - 把 `libsensorservice.so` 里的 `AOSP` 字面量等长改写为 `MTK!`（链接器合并字符串，一次替换全部生效）
  - 以 KernelSU/RKSU/BakaSU/NoMount 模块形式 systemless 挂载，不动 system 分区
  - 64/32 位自动探测、SELinux 标签继承、三重校验（残留=0 / 尺寸不变 / 差异字节数）、幂等
  - 子命令：`apply` / `status` / `restore`

- **文档**（`1f1f3ee07`、`7aa6519e4`、`cd6561b5c`、`1f00f361f`）
  - README 新增「刷入后教程」与「传感器 vendor 修复」说明，Feature 段补条目

---

## 完整提交对照

| 提交 | 说明 |
|---|---|
| `52ba5be29` | 内核侧传感器 vendor 消毒（nanohub / sensorhub） |
| `c9d853db3` | `sensor_vendor_fix.sh` 用户态一键脚本 |
| `1f1f3ee07` | 文档：传感器双层修复说明 |
| `7aa6519e4` | 文档：刷入后教程 |
| `cd6561b5c` | 文档：README 精简对齐上游风格 |
| `1f00f361f` | 文档：恢复 Feature 段 + 补传感器条目 |
| `4b33994a7` | Revert: 撤回 task_mmu SUS_MAP 修复 |
| `cc14b0b5d` | Revert: 撤回 fdinfo user_mask 修复 |
| `502028266` | Revert: 撤回 SUSFS v2.3.0 vendored 补丁，回归上游 wget 模式 |
| `4b5932a1a` | fix(ci): revert 后保留 fork clone-source 修复 |

---

## 当前特性

- SUSFS **v2.3.0**（遵循上游 `sanba0519/Kernel-patch`，构建时实时获取）
- 传感器 vendor 双层消毒（内核 + 用户态模块）
- 内核消毒 commit `52ba5be29` 已随本 fork 的 CI 构建
- RKSU / BakaSU 双路线，NoMount / BBR / SUSFS 等创意工坊功能