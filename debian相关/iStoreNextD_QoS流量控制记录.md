# Debian / iStoreNextD QoS 流量控制记录

## 一、背景

在 iStoreNextD 上检查当前 Linux QoS / 队列调度配置，目标是优化公网出口 `ppp0` 的上传延迟，减少上传满速时导致的：

- SSH 卡顿
- 网页打开慢
- 游戏延迟升高
- OpenVPN 访问不稳定
- Docker / NAS 上传抢占带宽

系统环境：

- Debian 13
- PPPoE 拨号接口：`ppp0`
- nftables
- Docker
- 多内网网段
- OpenVPN
- Hairpin NAT

---

## 二、查看当前 QoS 配置

执行：

```bash
tc qdisc show
```

初始关键输出：

```txt
qdisc fq_codel 0: dev eth1 root
qdisc fq_codel 0: dev eth2 root
qdisc fq 0: dev ppp0 root
```

说明：

| 接口 | 队列类型 | 说明 |
|---|---|---|
| eth1 | fq_codel | 内网接口，已有低延迟队列 |
| eth2 | fq_codel | 内网接口，已有低延迟队列 |
| ppp0 | fq | 公网 PPPoE 出口，只有公平队列 |

---

## 三、当前配置分析

`eth1`、`eth2` 使用的是：

```txt
fq_codel
```

这是现代 Linux 常用的低延迟队列，能减少队列堆积。

但公网出口 `ppp0` 使用的是：

```txt
fq
```

`fq` 是 Fair Queue，特点是：

- 可以做连接公平
- 支持 TCP pacing
- 比传统 FIFO 好
- 但不主动按带宽控速
- 对 Bufferbloat 的控制不如 CAKE

因此，真正需要优化的是：

```txt
ppp0 上传方向
```

---

## 四、为什么要优化 ppp0

家宽环境中最容易造成卡顿的是上传方向。

例如：

- NAS 上传文件
- Docker 推送镜像
- OpenVPN 传输
- BT / 下载器上传
- 内网设备跑满上行

一旦上传把运营商侧或本机队列塞满，就会出现：

```txt
上传满速 -> 队列堆积 -> ping 延迟暴涨 -> 交互卡顿
```

这类问题通常称为：

```txt
Bufferbloat
```

---

## 五、CAKE 是什么

CAKE 是 Linux 中非常适合家宽出口的队列调度器。

它可以：

- 主动控制带宽
- 减少队列延迟
- 按内网设备公平分配
- 支持 NAT 环境
- 改善上传满速时的延迟
- 比普通 `fq` 更适合路由器出口

---

## 六、替换 ppp0 队列

最开始尝试删除默认 qdisc：

```bash
tc qdisc del dev ppp0 root
```

返回：

```txt
Error: Cannot delete qdisc with handle of zero.
```

原因：

当前 `ppp0` 上的 `fq` 是默认 qdisc，handle 为 `0:`，不能直接删除。

正确方式是使用：

```bash
tc qdisc replace
```

---

## 七、启用 CAKE

执行：

```bash
tc qdisc replace dev ppp0 root cake bandwidth 45mbit nat dual-srchost
```

参数说明：

| 参数 | 含义 |
|---|---|
| cake | 使用 CAKE 队列 |
| bandwidth 45mbit | 将上传控制在 45 Mbit/s |
| nat | 识别 NAT 后面的真实内网 IP |
| dual-srchost | 按源 IP 公平分配上传带宽 |

---

## 八、45Mbit 换算

```txt
45 Mbit/s ÷ 8 = 5.625 MB/s
```

也就是说：

```txt
45Mbit ≈ 5.6 MB/s
```

如果真实上传带宽是 50M，那么设置为 45Mbit 比较合理。

QoS 通常需要比真实带宽略低一些，一般设置为：

```txt
真实上传带宽 × 0.9
```

这样 CAKE 才能掌握排队控制权。

---

## 九、查看 CAKE 是否生效

执行：

```bash
tc qdisc show dev ppp0
```

或者：

```bash
tc -s qdisc show dev ppp0
```

成功后看到：

```txt
qdisc cake 8002: root refcnt 2 bandwidth 45Mbit diffserv3 dual-srchost nat
```

说明 CAKE 已经生效。

---

## 十、实际运行状态

执行：

```bash
tc -s qdisc show dev ppp0
```

观察到：

```txt
qdisc cake 8002: root refcnt 2 bandwidth 45Mbit diffserv3 dual-srchost nat nowash no-ack-filter split-gso rtt 100ms raw overhead 0
Sent 157812206 bytes 1041119 pkt (dropped 52, overlimits 1453757 requeues 0)
backlog 0b 0p requeues 0
```

关键指标：

| 指标 | 当前值 | 含义 |
|---|---|---|
| bandwidth | 45Mbit | 上传限速 45M |
| backlog | 0b | 当前没有队列积压 |
| drops | 52 | CAKE 主动丢包控延迟，正常 |
| overlimits | 1453757 | 控速命中次数，正常 |
| pk_delay | 870us | 峰值排队延迟低 |
| av_delay | 259us | 平均排队延迟低 |
| marks | 5 | ECN 标记，正常 |

---

## 十一、结果判断

当前状态很好：

```txt
backlog 0b
pk_delay 870us
av_delay 259us
drops 52
```

说明：

- 队列没有堆积
- 延迟控制很好
- CAKE 正在正常控速
- 上传方向已经进入低延迟状态

`overlimits` 增长是正常现象，表示 CAKE 正在按照 45Mbit 限速。

`drops` 少量增长也是正常现象，表示 CAKE 主动丢包避免队列膨胀。

---

## 十二、持续观察命令

可以实时观察 CAKE 状态：

```bash
watch -n 1 'tc -s qdisc show dev ppp0'
```

重点看：

```txt
backlog
drops
marks
pk_delay
av_delay
overlimits
```

---

## 十三、测试建议

一边跑上传测速，一边 ping：

```bash
ping 223.5.5.5
```

或者：

```bash
ping 8.8.8.8
```

如果上传满速时 ping 仍然稳定，说明 QoS 效果良好。

---

## 十四、恢复原配置

如果需要恢复原来的 `fq`：

```bash
tc qdisc replace dev ppp0 root fq
```

查看：

```bash
tc qdisc show dev ppp0
```

如果看到：

```txt
qdisc fq
```

说明已经恢复。

---

## 十五、当前结论

当前 `ppp0` 已经成功切换为：

```txt
CAKE + 45Mbit + NAT + dual-srchost
```

适合当前环境：

- PPPoE
- 多内网设备
- Docker
- OpenVPN
- NAT
- 家宽上传优化

当前 QoS 配置属于比较合理的家宽出口低延迟方案。
