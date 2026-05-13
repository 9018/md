# iOS 小火箭 + v2rayA + NetBird 访问公司/家里内网并出国教程

> 记录时间：2026-05-12  
> 场景目标：iPhone 只开一个 VPN/代理客户端（小火箭），同时实现：
>
> - 访问家里内网：`192.168.99.0/24`
> - 访问公司内网：`10.0.1.0/24`
> - 访问海外网站走 v2rayA 出口
> - 公司与家里之间由 NetBird 打通
> - iOS 不直接跑 NetBird，只连接家里的 v2rayA / Xray 入站

---

## 1. 最终拓扑

```text
iPhone 小火箭
    |
    | VMess 入站 （带国内外分流，国内还走iphone，国外才走192.168.99.4的tun0代理，设置上要强制局域网走代理）
    v
家里 v2rayA / Xray 主机：192.168.99.4   eth0-tun0
    |
    | v2rayA 分流：
    | - 10.0.1.0/24      -> direct
    | - 192.168.99.0/24  -> direct
    | - 国外网站          -> proxy
    | - 国内网站          -> direct
    v
Linux 系统路由
    |
    | 10.0.1.0/24 dev wt0
    v
NetBird 虚拟网卡 wt0
    |
    v
公司机器：10.0.1.103
```

核心思路：

```text
小火箭：10.0.1.0/24 走 PROXY，到家里 v2rayA
v2rayA：10.0.1.0/24 走 direct，交给 Linux 系统路由
Linux：10.0.1.0/24 走 wt0，也就是 NetBird
```

---

## 2. 关键结论

这次问题不是单纯的 WireGuard 或 NetBird 问题，而是多层叠加：

1. `10.0.1.0/24` 必须走 NetBird 的 `wt0`，不能再走旧的 `wg0`。
2. v2rayA 的普通 HTTP 代理端口 `20171` 默认可能走 `proxy`。
3. v2rayA 的分流 HTTP 端口 `20172` 才会按 RoutingA 规则走 `direct/proxy`。
4. 小火箭 VMess 入站必须命中服务端 v2rayA 的分流规则。
5. 当 v2rayA 把流量 direct 到 NetBird 后，源地址可能是 `192.168.99.4`，公司端可能不回包。
6. 最终通过在家里 v2rayA 主机上对发往 `10.0.1.0/24` 的流量做 `MASQUERADE` 解决。

---

## 3. 基础环境

### 家里

```text
家里 LAN：192.168.99.0/24
v2rayA 主机：192.168.99.4
NetBird 网卡：wt0
WireGuard 网卡：wg0
```

### 公司

```text
公司 LAN：10.0.1.0/24
公司目标机器：10.0.1.103
Streamlit 服务：10.0.1.103:8501
v2rayA Web：10.0.1.103:2017
```

### v2rayA 常见端口

```text
2017   v2rayA Web 管理页面
20170  SOCKS5 代理
20171  HTTP 代理
20172  rule-http，带分流规则的 HTTP 代理
29017  VMess/VLESS 等给小火箭连接的入站端口
```

实际以命令输出为准：

```bash
ss -lntup | grep -E '2017|20170|20171|20172|29017|xray|v2ray|v2raya'
```

---

## 4. 第一步：确认系统路由走 NetBird

在家里 v2rayA 主机上执行：

```bash
ip route show table main | grep -E '10\.0\.1|wg0|wt0'
ip route get 10.0.1.103
```

正确结果应该类似：

```text
10.0.1.0/24 dev wt0 scope link metric 5
10.8.0.0/24 dev wg0 proto kernel scope link src 10.8.0.1
100.104.0.0/16 dev wt0 proto kernel scope link src 100.104.x.x
10.0.1.103 dev wt0 src 100.104.x.x
```

重点：

```text
10.0.1.103 dev wt0
```

这表示访问公司 `10.0.1.103` 会走 NetBird，而不是旧的 WireGuard `wg0`。

如果看到：

```text
10.0.1.0/24 dev wg0
```

说明旧路由还在抢 10 段，要删除或修改 WireGuard 配置。

临时删除：

```bash
ip route del 10.0.1.0/24 dev wg0 2>/dev/null
ip route replace 10.0.1.0/24 dev wt0
```

如果重启后又回来，检查：

```bash
grep -R "10.0.1.0/24" /etc/wireguard/ /etc/systemd/network/ /etc/netplan/ 2>/dev/null
wg show
```

---

## 5. 第二步：确认本机能直连公司服务

在家里 v2rayA 主机上执行：

```bash
curl -v http://10.0.1.103:2017
```

成功时会返回 v2rayA 页面，例如：

```html
<title>v2rayA</title>
```

这说明：

```text
家里主机 -> wt0 / NetBird -> 10.0.1.103:2017
```

已经通了。

也可以测 Streamlit：

```bash
curl -v http://10.0.1.103:8501
```

---

## 6. 第三步：确认 v2rayA 分流是否正确

### 6.1 测普通 HTTP 代理

```bash
curl -x http://127.0.0.1:20171 http://10.0.1.103:2017 -v
```

如果日志出现：

```text
[http -> proxy]
```

说明 `20171` 普通 HTTP 代理把 10 段送到了出国节点，不是我们想要的结果。

错误表现可能是：

```text
HTTP/1.1 503 Service Unavailable
```

### 6.2 测分流 HTTP 代理

```bash
curl -x http://127.0.0.1:20172 http://10.0.1.103:2017 -v
```

正确日志应该是：

```text
[rule-http -> direct]
```

这表示：

```text
v2rayA rule-http 入站 -> direct -> Linux 系统路由 -> wt0 -> NetBird
```

### 6.3 测 SOCKS

```bash
curl --socks5-hostname 127.0.0.1:20170 http://10.0.1.103:2017 -v
```

如果失败，要看日志里是 `socks -> proxy` 还是 `socks -> direct`。

---

## 7. v2rayA RoutingA 规则

进入 v2rayA Web：

```text
http://192.168.99.4:2017
```

切到：

```text
RoutingA / 分流模式
```

添加规则，放在最前面：

```text
ip(10.0.1.0/24) -> direct
ip(10.0.0.0/8) -> direct
ip(100.64.0.0/10) -> direct
ip(192.168.0.0/16) -> direct
default: proxy
```

说明：

```text
10.0.1.0/24      公司内网
10.0.0.0/8       兼容更大私有地址范围
100.64.0.0/10    NetBird / CGNAT 虚拟地址段
192.168.0.0/16   家里内网
default: proxy   其他流量走出国节点
```

保存后，在 v2rayA 页面点：

```text
应用 / 重启核心
```

也可以重启服务：

```bash
systemctl restart v2raya
```

---

## 8. 小火箭规则

手机端小火箭规则不要把 10 段写成 `DIRECT`。

因为 iPhone 本地没有公司内网路由，必须先把流量送到家里的 v2rayA。

小火箭里建议：

```text
10.0.1.0/24       PROXY
192.168.99.0/24   PROXY
192.168.100.0/24  PROXY
100.64.0.0/10     PROXY
GEOIP,CN          DIRECT
FINAL             PROXY
```

这里的 `PROXY` 是指：

```text
手机 -> 家里 v2rayA / Xray 入站
```

而不是直接去国外节点。

服务端收到后，再由 v2rayA 分流：

```text
10.0.1.0/24 -> direct -> wt0 / NetBird
```

---

## 9. 验证小火箭 VMess 入站是否走 direct

手机开小火箭，访问：

```text
http://10.0.1.103:8501
```

在家里 v2rayA 主机看日志：

```bash
journalctl -u v2raya -f
```

正确日志：

```text
from 192.168.99.1:54133 accepted tcp:10.0.1.103:8501 [vmess -> direct]
```

重点：

```text
[vmess -> direct]
```

这表示：

```text
小火箭 -> VMess 入站 -> direct -> 系统路由 -> wt0 / NetBird
```

如果看到：

```text
[vmess -> proxy]
```

说明 VMess 入站没命中分流规则，仍然走出国节点，需要重新检查 RoutingA 和入站绑定。

---

## 10. 抓包判断链路卡在哪里

### 10.1 在家里 v2rayA 主机抓 wt0

```bash
tcpdump -ni wt0 host 10.0.1.103 and port 8501
```

如果看到：

```text
192.168.99.4.xxxxx > 10.0.1.103.8501: Flags [S]
```

说明 v2rayA 已经把包发进 NetBird。

如果一直只有 `[S]`，没有 `[S.]`，说明对端没有回 SYN-ACK。

---

### 10.2 在公司 10.0.1.103 上抓包

```bash
tcpdump -ni any host 192.168.99.4 and port 8501
```

这次排查中，能看到：

```text
wt0 In IP 192.168.99.4.xxxxx > 10.0.1.103.8501: Flags [S]
```

说明包已经到达公司机器。

同时确认服务监听：

```bash
ss -lntup | grep 8501
```

结果：

```text
tcp LISTEN 0 2048 0.0.0.0:8501 0.0.0.0:* users:(("streamlit",pid=...,fd=6))
```

这说明 Streamlit 已经监听在所有地址上，不是 `127.0.0.1` 问题。

再查回程路由：

```bash
ip route get 192.168.99.4
```

结果类似：

```text
192.168.99.4 dev wt0 table 7120 src 100.104.200.116
```

说明公司端理论上知道回 `192.168.99.4` 应该走 `wt0`。

但是仍然没有 SYN-ACK，说明公司端对这种源地址为 `192.168.99.4` 的连接处理有问题，可能是防火墙、rp_filter、NetBird 策略或策略路由细节导致。

---

## 11. 曾尝试过的公司端 INPUT 放行

在 10.0.1.103 上尝试：

```bash
iptables -I INPUT 1 -i wt0 -s 192.168.99.0/24 -p tcp --dport 8501 -j ACCEPT
iptables -I INPUT 1 -i wt0 -s 100.64.0.0/10 -p tcp --dport 8501 -j ACCEPT
```

但抓包仍然只有：

```text
192.168.99.4.xxxxx > 10.0.1.103.8501: Flags [S]
```

没有：

```text
10.0.1.103.8501 > 192.168.99.4.xxxxx: Flags [S.]
```

所以最后选择在家里侧做 SNAT/MASQUERADE。

---

## 12. 最终修复：在家里 v2rayA 主机对 10 段做 MASQUERADE

### 12.1 临时规则

在家里 v2rayA 主机，也就是 `192.168.99.4` 上执行：

```bash
iptables -t nat -I POSTROUTING 1 -o wt0 -d 10.0.1.0/24 -j MASQUERADE
```

作用：

```text
把发往 10.0.1.0/24 且从 wt0 出去的流量做源地址伪装
```

修复前公司看到的源地址：

```text
192.168.99.4 -> 10.0.1.103:8501
```

修复后公司看到的源地址会变成 NetBird 侧地址，例如：

```text
100.104.162.148 -> 10.0.1.103:8501
```

这样公司端能正常回包。

---

### 12.2 验证

手机开小火箭访问：

```text
http://10.0.1.103:8501
```

同时在家里 v2rayA 主机抓包：

```bash
tcpdump -ni wt0 host 10.0.1.103 and port 8501
```

成功时应该看到：

```text
100.104.x.x.xxxxx > 10.0.1.103.8501: Flags [S]
10.0.1.103.8501 > 100.104.x.x.xxxxx: Flags [S.]
```

或者浏览器直接打开 Streamlit 页面。

---

## 13. nftables 持久化方案

如果系统主要用 nftables，建议在家里 v2rayA 主机上持久化成 nft 规则。

创建文件：

```bash
mkdir -p /etc/nftables.d

cat >/etc/nftables.d/netbird-snat.nft <<'EOF'
table ip netbird_snat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "wt0" ip daddr 10.0.1.0/24 masquerade
    }
}
EOF
```

确认 `/etc/nftables.conf` 有 include：

```bash
cat >/etc/nftables.conf <<'EOF'
#!/usr/sbin/nft -f

include "/etc/nftables.d/*.nft"
EOF
```

启用并重启 nftables：

```bash
systemctl enable nftables
systemctl restart nftables
```

检查规则：

```bash
nft list table ip netbird_snat
```

应该看到：

```text
table ip netbird_snat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "wt0" ip daddr 10.0.1.0/24 masquerade
    }
}
```

---

## 14. iptables 持久化方案

如果不想用 nftables，也可以用 iptables-persistent。

安装：

```bash
apt update
apt install -y iptables-persistent netfilter-persistent
```

保存当前规则：

```bash
netfilter-persistent save
systemctl enable netfilter-persistent
```

查看保存文件：

```bash
grep -n "10.0.1.0/24" /etc/iptables/rules.v4
```

---

## 15. 常用排查命令汇总

### 路由

```bash
ip route get 10.0.1.103
ip route show table main | grep -E '10\.0\.1|wg0|wt0'
ip rule
```

### 端口监听

```bash
ss -lntup | grep -E '2017|20170|20171|20172|29017|8501|xray|v2ray|v2raya|streamlit'
```

### v2rayA 日志

```bash
journalctl -u v2raya -f
journalctl -u v2raya -n 200 --no-pager | grep -Ei '10\.0\.1\.103|2017|8501|direct|proxy|block|outbound|error|failed'
```

### 本地直连测试

```bash
curl -v http://10.0.1.103:2017
curl -v http://10.0.1.103:8501
```

### v2rayA 代理测试

```bash
curl -x http://127.0.0.1:20171 http://10.0.1.103:2017 -v
curl -x http://127.0.0.1:20172 http://10.0.1.103:2017 -v
curl --socks5-hostname 127.0.0.1:20170 http://10.0.1.103:2017 -v
```

### 抓包

家里 v2rayA 主机：

```bash
tcpdump -ni wt0 host 10.0.1.103 and port 8501
```

公司 10.0.1.103：

```bash
tcpdump -ni any host 192.168.99.4 and port 8501
```

### NAT 规则

```bash
iptables -t nat -S POSTROUTING
iptables -t nat -L POSTROUTING -n -v --line-numbers
nft list ruleset | grep -Ei 'masquerade|10\.0\.1|wt0'
```

---

## 16. 最终可用配置摘要

### 家里 v2rayA 主机

必须满足：

```text
10.0.1.0/24 dev wt0
v2rayA RoutingA: 10.0.1.0/24 -> direct
小火箭入站日志: [vmess -> direct]
POSTROUTING: -o wt0 -d 10.0.1.0/24 MASQUERADE
```

临时关键命令：

```bash
iptables -t nat -I POSTROUTING 1 -o wt0 -d 10.0.1.0/24 -j MASQUERADE
```

### iPhone 小火箭

规则：

```text
10.0.1.0/24       PROXY
192.168.99.0/24   PROXY
100.64.0.0/10     PROXY
FINAL             PROXY
```

### 公司 10.0.1.103

确认服务监听：

```bash
ss -lntup | grep 8501
```

应为：

```text
0.0.0.0:8501
```

Streamlit 示例：

```bash
streamlit run app.py --server.address 0.0.0.0 --server.port 8501
```

---

## 17. 本次排查结论

最终可用链路：

```text
iPhone 小火箭
    ↓
家里 v2rayA / VMess
    ↓
v2rayA RoutingA 命中 direct
    ↓
Linux 路由走 wt0
    ↓
iptables/nftables MASQUERADE
    ↓
NetBird
    ↓
公司 10.0.1.103:8501
```

最关键的成功日志：

```text
[vmess -> direct]
```

最关键的成功修复：

```bash
iptables -t nat -I POSTROUTING 1 -o wt0 -d 10.0.1.0/24 -j MASQUERADE
```

一句话总结：

```text
iOS 不跑 NetBird，只连家里 v2rayA；
v2rayA 负责入口和分流；
NetBird 负责家里到公司的三层互通；
家里 v2rayA 主机对 10.0.1.0/24 做 MASQUERADE，解决公司端回包问题。
```
