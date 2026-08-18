# 方案 B: BPF 版本覆盖

## 思路
在 Helium 内核中伪造更高的 BPF 版本号，让 Android 16 的 `netbpfload` 认为内核支持所需 BPF 特性。

## 技术路线
1. 修改 `include/uapi/linux/bpf.h`，将 `LINUX_BPF_MAX_VER` 提升到 5.4 或 5.10
2. 参考 VoltageOS 的 BPF 版本覆盖 patch：`"Override kernel BPF version to 5.4.299"`
3. 参考 ProjectInfinity-X 的做法：`"Bump kernel BPF version override to 5.10"`
4. 可能需要额外回移植部分 BPF helper 函数和 map 类型

## 关键风险

### 1. 假版本 ≠ 真功能 (极高风险)
单纯改版本号**不能**让内核获得新的 BPF 能力。`netbpfload` 加载 `netd.o` 时会调用 `bpf()` syscall，如果内核缺少对应的 BPF helper 函数 (如 `bpf_bind`, `bpf_connect`, `bpf_sendmsg`, `bpf_recvmsg`, `bpf_getsockopt`, `bpf_setsockopt`)，syscall 会返回 `-EINVAL`，`netbpfload` 仍然会失败。

Helium openela 树**已有部分 BPF 回移植**，但可能不完整。需要逐项验证：
- `BPF_CGROUP_INET4_BIND` / `BPF_CGROUP_INET6_BIND`
- `BPF_CGROUP_INET4_CONNECT` / `BPF_CGROUP_INET6_CONNECT`
- `BPF_CGROUP_UDP4_SENDMSG` / `BPF_CGROUP_UDP6_SENDMSG`
- `BPF_CGROUP_UDP4_RECVMSG` / `BPF_CGROUP_UDP6_RECVMSG`
- `BPF_CGROUP_GETSOCKOPT` / `BPF_CGROUP_SETSOCKOPT`
- `BPF_CGROUP_INET_SOCK_RELEASE`

### 2. 内核与用户态 BPF 程序不匹配 (中风险)
`netd.o` 是预编译的 BPF 对象文件，它期望的 map 格式和 helper 函数版本必须与内核提供的完全一致。即使版本号覆盖了，如果内核缺少某个 map 类型或 helper 函数 ID，加载会失败。

### 3. 网络功能降级 (中风险)
即使 `netbpfload` 能加载 `netd.o`，如果某些 BPF helper 实际行为与预期不同，可能导致：
- Wi-Fi 能连接但无法上网
- 移动数据不可用
- 热点功能异常
- VPN 无法连接