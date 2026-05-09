# WireGuard + v2rayA TUN + Docker + 小火箭 完整稳定方案（Kali / Debian / iStoreOS）

# 一、目标架构

实现：

```text
iPhone / 外网设备
        ↓
    WireGuard
        ↓
  家里 Kali / Debian
        ↓
     v2rayA TUN
        ↓
      国外代理
```

同时支持：

- 内网访问（99 段 / 50 段）
    
- 外网翻墙
    
- iPhone 单 VPN
    
- 小火箭
    
- Docker
    
- v2rayA TUN
    
- wg-easy WebUI
    

---

# 二、最终网络结构

## 宿主机

```text
eth0 = 192.168.99.4
```

## PPPoE

```text
ppp0 = 公网IP
MTU = 1492
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

# 三、wg-easy 最终 docker-compose.yml

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

# 四、为什么必须 Host 模式

之前 bridge 模式：

```text
WG → Docker bridge → v2rayA tun0
```

会出现：

- NAT 混乱
    
- policy route 错乱
    
- MSS 问题
    
- MTU 黑洞
    

最终改：

```yaml
network_mode: host
```

彻底稳定。

---

# 五、开启系统转发

## 临时

```bash
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.all.src_valid_mark=1
```

---

## 永久

```bash
cat >/etc/sysctl.d/99-wg-v2raya.conf <<EOF
net.ipv4.ip_forward=1
net.ipv4.conf.all.src_valid_mark=1
EOF

sysctl --system
```

---

# 六、关键 NAT 规则

## WG 客户端流量 NAT 到 tun0

```bash
iptables -t nat -I POSTROUTING 1 \
-s 10.8.0.0/24 -o tun0 -j MASQUERADE
```

---

# 七、关键 FORWARD 规则

```bash
iptables -I FORWARD 1 -i wg0 -j ACCEPT
iptables -I FORWARD 1 -o wg0 -j ACCEPT
```

---

# 八、核心问题：MTU / MSS 黑洞

## 症状

表现：

- 百度打不开
    
- Telegram 卡
    
- 有的网站能开有的不行
    
- curl 正常但浏览器卡
    
- TCP 重传
    
- SYN ACK 后疯狂 SACK
    

抓包发现：

```text
TCP Retransmission
SACK
Repeated ACK
```

---

# 九、真正修复：MSS Clamp

## 核心规则

```bash
iptables -t mangle -I FORWARD \
-o wg0 \
-p tcp --tcp-flags SYN,RST SYN \
-j TCPMSS --clamp-mss-to-pmtu

iptables -t mangle -I FORWARD \
-i wg0 \
-p tcp --tcp-flags SYN,RST SYN \
-j TCPMSS --clamp-mss-to-pmtu
```

---

# 十、为什么会出现 MSS 问题

因为：

```text
WireGuard
↓
v2rayA TUN
↓
Docker
↓
PPPoE 1492
```

多层隧道叠加。

最终：

```text
MTU 不一致
```

导致：

```text
TCP 分片失败
```

---

# 十一、最终 MTU 推荐

## WireGuard 客户端

```ini
MTU = 1280
```

最稳。

---

# 十二、真正复杂的问题：policy routing

## 目标

实现：

```text
WG 客户端：
    内网直连
    外网走 tun0
```

---

# 十三、错误方案（踩坑）

之前：

```bash
ip rule add from 10.8.0.0/24 lookup 2022
```

导致：

```text
WG 访问 NAS
↓
被 v2rayA TUN 接管
↓
Docker NAT 错乱
```

表现：

```text
访问 18901
结果进入 3000
```

---

# 十四、正确方案（最终）

## 内网直连

```bash
ip rule add from 10.8.0.0/24 \
to 192.168.0.0/16 \
lookup main priority 50

ip rule add from 10.8.0.0/24 \
to 10.0.0.0/8 \
lookup main priority 51

ip rule add from 10.8.0.0/24 \
to 172.16.0.0/12 \
lookup main priority 52
```

---

## 外网走 v2rayA

```bash
ip rule add from 10.8.0.0/24 \
lookup 2022 priority 100
```

---

## 刷新缓存

```bash
ip route flush cache
```

---

# 十五、最终逻辑

## 内网

```text
10.8.0.x
 ↓
192.168.x.x
 ↓
main table
 ↓
直连
```

---

## 外网

```text
10.8.0.x
 ↓
table 2022
 ↓
tun0
 ↓
v2rayA
 ↓
国外
```

---

# 十六、小火箭配置

## 推荐：

```text
FINAL,PROXY
```

---

## 内网规则

如果：

### 外网回家：

```text
IP-CIDR,192.168.99.0/24,PROXY
IP-CIDR,192.168.50.0/24,PROXY
```

---

## 家里 WiFi 测试：

```text
IP-CIDR,192.168.99.0/24,DIRECT
```

否则：

```text
WiFi 本地路由
+
WG 隧道路由
```

会冲突。

---

# 十七、验证命令

## 查看 WG

```bash
wg
```

---

## 查看接口

```bash
ip -br addr | grep -E 'wg|tun'
```

---

## 查看策略路由

```bash
ip rule
```

---

## 查看 NAT

```bash
iptables -t nat -L POSTROUTING -n -v
```

---

## 查看 FORWARD

```bash
iptables -L FORWARD -n -v
```

---

## 查看 MSS

```bash
iptables -t mangle -L FORWARD -n -v
```

---

# 十八、抓包排查

## 查看 WG

```bash
tcpdump -ni wg0
```

---

## 查看 tun0

```bash
tcpdump -ni tun0
```

---

## 查看 Docker

```bash
tcpdump -ni any 'tcp port 18901 or tcp port 3000'
```

---

# 十九、最终效果

最终实现：

```text
iPhone
 ↓
WireGuard
 ↓
家里 99.4
 ├── 内网直连
 └── v2rayA 出国
```

同时：

- 小火箭单 VPN
    
- 内网访问
    
- 国外代理
    
- Docker
    
- wg-easy
    
- policy routing
    
- MSS 修复
    

全部稳定运行。

---

# 二十、开机持久化

## 安装

```bash
apt install iptables-persistent -y
```

---

## 保存 iptables

```bash
netfilter-persistent save
```

---

# 二十一、持久化 ip rule

## 脚本

```bash
nano /usr/local/bin/wg-v2raya-route.sh
```

内容：

```bash
#!/bin/bash

ip rule add from 10.8.0.0/24 to 192.168.0.0/16 lookup main priority 50 2>/dev/null
ip rule add from 10.8.0.0/24 to 10.0.0.0/8 lookup main priority 51 2>/dev/null
ip rule add from 10.8.0.0/24 to 172.16.0.0/12 lookup main priority 52 2>/dev/null

ip rule add from 10.8.0.0/24 lookup 2022 priority 100 2>/dev/null

ip route flush cache
```

---

## 赋权

```bash
chmod +x /usr/local/bin/wg-v2raya-route.sh
```

---

# 二十二、systemd 开机启动

```bash
nano /etc/systemd/system/wg-v2raya-route.service
```

内容：

```ini
[Unit]
Description=WG v2rayA routing
After=network.target v2raya.service docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/wg-v2raya-route.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

## 启用

```bash
systemctl daemon-reload
systemctl enable --now wg-v2raya-route.service
```