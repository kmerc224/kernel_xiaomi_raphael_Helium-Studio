# 方案 D: System BPF Patch (系统端 BPF 修补)

来源: [酷安 @tiffanyy](https://www.coolapk.com/feed/71090978)

## 核心原理

**Android 16 已删除对 5.4 以下内核的支持**。`com.android.tethering` APEX 中的 `netbpfload` 加载 `netd.o` BPF 程序时，需要 4.17–5.10 内核才有的 BPF cgroup hook（`bind4/6`、`connect4/6`、`sendmsg4/6`、`recvmsg4/6` 等），4.14 内核缺少这些 hook，导致 `netbpfload` 失败 → 系统重启。

**此方案不从内核层面解决，而是修改系统 APEX 包**，将 tethering 中的 BPF 程序替换为兼容 4.14 内核的版本。

## 已测试设备

经原作者验证，以下设备 HyperOS3 A16 均成功开机：
- Redmi 9 (MTK)
- Redmi Note 10 (高通)
- Redmi K40 Gaming (MTK)

## 操作步骤

### 1. 准备工具
- 下载 [AndroidApexTools](https://github.com/Meetingf/AndroidApexTools)
- 下载 BPF patch 文件 (原作者提供的 `system_bpf-patch_A16.zip`)
- 需要 root 权限 (Magisk/KernelSU)

### 2. 提取 tethering APEX
```bash
adb shell
su
cp /system/apex/com.android.tethering.capex /sdcard/
# 或
cp /system/apex/com.android.tethering.apex /sdcard/
```

### 3. 解包 APEX
```bash
cd AndroidApexTools
python3 ./deapexer.py extract ./com.android.tethering.capex
# 生成 manifest/ 和 payload/ 文件夹
```

### 4. 替换 BPF 文件
```bash
# 将 patch 压缩包中的文件覆盖到 payload/ 目录
# 具体替换了哪些文件需要查看原始 patch 包内容
```

### 5. 重新打包
```bash
# 无压缩 APEX
python3 ./apexer.py --api 36 ./com.android.tethering.apex
# 或压缩 APEX (与原格式一致)
python3 ./apexer.py --api 36 --compress ./com.android.tethering.capex
```

### 6. 替换系统文件
```bash
adb push com.android.tethering.capex /sdcard/
adb shell
su
mount -o rw,remount /system
cp /sdcard/com.android.tethering.capex /system/apex/
chmod 644 /system/apex/com.android.tethering.capex
reboot
```

## 关键风险

### 1. 网络功能降级 (中风险)
方案本质是**移除/降级 BPF 网络控制**。Android 16 的 netd 依赖 BPF 做：
- 流量控制 (bandwidth limiting)
- 每应用防火墙 (per-app firewall)
- 网络权限管理

如果 BPF 程序被替换为兼容版本：
- 可能丢失部分网络策略功能
- 热点 (tethering) 功能可能异常
- VPN 可能受影响

### 2. patch 文件来源不明 (中风险)
原作者提供的 `system_bpf-patch_A16.zip` 来自 123 网盘，需要付费下载。不清楚具体替换了哪些文件，是否有后门。

### 3. OTA 后被覆盖 (低风险)
系统更新会替换 `system/apex` 下的文件，需要重新 patch。

### 4. 与 MIUI vendor 的兼容性 (中风险)
虽然此方案在 Redmi 设备上测试通过，但 raphael (K20 Pro) 的 MIUI 12.5.6 vendor 可能有额外兼容性问题。

## 与方案 A/B/C 的对比

| 维度 | 方案 D (System BPF) | 方案 A (内核兼容) | 方案 B (BPF 版本覆盖) |
|---|---|---|---|
| 修改层 | 系统 (APEX) | 内核源码 | 内核源码 |
| 难度 | 低 | 最高 | 高 |
| 内核修改 | 不需要 | 大量 | 少量 |
| 网络功能 | 可能降级 | 完整 | 完整 |
| OTA 持久性 | 会被覆盖 | 持久 | 持久 |
| 风险 | 中 | 高 | 高 |

## 推荐路径

**先尝试方案 D**，因为它：
1. 不需要修改内核 → 不存在驱动兼容性问题
2. 已在多个 HyperOS3 A16 设备上验证
3. 操作简单，即使失败也可以恢复

如果方案 D 成功，再考虑将修改整合到方案 A 中做持久化。