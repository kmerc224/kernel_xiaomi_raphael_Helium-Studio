# 方案 C: 绕过 netbpfload 检查

## 思路
在内核层面或 init 阶段绕过 `netbpfload` 对 `netd.o` 的 critical 标记，让系统在 BPF 加载失败时继续启动。

## 技术路线
1. **内核层面**: 修改 `kernel/bpf/syscall.c`，使 BPF 程序加载失败时返回成功 (高风险)
2. **init.rc 层面**: 通过 Magisk/KernelSU 模块，在 `netbpfload` 执行前注入 `setprop` 跳过 critical 检查
3. **内核 cmdline**: 添加 `androidboot.bpf_fallback=1` 让 netd 回退到 iptables 模式

## 关键风险

### 1. 网络功能完全不可用 (极高风险)
`netd.o` 被标记为 **critical** 是有原因的——Android 16 的网络栈 (netd) 依赖 BPF 程序做流量控制、防火墙、带宽限制。如果 `netd.o` 加载失败且被绕过：
- Wi-Fi 和数据连接可能完全无法建立
- 即使能连接，网络权限控制 (per-app firewall) 失效
- `iptables-restore` 可能因 SELinux 拒绝而失败 (MIUI vendor 的 SELinux policy 可能不兼容)

### 2. SELinux 兼容性 (中风险)
MIUI 12.5.6 vendor 的 SELinux policy 是为 Android 11 设计的。在 Android 16 的 init 环境中，`netd` 的 iptables 操作可能被 SELinux 拒绝，因为：
- 文件上下文 (file_contexts) 不匹配
- 进程域 (domain) 转换规则缺失
- `avc: denied` 导致 netd 操作失败

### 3. 系统稳定性 (中风险)
绕过 critical 服务可能导致：
- `netd` 持续崩溃重启
- `system_server` 等待 netd 超时
- 连锁反应导致其他系统服务不可用
- 随机 reboot

### 4. 安全风险 (低风险)
绕过 BPF 意味着失去了 Android 16 的网络层安全增强，包括：
- 每应用网络权限控制
- 网络流量统计失效
- DoH/DoT 加密 DNS 可能异常