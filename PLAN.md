# 方案 A: MIUI 兼容性修复 + BPF 回移植

## 思路
在 Helium 内核 (AOSP/openela 谱系) 基础上，修复 MIUI vendor 兼容性，使其能在 HyperOS 3 上启动。

## 技术路线
1. 关闭 `CONFIG_BUILD_ARM64_DT_OVERLAY`，使 appended DTB 自包含，不依赖 bootloader 的 dtbo overlay
2. 对比 MIUI 原厂内核的 defconfig，补充 MIUI vendor HAL 所需的内核配置
3. 检查 MIUI 框架依赖的 sysfs/procfs 接口是否缺失
4. Helium 已有完整 BPF 回移植 (openela 来源)，启动后应能通过 netbpfload

## 关键风险

### 1. 驱动版本不匹配 (高风险)
HyperOS 3 的 vendor 分区是 MIUI 12.5.6 (Android 11)，其 HAL 层 (display, camera, audio, sensors) 与内核驱动通过特定 ioctl 和 sysfs 节点通信。AOSP 谱系内核的驱动可能：
- 使用了不同的 ioctl 命令码
- 缺少 MIUI 特有的 sysfs 节点 (如 `/sys/class/touchscreen/*` 的 xiaomi 定制属性)
- 显示面板初始化时序不同 → 黑屏/花屏

### 2. 电源管理 / 热管理冲突 (高风险)
MIUI 的 `thermal-engine` 和 `perfd` 通过内核接口控制 CPU 频率和热策略。AOSP 谱系内核的 thermal/power 驱动若与 MIUI 用户态守护进程不兼容，会导致：
- 设备过热自动关机
- CPU 频率锁死最低/最高
- 充电异常

### 3. 指纹 (FOD) 驱动 (中风险)
raphael 的光学屏下指纹驱动在 MIUI 和 AOSP 中有不同的用戶态接口。如果内核驱动接口不匹配，指纹将完全不可用。

### 4. DT_OVERLAY 关闭的风险 (低风险)
关闭 DT_OVERLAY 后，内核不再从 dtbo 分区加载 overlay。若某些硬件初始化依赖 dtbo 中的配置，可能导致设备树不完整。