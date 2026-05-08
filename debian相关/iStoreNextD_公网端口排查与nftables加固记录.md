# iStoreNextD 公网暴露端口排查与 nftables 加固记录

## 一、排查背景

发现 iStoreNextD 存在较多监听端口，怀疑公网暴露面过大，需要确认：

- 哪些端口只是本地监听
- 哪些端口真正暴露公网
- 哪些属于 Docker DNAT 转发
- 如何通过 nftables 限制公网访问

系统环境：

- Debian 13
- nftables
- PPPoE 拨号
- Docker
- iStoreOS / roceos

---

# 二、初始端口监听情况

通过：

```bash
netstat -an
ss -tulpn
```

发现存在以下监听：

## TCP

| 端口 | 服务 |
|---|---|
| 22 | SSH |
| 80 | nginx |
| 443 | nginx |
| 139 | Samba |
| 445 | Samba |
| 3000 | docker-proxy |
| 5005 | roceos |
| 8833 | docker-proxy |
| 9876 | ddns-go |

## UDP

| 端口 | 服务 |
|---|---|
| 1194 | OpenVPN |
| 137/138 | Samba nmbd |

---

# 三、确认公网出口接口

执行：

```bash
ip route get 8.8.8.8
```

结果：

```txt
8.8.8.8 dev ppp0
```

确认：

- 公网出口接口为 `ppp0`
- PPPoE 拨号公网 IP：
  `219.157.177.138`

随后：

```bash
ip -br addr
```

确认：

- eth1 = 192.168.100.1
- eth2 = 192.168.99.1
- ppp0 = 公网接口

---

# 四、分析 nftables 规则

通过：

```bash
nft list ruleset
```

发现：

## Docker DNAT

```nft
tcp dport 3000 dnat to 172.18.0.2:3000
tcp dport 8833 dnat to 172.19.0.2:8833
udp dport 1194 dnat to 172.19.0.2:1194
```

## 自定义端口映射

```nft
tcp dport 25666 dnat to 192.168.99.3:5666
tcp dport 58901 dnat to 192.168.99.4:18901
```

说明：

公网存在多个 DNAT 转发。

---

# 五、发现 Hairpin NAT

发现规则：

```nft
table inet hairpin_nat {
    chain postrouting {
        ip saddr 192.168.0.0/16 ip daddr 192.168.0.0/16 masquerade
    }
}
```

解释：

这是 NAT Loopback / Hairpin NAT。

作用：

- 内网设备访问“公网域名”
- 流量再转回内网服务
- 支持 DDNS / HTTPS 域名回环访问

因此：

内网测试公网 IP 时：

```txt
192.168.x.x -> 公网IP -> 内网服务
```

不一定经过：

```txt
ppp0 INPUT
```

所以：

即使已经 drop 公网端口，
内网访问公网 IP 仍可能正常。

---

# 六、nftables 加固过程

## 1. 备份规则

```bash
nft list ruleset > /root/nft-backup.txt
```

---

## 2. 初次添加规则失败

错误原因：

```bash
{139,445}
```

被 shell 展开导致 nft 语法错误。

报错：

```txt
syntax error, unexpected drop
```

---

## 3. 正确写法

需要加引号：

```bash
nft add rule inet roceos input iifname "ppp0" tcp dport "{139,445}" drop
nft add rule inet roceos input iifname "ppp0" udp dport "{137,138}" drop
```

---

## 4. 封禁 Docker/Web 调试端口

```bash
nft add rule inet roceos input iifname "ppp0" tcp dport "{3000,5005,8833,9876}" drop
```

---

## 5. 封禁公网 Web

```bash
nft add rule inet roceos input iifname "ppp0" tcp dport "{80,443}" drop
```

---

# 七、最终 INPUT 链

```nft
chain input {
    type filter hook input priority filter; policy accept;

    ct state established,related accept

    tcp dport 22 ip saddr 192.168.0.0/16 accept
    tcp dport 22 drop

    iifname "ppp0" tcp dport { 139, 445 } drop
    iifname "ppp0" udp dport { 137, 138 } drop

    iifname "ppp0" tcp dport { 3000, 5005, 8833, 9876 } drop

    iifname "ppp0" tcp dport { 80, 443 } drop
}
```

---

# 八、测试结果

使用：

- 手机关闭 Wi‑Fi
- 5G 网络测试

确认：

- 80/443 已无法公网访问
- Samba 已无法公网访问
- Docker 面板端口已无法公网访问

说明：

nftables 规则已生效。

---

# 九、当前公网仍开放

目前仍存在：

| 端口 | 用途 |
|---|---|
| 11194 | OpenVPN |
| 25666 | 转发到 5666 |
| 58901 | 转发到 18901 |

这些属于：

DNAT 转发规则。

若不再需要，可进一步删除对应 prerouting dnat。

---

# 十、经验总结

## 1. netstat 监听 ≠ 公网开放

必须结合：

- nftables
- DNAT
- FORWARD
- Docker
- PPPoE 接口

综合判断。

---

## 2. Hairpin NAT 会影响测试

内网访问公网 IP：

不代表真实公网访问路径。

必须：

- 手机 5G
- VPS
- 外部网络

进行验证。

---

## 3. nftables 规则顺序重要

规则从上往下匹配。

必要时：

```bash
nft insert rule
```

而不是：

```bash
nft add rule
```

---

## 4. Docker 会自动开放端口

docker-proxy 默认：

```txt
0.0.0.0
```

容易形成公网暴露。

需要重点检查：

```bash
docker ps
docker inspect
```

---

# 十一、建议后续优化

## 推荐

- INPUT policy 改为 drop
- 仅白名单放行
- 删除不必要 DNAT
- Samba 仅监听内网
- ddns-go 增加认证
- Docker 服务仅绑定 localhost
- 使用 fail2ban 防爆破

---

# 十二、关键命令记录

## 查看监听

```bash
ss -tulpn
netstat -an
```

## 查看公网出口

```bash
ip route get 8.8.8.8
```

## 查看地址

```bash
ip -br addr
```

## 查看 nftables

```bash
nft list ruleset
```

## 查看指定链

```bash
nft list chain inet roceos input
```

## 备份 nft

```bash
nft list ruleset > /root/nft-backup.txt
```
