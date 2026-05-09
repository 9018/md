# WireGuard + wg-easy + v2rayA TUN 全局代理完整教程（Kali / iStoreOS / Debian）

## 一、最终目标

实现：

```text
iPhone / 外网设备
↓
WireGuard
↓
家里 Kali / iStore
↓
v2rayA TUN
↓
国外节点
```

同时支持：

- 4G 回家
    
- 局域网访问
    
- 全局翻墙
    
- iPhone 单 VPN
    
- WireGuard + v2rayA 联动
    
- 外网访问内网
    
- SD-WAN 基础架构
    

---

# 二、网络结构

## 宿主机

```text
eth0 = 192.168.99.4
```

## PPPoE

```text
ppp0 = 公网IP
MTU 1492
```

## WireGuard

```text
wg0 = 10.8.0.1/24
```

## v2rayA TUN

```text
tun0 = 172.19.0.1/30
```

---

# 三、wg-easy Docker 部署（Host 模式）

## docker-compose.yml

```yaml
volumes:
  etc_wireguard:

services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy

    network_mode: host

    environment:
      - PORT=51821
      - HOST=0.0.0.0
      - INSECURE=true

    volumes:
      - etc_wireguard:/etc/wireguard
      - /lib/modules:/lib/modules:ro

    restart: unless-stopped

    cap_add:
      - NET_ADMIN
      - SYS_MODULE
```

---

# 四、启动 wg-easy

```bash
docker compose up -d
```

---

# 五、开启内核转发

## 临时

```bash
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.all.src_valid_mark=1
```

## 永久

```bash
cat >/etc/sysctl.d/99-wg-easy.conf <<EOF
net.ipv4.ip_forward=1
net.ipv4.conf.all.src_valid_mark=1
net.ipv6.conf.all.forwarding=1
EOF

sysctl --system
```

---

# 六、爱快端口映射

映射：

```text
UDP 51820
```

到：

```text
192.168.99.4
```

---

# 七、WireGuard 客户端配置

## iPhone 安装

WireGuard 官方客户端。

---

## 客户端配置关键项

```ini
AllowedIPs = 0.0.0.0/0
DNS = 1.1.1.1
MTU = 1280
```

---

# 八、v2rayA TUN 配置

## 开启 TUN

v2rayA WebUI：

```text
设置
→ TUN 模式
→ 开启
```

---

## 推荐关闭

- FakeDNS
    
- DNS Hijack
    
- 自动修改 resolv.conf
    

---

## 推荐 DNS

```text
1.1.1.1
8.8.8.8
```

---

# 九、查看当前路由

```bash
ip route get 8.8.8.8
```

正常：

```text
8.8.8.8 dev tun0 table 2022
```

说明：

v2rayA 已接管默认流量。

---

# 十、关键 iptables 配置

## 放行 WireGuard

```bash
iptables -I FORWARD 1 -i wg0 -j ACCEPT
iptables -I FORWARD 1 -o wg0 -j ACCEPT
```

---

## NAT 到 v2rayA tun0

```bash
iptables -t nat -I POSTROUTING 1 -s 10.8.0.0/24 -o tun0 -j MASQUERADE
```

---

# 十一、查看当前规则

```bash
iptables-save
```

---

# 十二、查看 NAT 是否生效

```bash
watch -n 1 'iptables -t nat -L POSTROUTING -n -v'
```

重点看：

```text
10.8.0.0/24 -> tun0
```

计数是否增长。

---

# 十三、查看 WireGuard 状态

```bash
wg
```

正常：

```text
latest handshake
transfer
```

---

# 十四、查看接口

```bash
ip -br addr | grep -E 'wg|tun'
```

正常：

```text
wg0
tun0
```

---

# 十五、查看公网监听

```bash
ss -lunp | grep 51820
```

---

# 十六、抓包排查

## 查看 WireGuard UDP 包

```bash
tcpdump -ni ppp0 udp port 51820
```

---

# 十七、问题排查总结

## 1. 4G 无法连接

原因：

- 爱快 UDP 51820 未映射
    
- Endpoint 错误
    
- 公网 IP 错误
    

排查：

```bash
tcpdump -ni ppp0 udp port 51820
```

---

## 2. 能连接但不能上网

原因：

缺少：

```bash
MASQUERADE -> tun0
```

修复：

```bash
iptables -t nat -I POSTROUTING 1 -s 10.8.0.0/24 -o tun0 -j MASQUERADE
```

---

## 3. Docker bridge 模式冲突

原因：

Docker bridge + TUN + policy route 冲突。

修复：

改：

```text
network_mode: host
```

---

## 4. ping 看起来很慢

实际：

不是网络问题。

原因：

v2rayA 对 ICMP 有特殊策略：

```text
ipproto icmp goto 9010
```

表现：

```text
PING ...
（等待）
64 bytes from ...
```

但：

- curl 秒开
    
- HTTPS 正常
    
- RTT 正常
    

所以无需纠结 ping。

---

# 十八、验证最终是否成功

## 1. 查看出口 IP

```bash
curl https://ip.sb
```

---

## 2. HTTP 测试

```bash
curl -I https://www.baidu.com
```

---

## 3. iPhone 测试

访问：

```text
https://ip.sb
```

应显示代理节点 IP。

---

# 十九、最终效果

实现：

```text
iPhone
↓
WireGuard
↓
wg-easy
↓
v2rayA tun0
↓
国外
```

支持：

- 回家
    
- 翻墙
    
- 局域网
    
- 单 VPN
    
- SD-WAN
    
- 多地点互联
    
- 外网访问内网