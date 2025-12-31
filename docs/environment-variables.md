# Tailscale 环境变量完整参考指南

本文档详细列出了 Tailscale 客户端 (`tailscale` 和 `tailscaled`) 支持的所有环境变量，包括调试选项、网络控制和功能开关。

> **注意**：带有 `TS_DEBUG_` 前缀的变量主要用于调试和测试，可能会在未来版本中更改或移除。

## 目录

- [网络连接控制](#网络连接控制)
- [控制平面通信](#控制平面通信)
- [DERP 中继服务器](#derp-中继服务器)
- [端口映射和 NAT](#端口映射和-nat)
- [DNS 配置](#dns-配置)
- [SSH 功能](#ssh-功能)
- [日志和调试](#日志和调试)
- [TUN 设备](#tun-设备)
- [平台特定](#平台特定)
- [其他功能](#其他功能)
- [配置示例](#配置示例)

---

## 网络连接控制

### TS_DEBUG_ALWAYS_USE_DERP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:37`

**功能**：强制所有流量通过 DERP 中继服务器，完全禁用 UDP 直连。

**效果**：
- ✅ 禁用 UDP socket（替换为阻塞连接）
- ✅ 跳过 STUN 探测
- ✅ 跳过 ICMP 探测
- ✅ 所有数据通过 DERP (TCP/TLS) 传输

**使用场景**：
- DLP（数据防泄漏）环境
- 仅允许 TCP 443 出站的网络
- 需要完全控制流量的企业环境

**配置示例**：
```bash
# Linux (systemd)
Environment="TS_DEBUG_ALWAYS_USE_DERP=1"
```

---

### TS_DEBUG_NEVER_DIRECT_UDP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:67`

**功能**：禁用直接 UDP 连接，强制所有对等通信通过 DERP 或中继。

**与 `TS_DEBUG_ALWAYS_USE_DERP` 的区别**：
- `TS_DEBUG_ALWAYS_USE_DERP`：完全禁用 UDP socket
- `TS_DEBUG_NEVER_DIRECT_UDP`：仅禁用直接 UDP 对等连接

---

### TS_DEBUG_RESTUN_STOP_ON_IDLE

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:35`

**功能**：当连接空闲超过 45 秒时，停止周期性 STUN 检测。

**注意**：仅在设备实际空闲时生效。如果有持续的控制平面通信，周期性检测仍会运行。

---

### TS_DISABLE_PORTMAPPER

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/portmapper/portmapper.go:43`

**功能**：禁用自动端口映射功能（UPnP、NAT-PMP、PCP）。

**使用场景**：
- 不希望 Tailscale 修改路由器端口映射
- 网络环境不支持端口映射协议

---

### TS_DISABLE_UPNP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/portmapper/upnp.go:425`

**功能**：仅禁用 UPnP 端口映射，保留 NAT-PMP 和 PCP。

---

### TS_ENABLE_RAW_DISCO

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/magicsock_linux.go:44`  
**平台**: 仅 Linux

**功能**：启用原始 socket 的 disco（发现）协议。

---

### TS_DEBUG_RAW_DISCO

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/magicsock_linux.go:48`  
**平台**: 仅 Linux

**功能**：启用原始 disco 读取的调试日志。

---

### TS_DISCO_PONG_IPV4_DELAY

**类型**: Duration (时长)  
**默认值**: `0`  
**代码位置**: `wgengine/magicsock/magicsock.go:1925`

**功能**：为 IPv4 disco pong 添加人为延迟，用于测试 IPv6 优先级。

**格式**：Go 时长格式，如 `100ms`、`1s`

---

## 控制平面通信

### TS_FORCE_NOISE_443

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlhttp/client.go:181`

**功能**：强制控制平面连接使用 HTTPS (443 端口)，跳过 HTTP (80 端口) 尝试。

**默认行为**：
1. 先尝试 HTTP 80 端口
2. 失败后回退到 HTTPS 443 端口

**设置后行为**：
- 直接使用 HTTPS 443 端口
- 所有控制平面通信走 TLS 加密

**使用场景**：
- 80 端口被代理/中间人劫持
- Docker Desktop VPNKit 问题
- 需要全程 TLS 加密

**注意**：仅影响控制平面通信，不影响 DERP 数据传输。

---

### TS_USE_CONTROL_DIAL_PLAN

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlhttp/client.go:96`

**功能**：记录是否使用控制平面拨号计划。

---

### TS_DEBUG_NOISE_DIAL

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlhttp/client.go:223`

**功能**：启用 Noise 协议拨号的详细调试日志。

---

### TS_DEBUG_MAP

**类型**: Integer  
**默认值**: `0`  
**代码位置**: `control/controlclient/direct.go:1399`

**功能**：设置 map 响应的调试级别。

---

### TS_DEBUG_REGISTER

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlclient/direct.go:1403`

**功能**：启用注册过程的详细调试日志。

---

### TS_DEBUG_PROXY_DNS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlclient/direct.go:1404`

**功能**：强制代理 DNS 查询。

---

### TS_DEBUG_STRIP_ENDPOINTS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlclient/direct.go:1405`

**功能**：从 netmap 中移除端点信息（调试用）。

---

### TS_DEBUG_STRIP_HOME_DERP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlclient/direct.go:1406`

**功能**：从 netmap 中移除 home DERP 信息（调试用）。

---

### TS_DEBUG_STRIP_CAPS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlclient/direct.go:1407`

**功能**：从 netmap 中移除能力信息（调试用）。

---

### TS_DEBUG_CONTROL_FLAGS

**类型**: String  
**默认值**: `""`  
**代码位置**: `ipn/ipnlocal/local.go:113`

**功能**：设置额外的控制标志。

---

### TS_PANIC_IF_HIT_MAIN_CONTROL

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlclient/direct.go:350`

**功能**：如果连接到主控制服务器则触发 panic（用于测试自建服务器）。

---

## DERP 中继服务器

### TS_DEBUG_DERP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:30`

**功能**：记录所有收到的 DERP 数据包，包括完整负载（警告：会产生大量日志）。

---

### TS_DEBUG_USE_DERP_ADDR

**类型**: String  
**默认值**: `""`  
**代码位置**: `wgengine/magicsock/debugknobs.go:39`

**功能**：手动设置 DERP 地址，覆盖控制平面提供的 DERP map。

**格式**：`host:port`

---

### TS_DEBUG_USE_DERP_HTTP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:42` 和 `derp/derphttp/derphttp_client.go:250`

**功能**：使用 HTTP (3340 端口) 连接 DERP，而非 HTTPS (443 端口)。

**警告**：这会禁用 DERP 连接的加密，仅用于测试环境。

---

### TS_DEBUG_DERP_WS_CLIENT

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `derp/derphttp/derphttp_client.go:326`

**功能**：启用 DERP WebSocket 客户端调试日志。

---

### TS_DEBUG_VERBOSE_DROPS

**类型**: String  
**默认值**: `""`  
**代码位置**: `derp/derpserver/derpserver.go:70`  
**用途**: DERP 服务器端

**功能**：设置详细丢包日志的节点列表。

---

## 端口映射和 NAT

### TS_DEBUG_NETCHECK

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/netcheck/netcheck.go:52`

**功能**：启用网络检测的详细调试日志。

---

### TS_DEBUG_NETCHECK_UDP_BIND

**类型**: String  
**默认值**: `""`  
**代码位置**: `cmd/tailscale/cli/netcheck.go:92`

**功能**：指定 netcheck UDP 绑定地址。

---

### TS_DEBUG_FORCE_ALL_IPV6_ENDPOINTS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/netmon/state.go:29`

**功能**：强制包含所有 IPv6 端点。

---

### TS_DEBUG_DISABLE_LIKELY_HOME_ROUTER_IP_SELF

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/netmon/state.go:605`

**功能**：禁用自动检测可能的家庭路由器 IP。

---

### TS_DEBUG_ENABLE_PMTUD

**类型**: OptBool (可选布尔)  
**默认值**: `未设置`  
**代码位置**: `wgengine/magicsock/debugknobs.go:60`  
**平台**: Linux/Darwin

**功能**：启用路径 MTU 发现（PMTUD）功能。

---

### TS_DEBUG_PMTUD

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:64`  
**平台**: Linux/Darwin

**功能**：启用 PMTUD 调试日志。

---

## DNS 配置

### TS_DEBUG_MAGIC_DNS_DUAL_STACK

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/dns/config.go:56`

**功能**：启用 MagicDNS 双栈（IPv4 + IPv6）支持。

---

### TS_DEBUG_DNS_FORWARD_SEND

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/dns/resolver/forwarder.go:514`

**功能**：启用 DNS 转发发送的详细调试日志。

---

### TS_DNS_FORWARD_SKIP_TCP_RETRY

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/dns/resolver/forwarder.go:515`

**功能**：跳过 DNS 转发的 TCP 重试。

---

### TS_DEBUG_DNS_FORWARD_USE_ROUTES

**类型**: OptBool (可选布尔)  
**默认值**: `未设置`  
**代码位置**: `net/dns/resolver/forwarder.go:748`

**功能**：控制 DNS 转发是否使用路由。

---

### TS_DEBUG_EXIT_NODE_DNS_NET_PKG

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/dns/resolver/tsdns.go:436`

**功能**：为出口节点 DNS 使用 net 包（调试用）。

---

### TS_DEBUG_DNS_CACHE

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/dnscache/dnscache.go:166`

**功能**：启用 DNS 缓存调试日志。

---

### TS_DEBUG_CONFIGURE_WSL

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/dns/manager_windows.go:43`  
**平台**: 仅 Windows

**功能**：配置 WSL 的 DNS 设置。

---

## SSH 功能

### TS_DEBUG_SSH_VLOG

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ssh/tailssh/tailssh.go:50`

**功能**：启用 SSH 详细日志。

---

### TS_SSH_DISABLE_SFTP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ssh/tailssh/tailssh.go:51`

**功能**：禁用 SFTP 子系统。

---

### TS_SSH_DISABLE_FORWARDING

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ssh/tailssh/tailssh.go:52`

**功能**：禁用 SSH 端口转发。

---

### TS_SSH_DISABLE_PTY

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ssh/tailssh/tailssh.go:53`

**功能**：禁用 SSH PTY（伪终端）分配。

---

### TS_DEBUG_LOG_SSH

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ssh/tailssh/tailssh.go:1058`

**功能**：将 SSH 会话记录到本地磁盘。

---

### TAILSCALE_SSH_DEFAULT_PATH

**类型**: String  
**默认值**: `""`  
**代码位置**: `ssh/tailssh/user.go:85`

**功能**：设置 SSH 会话的默认 PATH 环境变量模板。

---

### TS_DEBUG_SSH_EXEC

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `cmd/tailscale/cli/ssh.go:110`

**功能**：启用 SSH 执行的调试日志。

---

## 日志和调试

### TS_LOG_VERBOSITY

**类型**: Integer  
**默认值**: `0`  
**代码位置**: `cmd/tailscaled/tailscaled.go:190`

**功能**：设置日志详细级别。

---

### TS_DEBUG_LOG_TIME

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `logpolicy/logpolicy.go:534`

**功能**：在日志中包含时间戳。

---

### TS_DEBUG_FORCE_H1_LOGS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `logpolicy/logpolicy.go:918`

**功能**：强制日志使用 HTTP/1.1。

---

### TS_DEBUG_LOGTAIL_WAKES

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `logtail/logtail.go:597`

**功能**：记录 logtail 唤醒和上传事件。

---

### TS_DEBUG_LOGTAIL_FLUSHDELAY

**类型**: String (Duration)  
**默认值**: `""`  
**代码位置**: `logtail/logtail.go:91`

**功能**：设置 logtail 刷新延迟。

---

### TS_DEBUG_LOG_RATE

**类型**: String  
**默认值**: `""`  
**代码位置**: `types/logger/logger.go:164`

**功能**：设置为 `"all"` 可禁用日志速率限制。

---

### TS_PLEASE_PANIC

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `cmd/tailscaled/tailscaled.go:465`

**功能**：启用后，某些错误会触发 panic 而非优雅处理（调试用）。

---

### TS_DEBUG_MEMORY

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `cmd/tailscaled/tailscaled.go:460`

**功能**：启用内存调试。

---

### TS_DEBUG_DISABLE_WATCHDOG

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/watchdog.go:36`

**功能**：禁用 WireGuard 引擎看门狗。

---

## TUN 设备

### TS_TUN_DISABLE_UDP_GRO

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/tstun/wrap_linux.go:30`  
**平台**: 仅 Linux

**功能**：禁用 TUN 设备的 UDP GRO（通用接收卸载）。

---

### TS_TUN_DISABLE_TCP_GRO

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/tstun/wrap_linux.go:33`  
**平台**: 仅 Linux

**功能**：禁用 TUN 设备的 TCP GRO。

---

### TS_DEBUG_NETSTACK

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/netstack/netstack.go:119`

**功能**：启用 netstack（用户空间网络栈）调试日志。

---

### TS_DEBUG_NETSTACK_LEAK_MODE

**类型**: String  
**默认值**: `""`  
**代码位置**: `wgengine/netstack/netstack.go:127`

**功能**：设置 netstack 泄漏检测模式。

---

## 平台特定

### TS_FORCE_LINUX_BIND_TO_DEVICE

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/netns/netns_linux.go:57`  
**平台**: 仅 Linux

**功能**：强制使用 `SO_BINDTODEVICE` 绑定到特定网络设备。

---

### TS_BIND_TO_INTERFACE_BY_ROUTE

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/netns/netns_darwin.go:33` 和 `net/netns/netns_windows.go:48`  
**平台**: macOS 和 Windows

**功能**：基于路由表绑定到接口。

---

### TS_DEBUG_USE_IP_COMMAND

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/router/osrouter/router_linux.go:263`  
**平台**: 仅 Linux

**功能**：强制使用 `ip` 命令而非 netlink 进行路由配置。

---

### TS_DEBUG_FIREWALL_MODE

**类型**: String  
**默认值**: `""`  
**代码位置**: `util/linuxfw/detector.go:34`  
**平台**: 仅 Linux

**功能**：设置防火墙模式（`iptables`、`nftables` 等）。

---

### TS_DEBUG_NETLINK

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/netmon/netmon_linux.go:22`  
**平台**: 仅 Linux

**功能**：启用 Netlink 消息调试日志。

---

### TS_FAKE_SYNOLOGY

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `client/web/web.go:960`

**功能**：模拟 Synology 环境（测试用）。

---

## 其他功能

### TS_DEBUG_DISCO

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:22`

**功能**：打印活跃 discovery 事件的详细日志。

---

### TS_DEBUG_MAGICSOCK_PEERMAP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:24`

**功能**：打印 peermap 变更的详细日志。

---

### TS_DEBUG_OMIT_LOCAL_ADDRS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:27`

**功能**：从发现的本地端点中移除所有本地接口地址（测试用）。

---

### TS_DEBUG_ENABLE_SILENT_DISCO

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:45`

**功能**：禁用端点结构的心跳计时器，尝试静默处理 disco。

---

### TS_DEBUG_SEND_CALLME_UNKNOWN_PEER

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:48`

**功能**：每次发送真实 CallMeMaybe 时，同时向不存在的目标发送一个（测试用）。

---

### TS_DEBUG_MAGICSOCK_BIND_SOCKET

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/magicsock/debugknobs.go:50`

**功能**：打印 magicsock socket 重绑定的额外调试信息。

---

### TS_DEBUG_MAGICSOCK_RING_BUFFER_MAX_SIZE_BYTES

**类型**: Integer  
**默认值**: `0`  
**代码位置**: `wgengine/magicsock/debugknobs.go:53`

**功能**：覆盖端点历史环形缓冲区的默认大小。

---

### TS_DEBUG_PRETENDPOINT

**类型**: String  
**默认值**: `""`  
**代码位置**: `wgengine/magicsock/debugknobs.go:86`

**功能**：设置 0-3 个逗号分隔的假端点地址（格式：`ip:port`）。

---

### TS_DEBUG_TLS_DIAL

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `net/tlsdial/tlsdial.go:40`

**功能**：启用 TLS 拨号调试日志。

---

### TS_DEBUG_ACME

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ipn/ipnlocal/cert.go:90`

**功能**：启用 ACME 证书管理调试日志。

---

### TS_DEBUG_ACME_FORCE_RENEWAL

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ipn/ipnlocal/cert.go:558`

**功能**：强制 ACME 证书续期。

---

### TS_DEBUG_ACME_DIRECTORY_URL

**类型**: String  
**默认值**: `""`  
**代码位置**: `ipn/ipnlocal/cert.go:747`

**功能**：设置自定义 ACME 目录 URL。

---

### TS_DEBUG_PROFILES

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ipn/ipnlocal/profiles.go:30`

**功能**：启用配置文件调试日志。

---

### TS_DEBUG_WHOIS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ipn/ipnlocal/local.go:1460`

**功能**：启用 whois 查询调试日志。

---

### TS_ALLOW_SELF_INGRESS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ipn/ipnlocal/peerapi.go:583`

**功能**：允许自身入站流量。

---

### TS_DEBUG_PANIC_MACHINE_KEY

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ipn/ipnlocal/local.go:3548`

**功能**：机器密钥生成时触发 panic（调试用）。

---

### TS_DEBUG_DISABLE_PORTLIST

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `portlist/poller.go:23`

**功能**：禁用端口列表轮询。

---

### TS_DEBUG_TRIM_WIREGUARD

**类型**: OptBool (可选布尔)  
**默认值**: `未设置`  
**代码位置**: `wgengine/userspace.go:629`

**功能**：控制 WireGuard 配置修剪行为。

---

### TS_DEBUG_RAW_WGLOG

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `wgengine/wglog/wglog.go:91`

**功能**：输出原始 WireGuard 日志。

---

### TS_WAKE_MAC

**类型**: String  
**默认值**: `""`  
**代码位置**: `feature/wakeonlan/wakeonlan.go:160`

**功能**：设置 Wake-on-LAN MAC 地址。值可以是 MAC 地址、`"false"` 或 `"auto"`。

---

### TS_DEBUG_TPM

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `feature/tpm/tpm.go:83`

**功能**：启用 TPM（可信平台模块）调试日志。

---

### TS_DEBUG_WEB_CLIENT_DEV

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `client/web/web.go:191`

**功能**：启用 Web 客户端开发模式。

---

### TS_PERMIT_CERT_UID

**类型**: String  
**默认值**: `""`  
**代码位置**: `ipn/ipnserver/server.go:396`

**功能**：允许特定 UID 获取证书。

---

### TS_ALLOW_DEBUG_IP

**类型**: String  
**默认值**: `""`  
**代码位置**: `tsweb/tsweb.go:74`

**功能**：允许特定 IP 访问调试端点。

---

### TS_DEBUG_KEY_PATH

**类型**: String  
**默认值**: `""`  
**代码位置**: `tsweb/tsweb.go:85`

**功能**：设置调试密钥文件路径。

---

### TS_DEBUG_PATCHIFY_PEER

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlclient/map.go:591`

**功能**：启用 peer patchify 调试日志。

---

### TS_DEBUG_FILTER_RATE_LIMIT_LOGS

**类型**: String  
**默认值**: `""`  
**代码位置**: `wgengine/filter/filter.go:303`

**功能**：设置为 `"all"` 可禁用过滤器日志速率限制。

---

### TS_DEBUG_PERMIT_HTTP_C2N

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `control/controlclient/direct.go:1475`

**功能**：允许 HTTP C2N（控制到节点）连接。

---

### TS_SERIAL_TESTS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `tstest/tstest.go:89`

**功能**：串行运行测试（而非并行）。

---

### TS_DEBUG_FAKE_HEALTH_ERROR

**类型**: String  
**默认值**: `""`  
**代码位置**: `health/health.go:1068`

**功能**：设置假的健康检查错误（测试用）。

---

### TS_BE_CLI

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `cmd/tailscaled/tailscaled.go:163`

**功能**：让 tailscaled 作为 CLI 工具运行。

---

### TS_DUMP_HELP

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `cmd/tailscale/cli/cli.go:148`

**功能**：输出帮助信息（用于生成文档）。

---

### TS_DEBUG_SLOW_PUSH

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `cmd/tailscale/cli/file.go:170`

**功能**：模拟慢速文件推送（测试用）。

---

### TS_ASSUME_NETWORK_UP_FOR_TEST

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `ipn/ipnlocal/local.go:933`

**功能**：测试时假设网络已连接。

---

### TS_EXPERIMENTAL_KUBE_API_EVENTS

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `k8s-operator/api-proxy/proxy.go:100`

**功能**：启用实验性 Kubernetes API 事件功能。

---

### TAILSCALE_USE_WIP_CODE

**类型**: Boolean  
**默认值**: `false`  
**代码位置**: `envknob/envknob.go:385`

**功能**：允许使用 WIP（进行中）代码。某些实验性功能需要此变量。

---

### TAILSCALE_DERPER_MESH_KEY

**类型**: String  
**默认值**: `""`  
**代码位置**: `cmd/derper/derper.go:107`  
**用途**: DERP 服务器

**功能**：设置 DERP mesh 密钥，用于 DERP 服务器之间的认证。

---

## 配置示例

### DLP 环境完整配置（Linux systemd）

```ini
[Service]
# 强制所有流量通过 DERP（禁用 UDP/ICMP）
Environment="TS_DEBUG_ALWAYS_USE_DERP=1"

# 禁用直接 UDP 连接
Environment="TS_DEBUG_NEVER_DIRECT_UDP=1"

# 空闲时停止周期性检测（需要 45 秒无活动）
Environment="TS_DEBUG_RESTUN_STOP_ON_IDLE=1"

# 强制控制平面使用 HTTPS 443
Environment="TS_FORCE_NOISE_443=1"

# 禁用端口映射
Environment="TS_DISABLE_PORTMAPPER=1"
Environment="TS_DISABLE_UPNP=1"
```

**应用配置**：
```bash
sudo mkdir -p /etc/systemd/system/tailscaled.service.d/
sudo tee /etc/systemd/system/tailscaled.service.d/override.conf << 'EOF'
[Service]
Environment="TS_DEBUG_ALWAYS_USE_DERP=1"
Environment="TS_DEBUG_NEVER_DIRECT_UDP=1"
Environment="TS_DEBUG_RESTUN_STOP_ON_IDLE=1"
Environment="TS_FORCE_NOISE_443=1"
Environment="TS_DISABLE_PORTMAPPER=1"
Environment="TS_DISABLE_UPNP=1"
EOF

sudo systemctl daemon-reload
sudo systemctl restart tailscaled
```

### Windows 配置

通过系统环境变量设置：

```powershell
[System.Environment]::SetEnvironmentVariable("TS_DEBUG_ALWAYS_USE_DERP", "1", "Machine")
[System.Environment]::SetEnvironmentVariable("TS_FORCE_NOISE_443", "1", "Machine")
Restart-Service Tailscale
```

### macOS 配置

```bash
sudo launchctl setenv TS_DEBUG_ALWAYS_USE_DERP 1
sudo launchctl setenv TS_FORCE_NOISE_443 1
# 重启 Tailscale
```

---

## 注意事项

1. **`TS_DEBUG_` 前缀变量**：这些是调试变量，可能在未来版本中更改或移除，不建议在生产环境中长期依赖。

2. **环境变量值**：布尔类型通常接受 `1`、`true`、`yes` 为真值，空或 `0`、`false`、`no` 为假值。

3. **重启要求**：大多数环境变量需要重启 tailscaled 服务才能生效。

4. **平台差异**：某些变量仅在特定平台上有效（已在文档中标注）。

5. **组合使用**：某些变量组合使用可能产生意外效果，建议逐个测试。

---

*文档版本：基于 Tailscale 源代码分析*
*最后更新：2024*
