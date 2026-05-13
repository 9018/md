# Docker 部署 NetBird 服务端 + v2rayA 网关 + NetBird Client 教程

> 适用场景：NetBird 服务端、Dashboard、Traefik 都用 Docker 部署；家里/内网有一台机器同时运行 `v2rayA` 和 `NetBird Client`；手机/电脑连接 v2rayA 后，既要能出国，也要能访问 NetBird 对端内网，例如 `10.0.1.103:8501`。

---

## 0. 总体拓扑

```text
手机 / 电脑
  │
  │ 连接 v2rayA，例如 192.168.99.153
  ▼
家里网关机 / v2rayA 机器
  - Docker: v2raya
  - Docker: netbird-client
  - 宿主机出现 wt0
  - 对 192.168.99.0/24 -> wt0 做 SNAT
  │
  │ NetBird Overlay
  ▼
公司 / 远端 NetBird 节点
  - 例如 10.0.1.103
  - 服务端口 8501
```

核心思路：

```text
192.168.99.x 这种 LAN 客户端不是 NetBird Peer。
如果它原样从 wt0 进入 10.0.1.103，NetBird ACL 可能会丢弃。

所以在 v2rayA + NetBird Client 这台网关机上做 SNAT：
192.168.99.x -> 网关机自己的 NetBird IP，例如 100.104.x.x
```

这样目标机看到的是 NetBird Peer 来源，默认 Policy 或服务端 Policy 才能正常放行。

---

## 1. 服务端目录结构

服务端机器建议目录：

```bash
mkdir -p /opt/netbird-server
cd /opt/netbird-server
```

最终结构示例：

```text
/opt/netbird-server/
├── docker-compose.yml
├── config.yaml
├── dashboard.env
└── .env                 # 可选，用来放域名、邮箱、DNSPOD key 等变量
```

你现在的服务端 compose 结构是：

```text
Traefik 反代 + 自动证书
Dashboard 面板
netbird-server combined server
```

这套结构可以继续用。

---

## 2. 服务端 docker-compose.yml

> 下面用占位符写法，实际部署时把域名、邮箱、DNSPOD key、端口换成你自己的值。

```yaml
services:
  traefik:
    image: traefik:v3.6
    container_name: netbird-traefik
    restart: unless-stopped
    networks:
      netbird:
        ipv4_address: 172.30.0.10
    command:
      - "--log.level=INFO"
      - "--accesslog=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=netbird"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.websecure.allowACMEByPass=true"
      - "--entrypoints.websecure.transport.respondingTimeouts.readTimeout=0"
      - "--entrypoints.websecure.transport.respondingTimeouts.writeTimeout=0"
      - "--entrypoints.websecure.transport.respondingTimeouts.idleTimeout=0"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--certificatesresolvers.letsencrypt.acme.email=你的邮箱@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.dnschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.dnschallenge.provider=dnspod"
      - "--certificatesresolvers.letsencrypt.acme.dnschallenge.delaybeforecheck=30"
      - "--certificatesresolvers.letsencrypt.acme.dnschallenge.resolvers=119.29.29.29:53,223.5.5.5:53,8.8.8.8:53"
      - "--serverstransport.forwardingtimeouts.responseheadertimeout=0s"
      - "--serverstransport.forwardingtimeouts.idleconntimeout=0s"
    ports:
      - "80:80"
      - "443:443"
      # 如果你想让外部用 https://fq.9017i.cc:57548 访问，
      # 可以在路由器做 57548 -> 服务端内网 443 的端口映射；
      # 或者改成：
      # - "57548:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - netbird_traefik_letsencrypt:/letsencrypt
    environment:
      - DNSPOD_API_KEY=你的DNSPOD_API_ID,你的DNSPOD_API_TOKEN
      - DNSPOD_PROPAGATION_TIMEOUT=300
      - DNSPOD_POLLING_INTERVAL=10
    logging:
      driver: "json-file"
      options:
        max-size: "500m"
        max-file: "2"

  dashboard:
    image: netbirdio/dashboard:latest
    container_name: netbird-dashboard
    restart: unless-stopped
    networks:
      - netbird
    env_file:
      - ./dashboard.env
    labels:
      - traefik.enable=true
      - traefik.http.routers.netbird-dashboard.rule=Host(`fq.9017i.cc`)
      - traefik.http.routers.netbird-dashboard.entrypoints=websecure
      - traefik.http.routers.netbird-dashboard.tls=true
      - traefik.http.routers.netbird-dashboard.tls.certresolver=letsencrypt
      - traefik.http.routers.netbird-dashboard.service=dashboard
      - traefik.http.routers.netbird-dashboard.priority=1
      - traefik.http.services.dashboard.loadbalancer.server.port=80
    logging:
      driver: "json-file"
      options:
        max-size: "500m"
        max-file: "2"

  netbird-server:
    image: netbirdio/netbird-server:latest
    container_name: netbird-server
    restart: unless-stopped
    networks:
      - netbird
    ports:
      - "3478:3478/udp"
    volumes:
      - netbird_data:/var/lib/netbird
      - ./config.yaml:/etc/netbird/config.yaml
    command:
      - "--config"
      - "/etc/netbird/config.yaml"
    labels:
      - traefik.enable=true
      - traefik.http.routers.netbird-grpc.rule=Host(`fq.9017i.cc`) && (PathPrefix(`/signalexchange.SignalExchange/`) || PathPrefix(`/management.ManagementService/`))
      - traefik.http.routers.netbird-grpc.entrypoints=websecure
      - traefik.http.routers.netbird-grpc.tls=true
      - traefik.http.routers.netbird-grpc.tls.certresolver=letsencrypt
      - traefik.http.routers.netbird-grpc.service=netbird-server-h2c
      - traefik.http.routers.netbird-grpc.priority=100
      - traefik.http.routers.netbird-backend.rule=Host(`fq.9017i.cc`) && (PathPrefix(`/relay`) || PathPrefix(`/ws-proxy/`) || PathPrefix(`/api`) || PathPrefix(`/oauth2`))
      - traefik.http.routers.netbird-backend.entrypoints=websecure
      - traefik.http.routers.netbird-backend.tls=true
      - traefik.http.routers.netbird-backend.tls.certresolver=letsencrypt
      - traefik.http.routers.netbird-backend.service=netbird-server
      - traefik.http.routers.netbird-backend.priority=100
      - traefik.http.services.netbird-server.loadbalancer.server.port=80
      - traefik.http.services.netbird-server-h2c.loadbalancer.server.port=80
      - traefik.http.services.netbird-server-h2c.loadbalancer.server.scheme=h2c
    logging:
      driver: "json-file"
      options:
        max-size: "500m"
        max-file: "2"

volumes:
  netbird_data:
  netbird_traefik_letsencrypt:

networks:
  netbird:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/24
          gateway: 172.30.0.1
```

如果客户端使用：

```text
https://fq.9017i.cc:57548
```

要保证外网访问这个端口能到 Traefik 的 `443`：

```text
做法 A：路由器端口映射
外网 57548 -> 服务端内网 443
compose 保持 443:443

做法 B：Docker 直接监听 57548
ports:
  - "57548:443"
```

---

## 3. 启动 NetBird 服务端

```bash
cd /opt/netbird-server

docker compose pull
docker compose up -d
```

查看状态：

```bash
docker compose ps

docker logs -f netbird-traefik
docker logs -f netbird-server
docker logs -f netbird-dashboard
```

测试 HTTPS：

```bash
curl -vk https://fq.9017i.cc/

# 如果你用 57548 作为外部访问端口
curl -vk https://fq.9017i.cc:57548/
```

---

## 4. 服务端面板设置

进入：

```text
https://fq.9017i.cc
```

或者：

```text
https://fq.9017i.cc:57548
```

### 4.1 创建 Setup Key

在 NetBird Dashboard：

```text
Setup Keys -> Create Setup Key
```

建议创建两个 key：

```text
home-gateway-key     给家里 v2rayA + NetBird 网关用
server-peer-key      给公司 / 远端服务器用
```

### 4.2 创建 Group

建议建这些组：

```text
home-gateway
office-service
```

分配方式：

```text
home-gateway:
  - v2rayA + NetBird Client 这台机器

office-service:
  - 10.0.1.103 这台机器
```

### 4.3 Policy

如果保留默认 Policy，全 Peer 互通一般没问题。

如果想收紧策略，可以建：

```text
Policy name:
  home-gateway-to-office-service

Source group:
  home-gateway

Destination group:
  office-service

Protocol:
  TCP

Port:
  8501

Direction:
  One-way
```

注意：

```text
NetBird 默认 Policy 放行的是 NetBird Peer 之间的流量。
如果 10.0.1.103 看到来源是 192.168.99.x，仍然可能被本机 NetBird ACL 丢掉。
所以 v2rayA 网关侧必须做 SNAT，让来源变成 home-gateway 的 NetBird IP。
```

---

## 5. v2rayA + NetBird Client 网关机目录

在 v2rayA 那台机器上：

```bash
mkdir -p /opt/v2raya-netbird
cd /opt/v2raya-netbird
```

目录结构：

```text
/opt/v2raya-netbird/
└── docker-compose.yml
```

---

## 6. v2rayA + NetBird Client docker-compose.yml

```yaml
services:
  netbird-client:
    image: netbirdio/netbird:latest
    container_name: netbird-client
    hostname: home-gateway-192.168.99.153
    restart: unless-stopped

    # 关键：让 wt0 出现在宿主机网络命名空间
    network_mode: host

    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
      - SYS_RESOURCE

    devices:
      - /dev/net/tun:/dev/net/tun

    environment:
      - NB_SETUP_KEY=你的NetBirdSetupKey
      - NB_MANAGEMENT_URL=https://fq.9017i.cc:57548
      # 可选：有些环境需要显式写 Dashboard/Admin 地址
      - NB_ADMIN_URL=https://fq.9017i.cc:57548
      - NB_LOG_LEVEL=info

    volumes:
      # 关键：持久化 NetBird client 状态，避免容器重建后反复注册成新 peer
      - netbird_client_data:/var/lib/netbird

  v2raya:
    image: mzz2017/v2raya
    container_name: v2raya
    restart: always
    privileged: true
    network_mode: host

    environment:
      V2RAYA_LOG_FILE: /tmp/v2raya.log
      V2RAYA_V2RAY_BIN: /usr/local/bin/xray
      V2RAYA_NFTABLES_SUPPORT: "on"
      # IPTABLES_MODE: legacy

    volumes:
      - /lib/modules:/lib/modules:ro
      - /etc/resolv.conf:/etc/resolv.conf
      - /etc/v2raya:/etc/v2raya

volumes:
  netbird_client_data:
```

启动：

```bash
cd /opt/v2raya-netbird

docker compose pull
docker compose up -d
```

查看：

```bash
docker ps

docker logs -f netbird-client
docker logs -f v2raya
```

---

## 7. 宿主机开启转发

在 v2rayA + NetBird Client 网关机上执行：

```bash
cat >/etc/sysctl.d/99-v2raya-netbird.conf <<'SYSCTL_EOF'
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
SYSCTL_EOF

sysctl --system
```

确认：

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
```

应该看到：

```text
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

---

## 8. nftables include 持久化 SNAT

目标：

```text
192.168.99.0/24 只要从 wt0 出去，就统一 masquerade。
不要写死 ip daddr 10.0.1.0/24。
这样以后新增 10.0.55.0/24、10.0.66.0/24 等远端网段，不用再改 SNAT。
```

### 8.1 主配置 `/etc/nftables.conf`

```bash
cp -a /etc/nftables.conf /root/nftables.conf.bak.$(date +%F-%H%M%S) 2>/dev/null || true

mkdir -p /etc/nftables.d

cat >/etc/nftables.conf <<'NFTCONF_EOF'
#!/usr/sbin/nft -f

# 不要 flush ruleset，避免清空 Docker / NetBird / v2rayA 动态规则
include "/etc/nftables.d/*.nft"
NFTCONF_EOF
```

### 8.2 创建 SNAT 文件

优先用这个版本：

```bash
cat >/etc/nftables.d/90-netbird-snat.nft <<'NFT_EOF'
# 只管理我们自己的 nb_nat 表，不碰 Docker / NetBird / v2rayA

destroy table ip nb_nat

table ip nb_nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        ip saddr 192.168.99.0/24 oifname "wt0" counter masquerade comment "SNAT 99 LAN clients to NetBird"
    }
}
NFT_EOF
```

检查语法：

```bash
nft -c -f /etc/nftables.conf
```

加载：

```bash
nft -f /etc/nftables.conf
```

查看：

```bash
nft list table ip nb_nat
```

应该看到：

```nft
table ip nb_nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 192.168.99.0/24 oifname "wt0" counter masquerade comment "SNAT 99 LAN clients to NetBird"
    }
}
```

### 8.3 如果 `destroy table` 不支持

如果检查语法时报错：

```text
syntax error, unexpected destroy
```

改成这个版本：

```bash
cat >/etc/nftables.d/90-netbird-snat.nft <<'NFT_EOF'
table ip nb_nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        ip saddr 192.168.99.0/24 oifname "wt0" counter masquerade comment "SNAT 99 LAN clients to NetBird"
    }
}
NFT_EOF

nft delete table ip nb_nat 2>/dev/null || true
nft -f /etc/nftables.conf
```

### 8.4 开机加载

```bash
systemctl enable nftables
```

不要随便执行：

```bash
systemctl restart nftables
```

除非你已经确认 `/etc/nftables.conf` 里面没有 `flush ruleset`。

---

## 9. v2rayA 里要注意的路由

v2rayA 里内网段不要走代理节点，要走 direct/bypass。

建议确保这些网段走直连：

```text
10.0.0.0/8
100.64.0.0/10
192.168.0.0/16
172.16.0.0/12
127.0.0.0/8
```

原因：

```text
10.0.0.0/8        公司 / 远端内网
100.64.0.0/10     NetBird overlay IP 段
192.168.0.0/16    家里 LAN
```

如果 v2rayA 把 `10.0.1.103` 丢到外网代理节点，肯定打不开。

---

## 10. 验证流程

### 10.1 NetBird Client 是否上线

在 v2rayA 网关机：

```bash
docker logs netbird-client --tail=100
ip -br addr | grep -E 'wt0|100\.104'
ip route | grep -E 'wt0|100\.104|10\.0'
```

如果容器里有 `netbird` 命令：

```bash
docker exec -it netbird-client netbird status --detail
```

### 10.2 v2rayA 是否正常

```bash
docker logs v2raya --tail=100
ss -lntup | grep -E '2017|52345|v2ray|xray|v2raya'
```

v2rayA Web UI 常见地址：

```text
http://网关机IP:2017
```

### 10.3 SNAT 是否命中

在 v2rayA 网关机执行：

```bash
nft list table ip nb_nat
```

然后手机 / 电脑通过 v2rayA 访问：

```text
http://10.0.1.103:8501
```

再看：

```bash
nft list table ip nb_nat
```

如果 `counter packets` 增加，说明 SNAT 命中。

### 10.4 在 v2rayA 网关机抓包

```bash
tcpdump -ni wt0 'host 10.0.1.103 and tcp port 8501'
```

正常能看到发往 `10.0.1.103:8501` 的包。

### 10.5 在 10.0.1.103 上抓包

```bash
tcpdump -ni wt0 'tcp port 8501'
```

如果 SNAT 正常，目标机应该看到来源是：

```text
100.104.x.x -> 10.0.1.103.8501
```

如果看到来源是：

```text
192.168.99.x -> 10.0.1.103.8501
```

说明 SNAT 没命中，目标机 NetBird ACL 可能会丢。

---

## 11. 访问 10.0.1.103:8501 不通时怎么查

### 11.1 查服务是否监听

在 `10.0.1.103`：

```bash
ss -lntp | grep ':8501'
```

应该是：

```text
0.0.0.0:8501
```

或者：

```text
10.0.1.103:8501
```

如果只监听：

```text
127.0.0.1:8501
```

外面访问不到。

例如 Streamlit 要这样启动：

```bash
streamlit run app.py --server.address 0.0.0.0 --server.port 8501
```

### 11.2 查 NetBird ACL 链

在 `10.0.1.103`：

```bash
nft -a list chain ip netbird netbird-acl-input-rules
nft -a list chain ip netbird netbird-acl-input-filter
```

如果只看到：

```nft
ip saddr @nb0000001 accept
```

而没有 `192.168.99.0/24`，这是正常的。

因为长期方案不是在 103 上允许 `192.168.99.0/24`，而是让 v2rayA 网关做 SNAT，让来源变成 `100.104.x.x`。

### 11.3 临时验证规则

如果想临时验证是不是 NetBird ACL 的问题，可以在 `10.0.1.103` 上执行：

```bash
nft insert rule ip netbird netbird-acl-input-rules \
  ip saddr 192.168.99.0/24 tcp dport 8501 counter accept
```

如果加完立刻能打开，说明问题就是源地址没 SNAT，被 NetBird ACL 丢了。

这条只适合验证，不适合长期用；NetBird 重启后可能会覆盖。

---

## 12. 新增远端网段时要不要改 nft？

如果你用的是：

```nft
ip saddr 192.168.99.0/24 oifname "wt0" masquerade
```

那么新增这些都不用改 nft：

```text
10.0.55.0/24
10.0.66.0/24
172.16.50.0/24
其他 NetBird 可达网段
```

你只需要在 NetBird 面板里保证路由和策略正确。

不要再写死这种：

```nft
ip saddr 192.168.99.0/24 ip daddr 10.0.1.0/24 oifname "wt0" masquerade
```

否则以后每增加一个远端网段，都要继续补 `ip daddr`。

---

## 13. 新增 NetBird 客户端时怎么处理

### 13.1 新增普通客户端

不用改 nft。

只要客户端是正常 NetBird Peer，默认 policy 或你自己配置的 policy 放行即可。

### 13.2 新增远端 LAN 网段

比如新增：

```text
10.0.55.0/24
```

你需要在 NetBird 面板里添加 Network Route / Network Resource：

```text
Network:
  10.0.55.0/24

Routing Peer:
  对应远端网关机器

Masquerade:
  Enabled

Access Control:
  按需分配 group / policy
```

v2rayA 网关机本地 nft 不用改，因为它只看出口是不是 `wt0`。

---

## 14. 常见问题

### 问题 1：NetBird 客户端互相能访问，但手机连 v2rayA 访问 103 不行

原因通常是：

```text
NetBird Peer 互访来源是 100.104.x.x，能被默认 policy 放行。
手机经 v2rayA 访问时，来源可能还是 192.168.99.x。
目标机 NetBird ACL 不认 192.168.99.x，所以 drop。
```

解决：

```text
在 v2rayA + NetBird Client 网关机上做：
192.168.99.0/24 -> oifname wt0 masquerade
```

### 问题 2：普通 INPUT 链里 accept 了 wt0，为什么还是不通？

因为 NetBird 自己还有一条 input hook：

```nft
iifname "wt0" jump netbird-acl-input-rules
iifname "wt0" drop
```

普通 `table ip filter INPUT` 里 accept 了，不代表 NetBird 自己的 base chain 不会继续 drop。

### 问题 3：我能不能把 SNAT 写进 `/etc/nftables.conf`？

可以，但不要写 `flush ruleset`。

推荐：

```nft
#!/usr/sbin/nft -f
include "/etc/nftables.d/*.nft"
```

然后把自己的规则放：

```text
/etc/nftables.d/90-netbird-snat.nft
```

### 问题 4：v2rayA 重启会不会影响？

v2rayA 会维护自己的 nft 表。你的 SNAT 表叫：

```text
ip nb_nat
```

一般不冲突。

如果发现规则没了，重新加载：

```bash
nft -f /etc/nftables.conf
nft list table ip nb_nat
```

### 问题 5：Docker 重启会不会影响？

Docker 会维护 `ip nat`、`ip filter` 里面的 DOCKER 链。你的规则在单独的：

```text
table ip nb_nat
```

通常不冲突。

不要执行带 `flush ruleset` 的 nftables 配置。

---

## 15. 最终检查清单

服务端：

```bash
cd /opt/netbird-server

docker compose ps
docker logs netbird-server --tail=100
docker logs netbird-traefik --tail=100
curl -vk https://fq.9017i.cc:57548/
```

v2rayA 网关机：

```bash
cd /opt/v2raya-netbird

docker compose ps
docker logs netbird-client --tail=100
docker logs v2raya --tail=100

ip -br addr | grep wt0
nft list table ip nb_nat
sysctl net.ipv4.ip_forward
```

访问测试：

```bash
# 网关机上
curl -v http://10.0.1.103:8501/

# 手机/电脑连 v2rayA 后
浏览器打开 http://10.0.1.103:8501/
```

抓包确认：

```bash
# v2rayA 网关机
tcpdump -ni wt0 'host 10.0.1.103 and tcp port 8501'

# 10.0.1.103
tcpdump -ni wt0 'tcp port 8501'
```

目标机看到来源是 `100.104.x.x`，说明 SNAT 正常。

---

## 16. 备份命令

部署完成后备份当前规则：

```bash
mkdir -p /root/nft-backup
nft list ruleset > /root/nft-backup/running-ruleset-$(date +%F-%H%M%S).nft
```

备份 compose：

```bash
tar czf /root/netbird-v2raya-compose-backup-$(date +%F-%H%M%S).tar.gz \
  /opt/netbird-server \
  /opt/v2raya-netbird \
  /etc/nftables.conf \
  /etc/nftables.d/90-netbird-snat.nft
```

---

## 17. 一句话总结

```text
NetBird 服务端负责控制面板、Policy、Route。
v2rayA 负责手机/电脑入口和出国流量。
NetBird Client 负责打通 overlay。
nft 的 nb_nat 负责把 192.168.99.0/24 伪装成网关机 NetBird IP。

以后新增 NetBird 客户端或远端网段，不改本地 SNAT；只在 NetBird 面板里配置 Route / Policy。
```
