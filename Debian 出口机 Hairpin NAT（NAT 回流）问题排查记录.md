# 

## 问题现象

内网客户端访问：

```text
http://www.9017i.cc:58901
http://www.9017i.cc:25666
```

出现异常：

- 浏览器超时
- Linux curl 卡住
- Windows `curl.exe` 超时

但直接访问内网 IP 正常：

```text
http://192.168.99.4:18901
http://192.168.99.3:5666
```

---

# Linux 侧表现

执行：

```bash
curl -v http://www.9017i.cc:58901
```

输出：

```text
Established connection
Request completely sent off
```

然后无响应卡死。

说明：

- TCP 三次握手成功
- HTTP 请求已经发送
- 服务端响应未正确返回

---

# Windows 侧表现

浏览器：

```text
ERR_CONNECTION_TIMED_OUT
```

执行：

```powershell
curl.exe -v http://219.157.177.138:58901
```

卡在：

```text
Trying 219.157.177.138:58901...
```

但 PowerShell 的：

```powershell
curl
```

（实际是 Invoke-WebRequest）

却能返回 HTTP 200。

---

# 初期误判方向

曾怀疑：

- IPv6 AAAA 记录
- 浏览器 QUIC
- MTU/MSS
- VPN/TUN
- Windows 网络栈
- OpenWrt/爱快 Flow Offloading
- Docker 网络
- 硬件 NAT

后续逐步排除。

---

# 网络拓扑

```text
内网客户端
    ↓
Debian 出口机（NAT）
    ↓
公网 IP：219.157.177.138
    ↓
端口映射：
58901 -> 192.168.99.4:18901
25666 -> 192.168.99.3:5666
```

---

# 核心问题定位

系统存在：

## DNAT

例如：

```nft
tcp dport 58901 dnat ip to 192.168.99.4:18901
```

但：

# 缺少 Hairpin NAT 的 SNAT/MASQUERADE

---

# 问题根因

客户端：

```text
192.168.99.x
```

访问：

```text
219.157.177.138:58901
```

经过 DNAT：

```text
192.168.99.4:18901
```

但服务端看到：

```text
源IP仍是 192.168.99.x
```

于是：

- 服务端直接本地回包
- 回包绕过 NAT
- conntrack 状态不一致
- TCP 会话卡死

因此：

```text
Established connection
Request completely sent off
```

后无响应。

---

# nftables 排查过程

系统使用：

```text
iptables-nft
nftables
```

传统：

```bash
iptables -t nat -L
```

报错：

```text
table `nat' is incompatible, use 'nft' tool.
```

因此改用：

```bash
nft list ruleset
```

---

# 发现 DNAT 规则

在：

```nft
table inet roceos
```

中发现：

```nft
tcp dport 58901 dnat ip to 192.168.99.4:18901
tcp dport 25666 dnat ip to 192.168.99.3:5666
```

---

# 发现缺失 Hairpin SNAT

postrouting 中只有：

```nft
oifname "ppp0" masquerade
```

即：

# 只对出公网流量做 SNAT

但：

```text
内网 → 公网IP → 再回内网
```

不会经过：

```text
oifname ppp0
```

因此：

# Hairpin NAT 未做 masquerade

---

# 最终修复方案

新增独立 nftables 表：

```nft
table inet hairpin_nat {
    chain postrouting {
        type nat hook postrouting priority srcnat + 1; policy accept;

        ip saddr 192.168.0.0/16 \
        ip daddr 192.168.0.0/16 \
        masquerade
    }
}
```

作用：

# 所有 192.168.x.x 内网互访的 Hairpin NAT 自动做 SNAT

以后新增端口映射无需单独配置。

---

# 为什么使用独立 table

原系统：

```text
table inet roceos
```

由 Web 管理系统动态生成。

直接修改：

- 可能被覆盖
- Docker/Web 重启后可能丢失

因此：

# 独立 table 更安全

---

# 持久化配置

创建：

```bash
/etc/nftables.d/hairpin_nat.nft
```

内容：

```nft
table inet hairpin_nat {
    chain postrouting {
        type nat hook postrouting priority srcnat + 1; policy accept;

        ip saddr 192.168.0.0/16 \
        ip daddr 192.168.0.0/16 \
        masquerade
    }
}
```

---

# 主配置

创建：

```bash
/etc/nftables.conf
```

内容：

```nft
#!/usr/sbin/nft -f

include "/etc/nftables.d/*.nft"
```

---

# 启用 nftables

```bash
systemctl enable nftables
systemctl restart nftables
```

---

# 最终结果

内网客户端现在可以正常通过公网域名访问：

```text
http://www.9017i.cc:58901
http://www.9017i.cc:25666
```

Hairpin NAT 恢复正常。

---

# 最终结论

这是：

# Linux NAT 回流（Hairpin NAT）缺少 SNAT/MASQUERADE

导致的经典问题。

核心原则：

```text
DNAT 负责进去
SNAT 负责回来
```

Hairpin NAT 必须：

# DNAT + SNAT 同时存在

否则 conntrack 会异常。