# 欧加真 MT6853 系列通用4.14内核自动化编译脚本
[![Telegram](https://ziadoua.github.io/m3-Markdown-Badges/badges/Telegram/telegram2.svg)](https://t.me/mt6853o) 
[![QQ群](https://img.shields.io/badge/QQ-1098718272-blue?style=flat&logo=%3Csvg%20role%3D%22img%22%20viewBox%3D%220%200%2024%2024%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Ctitle%3EQQ%3C%2Ftitle%3E%3Cpath%20d%3D%22M21.395%2015.035a40%2040%200%200%200-.803-2.264l-1.079-2.695c.001-.032.014-.562.014-.836C19.526%204.632%2017.351%200%2012%200S4.474%204.632%204.474%209.241c0%20.274.013.804.014.836l-1.08%202.695a39%2039%200%200%200-.802%202.264c-1.021%203.283-.69%204.643-.438%204.673.54.065%202.103-2.472%202.103-2.472%200%201.469.756%203.387%202.394%204.771-.612.188-1.363.479-1.845.835-.434.32-.379.646-.301.778.343.578%205.883.369%207.482.189%201.6.18%207.14.389%207.483-.189.078-.132.132-.458-.301-.778-.483-.356-1.233-.646-1.846-.836%201.637-1.384%202.393-3.302%202.393-4.771%200%200%201.563%202.537%202.103%202.472.251-.03.581-1.39-.438-4.673%22%2F%3E%3C%2Fsvg%3E&logoColor=%231EBAFC&logoSize=auto&label=QQ%E7%BE%A4&labelColor=%2312B7F5&color=%23FFFFFF&cacheSeconds=3600&link=https%3A%2F%2Fqm.qq.com%2Fq%2FYF1CEUBJ0O
)](https://qm.qq.com/q/UbZQsSlIk2)

## android_kernel_4.14_MT6853_Action
>[!note]
>适用机型：MT6853 OPPO/Reamle OPPO A72, OPPO A53, OPPO K7x, OPPO Reno4SE, Realme Q2, Realme V15, Realme Q2 pro, OPPO A95, Realme V5 5G, Realme X7 5G Android 12 RKSU kernel
>
>版本：4.14.186
| 务必升级到安卓12再刷入此内核
>
***

## 内核基本信息
版本：4.14.186+

安卓版本：Android12

支持的Root管理器：
 - RKSU
 - BakaSU
***

## 如何使用？
 - star本项目（对我真的真的很重要><）
 - fork本项目（可以只fork `main` 分支）
 - 点击 GitHub顶栏的 `Action` 按钮
 - 点击 `Build Kernel for MT6853` 一栏
 - 选择Root管理器以及功能 点击`Run workflow`
 - 等待构建完成并刷入~

## 如何刷入？
 * 首先务必Root你的手机并升级系统至安卓12
    * 在rec刷入
      
      or
    * 先用Magisk `root`你的手机 然后使用刷写内核工具刷入

      比如`Franco Kernel Manager`

***

## 刷入后教程（按顺序做）

### 第 1 步：确认内核已生效

```sh
su -c 'cat /proc/version'
```

- 应显示 `4.14.186-by-sanba` + `clang` 版本 + **本次构建的日期**
- 时间戳对不上 = 刷错包或没刷上，重新刷

### 第 2 步：确认 Root 可用

- 安装 RKSU / BakaSU 管理器并授予终端 Root：

```sh
su -c 'id'    # 应输出 uid=0(root)
```

### 第 3 步：传感器 vendor 修复（过 Miscellaneous Check 5）

> 风控 SDK 会统计 vendor 含 `AOSP` 的传感器数量（>3 判定模拟器）。
> HAL 缺融合传感器时框架会兜底注册 8 个 `AOSP` vendor 传感器，必须处理。

```sh
# ① 先看基线计数
su -c 'dumpsys sensorservice | grep -ci aosp'

# ② 下载并执行修复脚本（raw 失败会自动切 jsDelivr）
su -c 'cd /sdcard/Download && { wget -O svf.sh https://raw.githubusercontent.com/HoshinoOVO/OPPO-Realme_kernel_4.14_MT6853/main/sensor_vendor_fix.sh || wget -O svf.sh https://cdn.jsdelivr.net/gh/HoshinoOVO/OPPO-Realme_kernel_4.14_MT6853@main/sensor_vendor_fix.sh; } && wc -c svf.sh && sh svf.sh apply'

# ③ 重启，然后复验：计数应为 0
su -c 'dumpsys sensorservice | grep -ci aosp'
```

原理与细节见下文[「传感器 vendor 修复（双层方案）」](#传感器-vendor-修复双层方案过-miscellaneous-check-5)。

### 第 4 步：按构建勾选使用功能模块

| 功能 | 说明 |
|---|---|
| SUSFS | 挂载/文件系统隐藏，配合 Root 管理器模块使用 |
| NoMount | systemless 挂载器；检测到 bootloop 会自动进安全模式禁用模块，进系统后修复模块再重启即可 |
| BBR | 网络拥塞控制，勾选即启用 |
| DroidSpaces | 独立空间，按其模块说明使用 |

### 第 5 步：日常维护

- **OTA/大版本升级后**：重新用 Actions 构建刷入内核 + `sensor_vendor_fix.sh` 先 `restore` 再 `apply`
- **清除数据/重刷 ROM 后**：模块在 `/data` 会被清掉，重新执行第 3 步
- **想还原传感器修复**：`su -c 'sh /sdcard/Download/svf.sh restore'` 后重启

### 常见问题

- **卡开机第二屏循环** → 多为 NoMount 安全模式已自动禁用模块，可正常进系统；
  若是 `sensor_vendor_fix` 模块引起，执行
  `su -c 'chcon u:object_r:system_lib_file:s0 /data/adb/modules/sensor_vendor_fix/system/lib64/libsensorservice.so'` 后重启
- **某 App 还有其他风控项报红** → 传感器修复只覆盖 Miscellaneous Check(5)，
  root 特征/挂载/hook 等属于其他检测项，带着报错截图来群里问

***

## Feature:

Upstream:
- [x] lxc
- [x] docker
- [x] kvm

此项目：
- [x] 适配最新版[RKSU](https://github.com/rsuntk/KernelSU)
- [x] 内置[BakaSU](https://github.com/Baka-SU/BakaSU)
- [x] 开放创意工坊
     - [x] BBR加速
     - [x] SUSFS-V2.2.0
     - [x] [DroidSpaces](https://github.com/ravindu644/Droidspaces-OSS)
     - [x] 墓碑 - [Cirno](https://cirno.shrairo.top/)
     - [x] [NoMount](https://github.com/maxsteeel/nomount)
     - [x] 修复几个CVE漏洞   --CVE-2026-43499 & CVE-2022-29581
     - [ ] [BBG](https://github.com/vc-teahouse/Baseband-guard)防格机
     - [ ] So on...
- [x] 修补bpf
- [x] 自动发布到[TG频道](https://t.me/mt6853o)

## 传感器 vendor 修复(双层方案,过 Miscellaneous Check 5)

### 背景

部分风控 SDK(银行/游戏等 App 的环境检测)有一条 **Miscellaneous Check(5)**:
遍历 `SensorManager.getSensorList(TYPE_ALL)`,统计 `Sensor.getVendor()` 包含 `AOSP`
(不分大小写)的传感器数量,**>3 即判定为模拟器/云手机**。

真机上这些 `AOSP` vendor 从哪来?当 sensors HAL **未提供融合传感器**时,Android 框架的
`SensorService`(`frameworks/native/services/sensorservice/`)会用 `vendor="AOSP"`
注册兜底虚拟传感器:

| 数量 | 传感器 | 注册条件 |
|---|---|---|
| 6 | 重力 / 线性加速 / 旋转矢量 / 游戏旋转矢量 / 地磁旋转矢量 / 方向 | HAL 未提供对应类型时兜底 |
| 2 | Corrected Gyroscope Sensor / Gyroscope Bias (debug) | 有 加速度计+陀螺仪+磁力计 就**无条件**注册 |

也就是说:磁力计齐全的真机**天生自带 2 个** AOSP vendor 传感器;再叠加 HAL 缺失
融合传感器的 6 个兜底,计数冲到 8 → 直接超阈值。

### 方案①内核侧(main 分支已内置)

`drivers/misc/mediatek/sensor/2.0/sensor_vendor_sanitize.h` +
`mtk_nanohub/mtk_nanohub.c` 与 `sensorhub/sensor_list.c` 调用点:
所有经内核/SCP 上报的传感器 vendor,包含 `aosp`(不分大小写)一律改写为 `mtk`。
兼容 NANOHUB / MTK_SENSORHUB 两种后端、clang/gcc、RKSU/BakaSU 构建路线。
细节见 commit `52ba5be29`。

### 方案②用户侧:sensor_vendor_fix.sh 一键脚本

针对 SensorService 兜底的 8 个:把 `libsensorservice.so` 中的 `AOSP` 字面量
**等长**改写为 `MTK!`(链接器合并字符串,一次替换全部生效),以模块形式
**systemless** 挂载(支持 KernelSU/RKSU/BakaSU/NoMount),不动 system 分区,
传感器功能零影响。

```sh
# 下载(已 root 的任意终端)
su -c 'cd /sdcard/Download && wget -O svf.sh https://raw.githubusercontent.com/HoshinoOVO/OPPO-Realme_kernel_4.14_MT6853/main/sensor_vendor_fix.sh'

# 打补丁 + 生成模块,重启生效
su -c 'sh /sdcard/Download/svf.sh apply'

# 查看状态(dumpsys 计数 + 自动判定)
su -c 'sh /sdcard/Download/svf.sh status'

# 还原(重启后恢复原状)
su -c 'sh /sdcard/Download/svf.sh restore'
```

- 64/32 位库自动探测,模块内路径镜像原始路径,SELinux 标签自动继承
- 内置三重校验(AOSP 残留=0 / 文件尺寸不变 / 差异字节数=4×出现次数),任一不过拒绝写入
- 幂等:重复执行安全;已打过补丁的设备执行会提示"无需操作"

### 验证

```sh
su -c 'dumpsys sensorservice | grep -ci aosp'      # 期望 0(≤3 即不命中检测)
su -c 'dumpsys sensorservice | grep -i "MTK!"'     # 8 个传感器应显示新 vendor
```

### FAQ

**Q: 用 NoMount 挂载后重启卡开机第二屏?**
A: bind mount 的 so 带着 `/data/adb` 的 SELinux 标签,system_server 加载被拒导致崩溃循环。
   脚本 v1.1 起自动继承标签并写入 `post-fs-data.sh` 兜底。老模块手动修复:
   ```sh
   su -c 'chcon u:object_r:system_lib_file:s0 /data/adb/modules/sensor_vendor_fix/system/lib64/libsensorservice.so'
   ```

**Q: OTA/大版本升级后要重做吗?**
A: 要。`restore` → 重启 → 重新 `apply`(so 会随 ROM 更新)。

**Q: 怎么彻底还原?**
A: `restore` 并重启;或直接删除 `/data/adb/modules/sensor_vendor_fix` 目录。

**Q: 融合传感器精度受影响吗?**
A: 不受。改写的只是 vendor 元数据,算法原样运行。

**Q: 内核侧和用户侧都做吗?**
A: 建议都做,互为纵深。只做其一也能过检测(5),但覆盖链路不同。

***
## 鸣谢
[RKSU](https://github.com/rsuntk/KernelSU) 一个专门为`no-GIK`内核的KernelSU分支

[BakaSU](https://github.com/Baka-SU/BakaSU) KernelSU 的分支

[NonGKI_Kernel_Build_2nd](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd) 一个专注于No-GKI内核编译的项目

[android_kernel_mtk6853](https://github.com/wu17481748/android_kernel_mtk6853) 上游项目 没有它就没有现在的项目

[android_kernel_oplus_mt6853](https://github.com/momo54181/android_kernel_oplus_mt6853) 感谢momo佬的帮助

[oppo_kernel_mt6885-mt6853-clean](https://github.com/Winkmoon/oppo_kernel_mt6885-mt6853-clean) 感谢xinran佬的帮助嘻嘻嘻嘻嘻
### 感谢Linux内核社区 正是因为你们开源社区才有了今天的繁荣

<a href="https://github.com/sanba0519/OPPO-Realme_kernel_4.14_MT6853/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=sanba0519/OPPO-Realme_kernel_4.14_MT6853" />
</a>



