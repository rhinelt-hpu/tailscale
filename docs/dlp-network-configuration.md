# DLP/严格网络环境下的 Tailscale 配置指南

## 概述

本文档说明在数据防泄漏(DLP)环境中如何配置 Tailscale，以最小化网络数据包生成，同时禁用 STUN/P2P 连接，仅使用 DERP 中继服务器。

## 问题分析

### 错误日志分析

当设置 `TS_DEBUG_ALWAYS_USE_DERP=1` 时，您观察到的错误日志：

```
netcheck: netcheck: UDP is blocked, trying HTTPS
netcheck: UDP is blocked, trying ICMP
```

**根本原因**（已修复）：

之前版本中，`TS_DEBUG_ALWAYS_USE_DERP` 禁用了用于点对点通信的 UDP 套接字，但 `netcheck`（网络检测）模块仍会尝试发送 STUN 探测包，导致：

1. STUN 探测超时（因为 UDP 套接字被替换为阻塞连接）
2. netcheck 检测到"UDP 被阻塞"，回退到 HTTPS 和 ICMP 探测
3. 产生上述日志消息

**修复后的行为**：

本次更新使 `TS_DEBUG_ALWAYS_USE_DERP` 与 `onlyTCP443` 模式行为一致：
- 跳过 STUN (UDP) 探测
- 跳过 ICMP 探测  
- 仅使用 HTTPS (TCP 443) 测量 DERP 服务器延迟
- 消除 "UDP is blocked" 日志消息

### 修复后是否还会产生数据包？

**是的，但数量大幅减少且更加可控。**

修复后的数据包生成分析：

| 探测类型 | 修复前 | 修复后 | 说明 |
|---------|--------|--------|------|
| STUN (UDP) | 尝试发送但超时 | 完全跳过 | 不产生任何 UDP 包 |
| HTTPS | STUN 失败后发送 | 直接发送 | 每个 DERP 区域 1 个 TLS 连接 |
| ICMP | STUN 失败后发送 | 完全跳过 | 不产生任何 ICMP 包 |
| 端口映射 | 尝试探测 | 尝试探测 | 本地网关通信 |

**周期性流量（每 20-26 秒）**：
- HTTPS 延迟测量：约 10+ 个 TLS 握手 + HTTP 请求（到各 DERP 区域）
- DERP 心跳：每 60 秒一次 keep-alive（到 home DERP）

## 可用的环境变量配置

### 核心网络控制变量

```bash
# 强制所有流量通过 DERP（禁用 P2P UDP）
TS_DEBUG_ALWAYS_USE_DERP=1

# 禁用直接 UDP 连接（但允许 netcheck UDP 探测）
TS_DEBUG_NEVER_DIRECT_UDP=1
```

### Netcheck 相关变量

```bash
# 启用 netcheck 详细日志
TS_DEBUG_NETCHECK=1
```

### DERP 相关变量

```bash
# 启用详细 DERP 日志
TS_DEBUG_DERP=1

# 使用 HTTP 连接 DERP（而非 HTTPS）
TS_DEBUG_USE_DERP_HTTP=1

# 手动指定 DERP 地址
TS_DEBUG_USE_DERP_ADDR=<address>
```

### 其他相关变量

```bash
# 空闲时停止周期性 STUN（移动端默认启用）
TS_DEBUG_RESTUN_STOP_ON_IDLE=1

# 禁用端口列表监控
TS_DEBUG_DISABLE_PORTLIST=1

# 禁用看门狗
TS_DEBUG_DISABLE_WATCHDOG=1
```

## DLP 环境推荐配置

### 方案一：使用 TS_DEBUG_ALWAYS_USE_DERP（推荐）

```bash
# Linux systemd service
Environment="TS_DEBUG_ALWAYS_USE_DERP=1"
```

**效果**（修复后）：
- ✅ 禁用点对点 UDP 连接
- ✅ 跳过 STUN (UDP) 探测
- ✅ 跳过 ICMP 探测
- ✅ 仅使用 HTTPS 测量 DERP 延迟
- ✅ 所有实际数据流量通过 DERP (TCP 443)
- ✅ 不再产生 "UDP is blocked" 日志消息

### 方案二：使用 tailscale set --onlytcp443

如果您的环境只允许 TCP 443 流量：

```bash
tailscale set --onlytcp443
```

或在环境中设置控制平面响应以启用此功能。

**效果**：
- ✅ 完全禁用 UDP STUN 探测
- ✅ 完全禁用 ICMP 探测
- ✅ 仅使用 TCP 443 进行 DERP 延迟测量
- ✅ 所有流量通过 DERP (TCP 443)

### 方案三：自定义 DERP + 禁用 netcheck 外部网络

对于完全隔离的环境，可以：

1. 部署私有 DERP 服务器
2. 配置 DERP Map 仅包含私有服务器
3. 禁用到公共 DERP 的连接

## 代码原理解释

### TS_DEBUG_ALWAYS_USE_DERP 的工作原理

在 `wgengine/magicsock/magicsock.go` 中：

```go
func (c *Conn) bindSocket(ruc *RebindingUDPConn, network string, curPortFate portFate) error {
    // ...
    if debugAlwaysDERP() {
        c.logf("disabled %v per TS_DEBUG_ALWAYS_USE_DERP", network)
        ruc.setConnLocked(newBlockForeverConn(), "", c.bind.BatchSize())
        return nil
    }
    // ...
}
```

这段代码将 UDP 套接字替换为一个永远阻塞的连接，禁用点对点 UDP 通信。

### 修复：netcheck 也尊重 TS_DEBUG_ALWAYS_USE_DERP

修复后，`sendUDPNetcheck` 函数也检查 `debugAlwaysDERP()`：

```go
func (c *Conn) sendUDPNetcheck(b []byte, addr netip.AddrPort) (int, error) {
    // 修复：当 TS_DEBUG_ALWAYS_USE_DERP 设置时，也跳过 UDP netcheck
    if c.onlyTCP443.Load() || runtime.GOOS == "js" || debugAlwaysDERP() {
        return 0, errors.ErrUnsupported
    }
    // ...
}
```

同时，`updateNetInfo` 函数在调用 `GetReport` 时也传递 `OnlyTCP443: true`：

```go
report, err := c.netChecker.GetReport(ctx, dm, &netcheck.GetReportOpts{
    GetLastDERPActivity: c.health.GetDERPRegionReceivedTime,
    // 当 TS_DEBUG_ALWAYS_USE_DERP 设置时，跳过 STUN 探测，仅使用 TCP 443
    OnlyTCP443: c.onlyTCP443.Load() || debugAlwaysDERP(),
})
```

这确保了：
1. netcheck 不会创建 STUN 探测计划
2. netcheck 不会尝试 ICMP 探测
3. 仅使用 HTTPS 测量 DERP 延迟

### onlyTCP443 vs TS_DEBUG_ALWAYS_USE_DERP 的区别

| 特性 | onlyTCP443 | TS_DEBUG_ALWAYS_USE_DERP（修复后）|
|------|------------|-----------------------------------|
| 设置方式 | 通过 `tailscale set` 或控制平面 | 环境变量 |
| 禁用 P2P UDP | ✅ | ✅ |
| 禁用 STUN 探测 | ✅ | ✅ |
| 禁用 ICMP 探测 | ✅ | ✅ |
| netcheck 使用 | 仅 HTTPS | 仅 HTTPS |
| 数据流量 | DERP only | DERP only |
| 行为差异 | 无 | 等效于 onlyTCP443 |

## 网络数据包详细分析

### 周期性 netcheck 产生的流量

**每次 netcheck 运行（约 20-26 秒间隔）**：

**修复后，当 TS_DEBUG_ALWAYS_USE_DERP=1 或使用 onlyTCP443**：
- STUN 请求：完全跳过，不发送
- ICMP 探测：完全跳过，不发送
- HTTPS 探测：每个 DERP 区域 1 个 TLS 握手 + HTTP 请求

### DERP 连接流量

无论哪种配置，与 DERP 服务器的持久连接都会产生：

- **TCP 连接建立**：每个活跃的 DERP 区域 1 个 TCP 连接
- **TLS 握手**：每个连接的初始 TLS 协商
- **心跳包**：约每 60 秒一次 keep-alive

## 安全建议

### DLP 策略配置

1. **允许列表**：
   - `*.tailscale.com:443` - DERP 服务器
   - `controlplane.tailscale.com:443` - 控制平面

2. **阻止列表**：
   - UDP 3478 (STUN) - 如果使用 onlyTCP443
   - 所有其他 UDP 出站流量

### 审计日志

启用详细日志以监控网络活动：

```bash
TS_DEBUG_DERP=1
TS_DEBUG_NETCHECK=1
```

## 常见问题解答

### Q: 设置 TS_DEBUG_ALWAYS_USE_DERP 后还会看到 "UDP is blocked" 消息吗？

A: **不会**。修复后，`TS_DEBUG_ALWAYS_USE_DERP` 会使 netcheck 跳过 STUN 和 ICMP 探测，直接使用 HTTPS 测量延迟，因此不会产生这些日志消息。

### Q: 如何完全禁用 netcheck？

A: 目前没有官方支持的方式完全禁用 netcheck，因为它是选择最佳 DERP 服务器所必需的。但您可以：
1. 使用 `--force-derp=<region>` 强制指定 DERP 区域
2. 配置私有 DERP 服务器并限制 DERP Map
3. 设置 `TS_DEBUG_ALWAYS_USE_DERP=1` 后，netcheck 仅产生 HTTPS 流量

### Q: 这些探测包会泄露敏感信息吗？

A: 探测包本身不包含用户数据，仅用于测量网络延迟。HTTPS 探测会暴露：
- 您的 IP 地址（到 DERP 服务器）
- Tailscale 客户端的存在
- TLS ClientHello 中的 SNI

### Q: TS_DEBUG_ALWAYS_USE_DERP 和 --onlytcp443 有什么区别？

A: 修复后，两者在 netcheck 行为上是等效的：
- 都跳过 STUN (UDP) 探测
- 都跳过 ICMP 探测
- 都仅使用 HTTPS 测量 DERP 延迟
- 都强制所有流量通过 DERP

主要区别在于设置方式：`TS_DEBUG_ALWAYS_USE_DERP` 是环境变量，而 `--onlytcp443` 是通过 tailscale CLI 或控制平面设置。

## 总结

对于 DLP 环境，推荐配置：

**首选方案**：设置环境变量 `TS_DEBUG_ALWAYS_USE_DERP=1`

这将：
- ✅ 禁用点对点 UDP 连接
- ✅ 跳过 STUN (UDP) 网络探测
- ✅ 跳过 ICMP 网络探测
- ✅ 仅使用 HTTPS (TCP 443) 测量 DERP 延迟
- ✅ 所有数据流量通过 DERP 中继

**替代方案**：使用 `tailscale set --onlytcp443`（效果相同）

所有方案都确保实际数据流量仅通过 DERP 中继，不建立点对点连接。

---

*文档版本：基于 Tailscale 源代码分析*
*最后更新：2024*
*注：包含 TS_DEBUG_ALWAYS_USE_DERP netcheck 行为修复*
