# 方案 D: System BPF Patch（系统端 BPF 修补）

来源: [酷安 @tiffanyy](https://www.coolapk.com/feed/71090978)

## 核心原理

Android 16 删除了对 5.4 以下内核的支持。`com.android.tethering` APEX 中的 `netbpfload` 加载 `netd.o` BPF 程序时，需要 4.17–5.10 内核才有的 BPF cgroup hook，4.14 内核缺少这些 hook，导致 `netbpfload` 标记 `netd.o` 为 **critical** 失败 → 系统重启。

**此方案的思路**：不修改内核，而是替换 tethering APEX 中的关键组件，让它们兼容低版本内核。

## Patch 文件分析

### 替换清单（共 7 个文件）

| 文件 | 大小 | 路径 | 作用 |
|---|---|---|---|
| `netbpfload` | 81K | `payload/bin/` | BPF 程序加载器（核心） |
| `framework-connectivity.jar` | 1.9M | `payload/javalib/` | 连接框架 Java 层 |
| `service-connectivity.jar` | 2.1M | `payload/javalib/` | 连接服务 Java 层 |
| `libnetd_updatable.so` | 53K | `payload/lib64/` | netd 原生库 |
| `libnetworkstats.so` | 514K | `payload/lib64/` | 网络统计原生库 |
| `libservice-connectivity.so` | 64K | `payload/lib64/` | 连接服务原生库 |
| `libcom.android.tethering.dns_helper.so` | 40K | `payload/lib64/` | DNS 辅助库 |

### netbpfload 关键改动

从二进制中提取的字符串揭示了具体修改：

```
# 1. 内核版本覆盖（通过系统属性）
ro.bpf.kver_override          ← 可以伪造内核版本

# 2. 跳过需要高版本内核的 BPF Map
skipping map %s which requires kernel version 0x%x >= 0x%x
skipping map %s which requires kernel version 0x%x < 0x%x
skipping LPM_TRIE map %s - requires kver 4.14+

# 3. 将 critical 程序降级为 optional
failed program %s is marked optional - continuing...

# 4. 保留的版本检查（但不阻塞启动）
Android 25Q2 requires kernel 5.4.
Android U requires kernel 4.14.
Android V requires kernel 4.19.
```

**核心改动**：原始 AOSP `netbpfload` 中 `netd.o` 被标记为 critical，加载失败 → `abort()` → 系统重启。patch 版本将其改为：
- 跳过不兼容的 BPF map
- 将失败的 critical 程序当作 optional 处理
- 支持 `ro.bpf.kver_override` 属性覆盖内核版本检测

## 已测试设备

经原作者验证，以下设备 HyperOS3 A16 均成功开机：
- Redmi 9 (MTK)
- Redmi Note 10 (高通)
- Redmi K40 Gaming (MTK)

## 操作步骤（在你的 K20 Pro 上）

### 1. 准备环境

```bash
# 安装依赖
sudo apt-get install python3 python3-pip
pip3 install protobuf==3.17.3

# 下载工具
git clone https://github.com/Meetingf/AndroidApexTools
cd AndroidApexTools
chmod -R a+x .
```

### 2. 提取 tethering APEX

```bash
adb shell
su
# 检查是 capex 还是 apex
ls /system/apex/com.android.tethering*
cp /system/apex/com.android.tethering.capex /sdcard/
exit
exit

adb pull /sdcard/com.android.tethering.capex .
```

### 3. 解包 → 替换 → 重打包

```bash
# 解包
python3 ./deapexer.py extract ./com.android.tethering.capex

# 替换（把 patch 文件覆盖到 payload/ 目录）
cp -r /path/to/bpf-patch/bin/* payload/bin/
cp -r /path/to/bpf-patch/javalib/* payload/javalib/
cp -r /path/to/bpf-patch/lib64/* payload/lib64/

# 重新打包（与原格式一致）
python3 ./apexer.py --api 36 --compress ./com.android.tethering_new.capex
```

### 4. 刷入替换

```bash
adb push com.android.tethering_new.capex /sdcard/
adb shell
su
mount -o rw,remount /system
cp /sdcard/com.android.tethering_new.capex /system/apex/com.android.tethering.capex
chmod 644 /system/apex/com.android.tethering.capex
reboot
```

### 5. 可选：设置内核版本覆盖

```bash
# 如果依然失败，尝试伪造内核版本
adb shell
su
setprop ro.bpf.kver_override 1
# 或
resetprop ro.bpf.kver_override 1
```

## 风险分析

### 1. 网络功能受限（中风险）
`netbpfload` 跳过了一些 BPF map 和程序，可能导致：
- 每应用流量统计不准确
- 部分网络策略失效
- 热点 (tethering) 功能异常（但原作者测试通过）

### 2. 与 MIUI vendor 的兼容性（中风险）
虽然 patch 在 Redmi 9/Note 10/K40 Gaming 上测试通过，但 raphael 的 MIUI 12.5.6 vendor 更老：
- netd 的 iptables 操作可能被 SELinux 拒绝
- MIUI 特有的网络管理守护进程可能冲突

### 3. OTA 后失效（低风险）
系统更新会覆盖 `system/apex`，需要重新 patch。

### 4. 安全性（低风险）
绕过 BPF 意味着失去了 Android 16 的网络层安全增强，但对于非 GKI 旧设备来说这是可接受的折衷。

## 与其他方案对比

| 维度 | 方案 D | 方案 A | 方案 B | 方案 C |
|---|---|---|---|---|
| 修改层 | APEX | 内核 | 内核 | 内核+init |
| 内核修改 | 不需要 | 大量 | 少量 | 少量 |
| 驱动兼容 | 无影响 | 高难度 | 高难度 | 高难度 |
| BPF 完整性 | 降级 | 完整 | 部分 | 降级 |
| 难度 | 最低 | 最高 | 高 | 中 |
| 已验证 | 是 | 否 | 否 | 否 |

## 推荐执行顺序

```
1. 立即操作方案 D → patch tethering APEX → 刷入 Helium 内核
2. 如果方案 D 成功 → 持续使用，后续考虑方案 A 做持久化
3. 如果方案 D 失败 → 说明 Helium 内核在 MIUI vendor 上有更深层问题
```