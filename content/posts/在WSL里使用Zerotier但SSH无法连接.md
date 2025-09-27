---
title: "在WSL里使用Zerotier但SSH无法连接SSH2_MSG_KEX_ECDH_REPLY"
date: 2025-09-27
---
# Bug 调查文档：WSL 内 ZeroTier SSH 连接卡住问题

## 问题描述
在两台 Windows 机器 A 和 B 上，都安装了 WSL 和 ZeroTier(A在windows和wsl都安装了zerotier, B只在wsl里安装了zerotier) 的情况下，出现以下问题(均以ssh连接，地址使用zerotier的vpn ip)：

- **A 的 Windows → B 的 WSL**：可以正常连接。
- **A 的 WSL → B 的 WSL**：SSH 连接卡住。
- **A 的 WSL → 其他处于VPN的机器**：ZeroTier 可以正常连接。

具体表现为在 SSH 调试日志中卡在：
```
debug1: expecting SSH2_MSG_KEX_ECDH_REPLY
```

## 排查过程

1. **分析 SSH debug 日志**
   - 客户端版本：OpenSSH_9.6p1
   - 服务端版本：OpenSSH_8.2p1
   - 握手开始正常，算法协商成功：
     ```
     kex: algorithm: curve25519-sha256
     kex: host key algorithm: ssh-ed25519
     ```
   - 卡在等待 `SSH2_MSG_KEX_ECDH_REPLY`，未收到服务端回应。

2. **分析 ZeroTier 网络情况**
   - A 的 WSL 可以通过 ZeroTier 连接其他机器，说明 WSL 内 ZeroTier 正常。
   - B 的 WSL 可以被 A 的 Windows 访问，说明 B 的 WSL ZeroTier 正常。
   - 问题只发生在 **A 的 WSL → B 的 WSL**，提示可能是网络层（路由/NAT/MTU）问题。

3. **推测原因**
   - WSL2 使用 Hyper-V NAT，ZeroTier 虚拟网卡通过 NAT，可能导致大数据包（如 SSH 握手的 KEX ECDH 消息）丢失或被截断。
   - 这与之前可以连通但后来不通的现象一致，属于网络条件微调导致的问题。

4. **排查方法**
   - 在 A 和 B 的 WSL 内检查 ZeroTier 网卡：
     ```bash
     ip link show
     ```
   - 测试对方的连通性（重要，实测，增大到1100时无法ping通）：
     ```bash
     ping -s 900 <对方 ZeroTier IP>
     ```
   - 观察 SSH 服务端日志：
     ```bash
     sudo /usr/sbin/sshd -D -d
     ```
     确认服务端是否收到 KEX 请求并尝试回应。

---

## 原因分析
- 问题主要由 **WSL2 + ZeroTier NAT 网络 MTU 不匹配** 引起。
- SSH 握手阶段发送的数据包过大，超过 WSL/ZeroTier 默认 MTU，导致包丢失，客户端卡在等待 ECDH_REPLY。
- Windows 客户端能够连接成功是因为 Windows ZeroTier 使用宿主机网络栈，不受 WSL2 NAT 限制。

---

## 解决方法
1. **调整 MTU**
   - 在 A 和 B 的 WSL 内，找到 ZeroTier 虚拟网卡名称 `ztXXXX`：
     ```bash
     ip link show
     ```
   - 修改 MTU 为较小值（测试可用值为 1000）：
     ```bash
     sudo ip link set dev ztXXXX mtu 1000
     ```
   - 验证：
     ```bash
     ping -s 900 <对方 ZeroTier IP>
     ssh <对方 ZeroTier IP>
     ```

2. **说明**
   - 调整 MTU 后 SSH 连接恢复正常。
   - 过小 MTU 会增加分片开销，但对 SSH 小流量影响不大。
   - 建议找到 **最大稳定 MTU**，可以用 `ping -s` 逐步测试包大小。

---

## 结论
- WSL2 内 ZeroTier 的 SSH 连接偶尔出现卡住，是 **MTU 导致的大包丢失**。
- 解决方案是 **在双方 WSL 内减小 ZeroTier 虚拟网卡 MTU**。
- 现象出现与之前能够连通但后来不通一致，属于网络微调引发的暂发性问题。
