# 更新日志

本仓库基于上游 [OPPO-Realme_kernel_4.14_MT6853](https://github.com/sanba0519/OPPO-Realme_kernel_4.14_MT6853)。
以下为本 fork 相比上游 `main` 的增量改动，按主题整理。

---

## 最近更新

### 🔧 SUSFS 升级到 v2.3.0（OPPO 4.14.186 适配）

从 v2.2.0 升级到 **SUSFS v2.3.0**，并针对 OPPO/ColorOS 的 4.14.186 vendor 树重新适配补丁。

- **引入 v2.3.0**（`a7cb1dd9a`）
  - 从 `JackA1ltman/NonGKI_Kernel_Build_2nd` 引入 4.14 版 SUSFS v2.3.0 补丁，落地到本仓库 `Patches/susfs_patch_to_4.14.patch`
  - 兼容 NANOHUB / MTK_SENSORHUB 两种传感器后端
  - workflow 改为自举：克隆**本仓库**而非硬编码上游（保证 CI 能编译到本 fork 的改动）；SUSFS 补丁直接从仓库内 `Patches/` 读取，不再依赖外部 wget

- **修复 fdinfo.c 编译失败**（`05045d952`）
  - 现象：`fs/notify/fdinfo.c:149: use of undeclared identifier 'user_mask'`
  - 原因：OPPO 老接口没有 `inotify_mark_user_mask()`；适配时将 SUS_KSTAT/SUS_MOUNT 两处统一改为内联表达式 `(mark->mask & IN_ALL_EVENTS)`，消除多余的局部变量声明

- **修复 task_mmu.c 编译失败**（`469ee5bb6`）
  - 现象：`fs/proc/task_mmu.c:823: error: void function 'smap_gather_stats' should not return a value`
  - 原因：ja 补丁的 SUS_MAP hunk 上下文用新版 `mss_stack`，在 OPPO 树里只能 fuzz 匹配到 `void smap_gather_stats` 内的 `smaps_walk.private = mss`，导致 `return 0;` 进 void 函数
  - 修复：将 SUS_MAP 早退逻辑重新锚定到 **int `show_smap`** 的 `__show_smap(m, &mss)` 处，`return 0;` 才合法
  - 验证：完整补丁在 OPPO 树上 dry-run **59/59 hunks 通过**

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
| `a7cb1dd9a` | SUSFS v2.3.0 引入 + workflow 自举 |
| `05045d952` | 修复 fdinfo.c `user_mask` 未声明 |
| `469ee5bb6` | 修复 task_mmu.c SUS_MAP 进 void 函数 |

---

## 当前特性

- SUSFS **v2.3.0**（非 GKI，OPPO 4.14.186 适配）
- 传感器 vendor 双层消毒（内核 + 用户态模块）
- 内核消毒 commit `52ba5be29` 已随本 fork 的 CI 构建
- RKSU / BakaSU 双路线，NoMount / BBR / SUSFS 等创意工坊功能