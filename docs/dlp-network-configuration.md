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

### 方案二：通过控制平面配置 NodeAttrOnlyTCP443（推荐用于单节点配置）

如果您希望大多数节点正常工作，仅让特定节点使用 TLS-only 模式（禁用 STUN/ICMP），这是最佳方案。

#### 对于官方 Tailscale 用户

1. 登录 [Tailscale Admin Console](https://login.tailscale.com/admin)
2. 在 ACL 配置中为特定节点或用户添加 `only-tcp-443` 能力

#### 对于 Headscale 用户（自建控制面板）

在 Headscale 的 ACL 策略文件中，为特定节点添加 `only-tcp-443` 能力：

```json
{
  "nodeAttrs": [
    {
      "target": ["user1@example.com"],
      "attr": ["only-tcp-443"]
    },
    {
      "target": ["tag:dlp-restricted"],
      "attr": ["only-tcp-443"]
    }
  ]
}
```

**配置方式**：
- `target`: 可以是用户邮箱、标签（如 `tag:dlp-restricted`）
- `attr`: 设置为 `["only-tcp-443"]`

**注意**：Headscale 的 ACL 策略通过 JSON 文件配置（通常位于 `/etc/headscale/acl.json`），需要在 `config.yaml` 中设置 `policy.path` 指向该文件。

**效果**（仅对配置了该属性的节点生效）：
- ✅ 完全禁用 UDP STUN 探测
- ✅ 完全禁用 ICMP 探测
- ✅ 仅使用 TCP/HTTPS 进行 DERP 延迟测量
- ✅ 所有流量通过 DERP
- ✅ 其他节点正常使用 P2P/STUN/ICMP

**优势**：
- 无需在客户端设置环境变量
- 可以精确控制哪些节点需要 TLS-only 模式
- 其他节点不受影响，可以正常使用直连和 NAT 穿透

### 方案三：自定义 DERP + 禁用 netcheck 外部网络

对于完全隔离的环境，可以：

1. 部署私有 DERP 服务器
2. 配置 DERP Map 仅包含私有服务器
3. 禁用到公共 DERP 的连接

### 方案四：自建控制面板和 DERP 服务器（非标准端口）

如果您使用自建的控制面板和 DERP 服务器，且使用非标准端口（如控制面板 7081，DERP 7080/7478），需要特别配置。

#### DERP 服务器端口说明

| 端口 | 用途 | 协议 | 说明 |
|------|------|------|------|
| 443（默认） | DERP HTTPS | TCP | 主要数据中继端口 |
| 3478（默认） | STUN | UDP | NAT 穿透探测端口 |
| 80（可选） | DERP HTTP | TCP | 备用端口（需显式启用） |

#### 自定义 DERP 端口配置

在 DERP Map 配置中，可以为每个节点设置自定义端口：

```json
{
  "Regions": {
    "900": {
      "RegionID": 900,
      "RegionCode": "custom",
      "RegionName": "Custom DERP",
      "Nodes": [{
        "Name": "900a",
        "RegionID": 900,
        "HostName": "your-derp.example.com",
        "DERPPort": 7080,
        "STUNPort": -1
      }]
    }
  },
  "OmitDefaultRegions": true
}
```

**关键配置项**：
- `DERPPort`: 自定义 DERP HTTPS 端口（默认 443，您可设置为 7080）
- `STUNPort`: 设置为 `-1` 可完全禁用该节点的 STUN 探测
- `OmitDefaultRegions`: 设置为 `true` 可禁用公共 DERP 区域

#### 禁用 STUN 的重要性

当 `STUNPort` 设置为 `-1` 时：
- 该 DERP 节点不会接收 STUN 探测请求
- netcheck 不会尝试向该节点发送 UDP STUN 包
- 延迟测量仅通过 HTTPS 进行

这对于 DLP 环境非常重要，因为它可以确保：
1. 不产生任何 UDP 外发流量
2. 所有探测流量都通过 TCP（可被防火墙更精确控制）
3. 减少被 DLP 系统误报的风险

#### 完整配置示例

假设您的环境：
- 自建控制面板：`headscale.example.com:7081`
- 自建 DERP：`derp.example.com:7080`（HTTPS）
- 不需要 STUN：端口 7478 可以关闭

**客户端环境变量配置**：
```bash
# 强制使用 DERP（禁用 P2P UDP）
TS_DEBUG_ALWAYS_USE_DERP=1

# 如果需要额外的调试信息
TS_DEBUG_DERP=1
TS_DEBUG_NETCHECK=1
```

**DERP Map 配置**（通过控制面板下发）：
```json
{
  "Regions": {
    "900": {
      "RegionID": 900,
      "RegionCode": "private",
      "RegionName": "Private DERP",
      "Nodes": [{
        "Name": "900a",
        "RegionID": 900,
        "HostName": "derp.example.com",
        "DERPPort": 7080,
        "STUNPort": -1
      }]
    }
  },
  "OmitDefaultRegions": true
}
```

**效果**：
- ✅ 所有流量通过 `derp.example.com:7080` (TCP)
- ✅ 无 UDP STUN 探测（端口 7478 可关闭）
- ✅ 无 ICMP 探测
- ✅ 延迟测量仅通过 HTTPS 到端口 7080
- ✅ 不连接任何公共 DERP 服务器

#### 注意：关于 onlyTCP443 功能与自定义端口

`onlyTCP443` 功能的命名可能造成误解（此命名是为了向后兼容）。实际上，当启用此功能或 `TS_DEBUG_ALWAYS_USE_DERP=1` 时：

1. **行为**：跳过 STUN (UDP) 和 ICMP 探测，仅使用 HTTPS 测量延迟
2. **端口**：HTTPS 探测会连接到 DERP 节点配置的 `DERPPort`，不一定是 443
3. **兼容性**：完全兼容自定义 DERP 端口（如 7080）

因此，即使您的 DERP 服务器运行在 7080 端口，`onlyTCP443` 模式仍然有效，它会通过 TCP 连接到 7080 端口进行 HTTPS 延迟测量，而不是强制使用 443 端口。

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

### Q: TS_DEBUG_ALWAYS_USE_DERP 和 NodeAttrOnlyTCP443 有什么区别？

A: 修复后，两者在 netcheck 行为上是等效的：
- 都跳过 STUN (UDP) 探测
- 都跳过 ICMP 探测
- 都仅使用 HTTPS 测量 DERP 延迟
- 都强制所有流量通过 DERP

主要区别在于设置方式：
- `TS_DEBUG_ALWAYS_USE_DERP`：环境变量，客户端可直接设置
- `NodeAttrOnlyTCP443`：需要通过控制平面（admin console）配置，客户端无法直接设置

### Q: 我使用自建 DERP 在非 443 端口，onlyTCP443 还能用吗？

A: **可以**。`onlyTCP443` 的名称具有误导性，它实际上意味着"仅使用 TCP/HTTPS 进行延迟测量，跳过 UDP 和 ICMP"。当启用此模式时：

1. HTTPS 延迟测量会连接到每个 DERP 节点配置的 `DERPPort`（可以是任何端口，如 7080）
2. 跳过所有 UDP STUN 探测（不管 `STUNPort` 配置为什么）
3. 跳过所有 ICMP 探测

因此，如果您的 DERP 服务器配置为 `DERPPort: 7080`，延迟测量会通过 TCP 连接到 7080 端口。

### Q: 如何完全禁止 netcheck 产生的网络流量？

A: 完全禁用 netcheck 不推荐，因为它用于选择最佳 DERP 服务器。但您可以最小化流量：

1. **设置 `TS_DEBUG_ALWAYS_USE_DERP=1`**：跳过 UDP/ICMP，仅使用 HTTPS
2. **在 DERP Map 中设置 `STUNPort: -1`**：确保不发送 STUN 请求
3. **只配置一个 DERP 区域**：减少 HTTPS 探测数量
4. **设置 `OmitDefaultRegions: true`**：不连接公共 DERP

这样，周期性流量仅为：
- 到您私有 DERP 的 HTTPS 延迟测量（每 20-26 秒一次 TLS 握手）
- DERP 连接心跳（每 60 秒一次）

### Q: "UDP is blocked" 日志会导致大量发包吗？

A: **分析原因**：

当看到这些日志时，说明 netcheck 执行了以下流程：
1. 尝试发送 STUN 探测（UDP）→ 超时或被阻塞
2. 等待超时（默认约 3 秒）
3. 回退到 HTTPS 探测
4. 尝试发送 ICMP 探测 → 通常也失败
5. 记录日志 "UDP is blocked, trying HTTPS/ICMP"

**发包量分析**：

在未修复的情况下，每次 netcheck（约每 20-26 秒）会产生：
- STUN 请求：每个 DERP 区域多个 UDP 包（尝试发送但失败）
- ICMP 请求：每个 DERP 区域 1 个 ICMP 包（尝试发送）
- HTTPS 请求：每个 DERP 区域 1 个 TLS 连接

**修复后**（设置 `TS_DEBUG_ALWAYS_USE_DERP=1`）：
- STUN 请求：完全不发送
- ICMP 请求：完全不发送
- HTTPS 请求：每个 DERP 区域 1 个 TLS 连接
- 无 "UDP is blocked" 日志

## 总结

对于 DLP 环境，推荐配置：

### 场景一：仅特定节点需要 TLS-only（推荐）

如果您使用 Headscale 且只想让某些节点禁用 STUN/ICMP，其他节点正常使用：

**在 Headscale ACL 中配置**：
```json
{
  "nodeAttrs": [
    {
      "target": ["tag:dlp-node"],
      "attr": ["only-tcp-443"]
    }
  ]
}
```

然后给需要 TLS-only 的节点添加 `dlp-node` 标签。

**效果**：
- ✅ 带 `dlp-node` 标签的节点：禁用 STUN/ICMP，仅 TLS
- ✅ 其他节点：正常使用 P2P、STUN、ICMP
- ✅ 无需修改客户端配置

### 场景二：所有节点都需要 TLS-only

**首选方案**：设置环境变量 `TS_DEBUG_ALWAYS_USE_DERP=1`

这将：
- ✅ 禁用点对点 UDP 连接
- ✅ 跳过 STUN (UDP) 网络探测
- ✅ 跳过 ICMP 网络探测
- ✅ 仅使用 HTTPS (TCP 443) 测量 DERP 延迟
- ✅ 所有数据流量通过 DERP 中继

### 场景三：自建 DERP 使用非标准端口

**推荐配置组合**：

1. **控制面板 ACL**（针对特定节点）：
   ```json
   {
     "nodeAttrs": [
       {
         "target": ["tag:dlp-restricted"],
         "attr": ["only-tcp-443"]
       }
     ]
   }
   ```

2. **DERP Map**（通过控制面板下发）：
   
   **如果所有节点都禁用 STUN**：
   ```json
   {
     "Regions": {
       "900": {
         "RegionID": 900,
         "Nodes": [{
           "HostName": "your-derp.example.com",
           "DERPPort": 7080,
           "STUNPort": -1
         }]
       }
     },
     "OmitDefaultRegions": true
   }
   ```

   **如果部分节点需要正常使用 STUN**（设置有效的 STUN 端口）：
   ```json
   {
     "Regions": {
       "900": {
         "RegionID": 900,
         "Nodes": [{
           "HostName": "your-derp.example.com",
           "DERPPort": 7080,
           "STUNPort": 7478
         }]
       }
     },
     "OmitDefaultRegions": true
   }
   ```

**注意**：`STUNPort` 是 DERP 服务器级别的配置，影响所有节点。如果设置为 `-1`，所有节点都无法使用 STUN。`only-tcp-443` 节点属性仅控制客户端是否尝试发送 STUN/ICMP 请求。

这将：
- ✅ 所有流量通过您的私有 DERP（自定义端口如 7080）
- ✅ 带 `dlp-restricted` 标签的节点：禁用 UDP/ICMP 探测
- ✅ 其他节点：根据 DERP 的 STUNPort 配置决定是否可用 STUN
- ✅ 不连接公共 Tailscale 服务器
- ✅ 配置了 `only-tcp-443` 的节点不产生 "UDP is blocked" 日志

所有方案都确保配置了 TLS-only 属性的节点数据流量仅通过 DERP 中继，不建立点对点连接。

---

*文档版本：基于 Tailscale 源代码分析*
*最后更新：2024*
*注：包含 TS_DEBUG_ALWAYS_USE_DERP netcheck 行为修复*
