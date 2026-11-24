# NS16 网口转发记录

对接步骤见 [README_network.md](README_network.md)。本文记录 2026-11-24 起生效的地址和转发。`192.168.88.0/24` 已从板子删除，不再由操作电脑做源地址转换。

## 1. 板上配置

NetworkManager 连接名为 `eth0`，IPv4 为手动，自动连接，IPv6 关闭。

| 项目 | 取值 |
| --- | --- |
| 登录 | `ssh nvidia@192.168.1.110`，密码 `nvidia`，`sudo` 密码同为 `nvidia` |
| 主机名 | `ubuntu` |
| 系统 | Ubuntu 20.04，aarch64，内核 `5.10.216-tegra` |
| 网口 | `eth0`，别名 `enP8p1s0`，MAC `4c:bb:47:ac:01:5f` |
| 地址 | `192.168.1.110/24` |
| 网关 | `192.168.1.106`，路由 metric `50` |
| DNS | `223.5.5.5` |

`l4tbr0` 是 `192.168.55.1/24`，当前无载波。路由表里还有一条 metric `32766`、状态为 `linkdown` 的默认路由 `192.168.55.100`。上网走 `eth0` 上 metric `50` 的 `192.168.1.106`。

板上没有 `curl`。检查网页用第 4 节的 `python3`。

## 2. 记录主机上的转发

记录主机为 `ysc@192.168.1.106`，主机名 `ysc-perception`。机器人内网网口是 `enp3s0`。手机热点接在 USB 网卡上，接口名随插拔变化；2026-11-24 核对时为 `enx1ef9d55947bd`，地址 `172.20.10.2/28`，网关 `172.20.10.1`。

`/etc/sysctl.d/99-ns16-hotspot.conf` 写入 `net.ipv4.ip_forward=1`。`/etc/NetworkManager/dispatcher.d/99-ns16-nat` 在网卡变为 up 时保证同一条规则存在：

```sh
iptables -t nat -A POSTROUTING -s 192.168.1.110/32 ! -o enp3s0 -j MASQUERADE
```

只转换来自板子的流量。出站网卡是路由选中的外部网卡，不写死热点接口名。`enp3s0` 上发给狗内其他地址的包不做转换。

该主机另有一条 metric `20100` 的默认路由 `192.168.137.1`。热点路由的 metric 更小，出网用热点。

## 3. 操作电脑直连

直连板子的那块网卡使用 `192.168.1.20/24`，路由器留空。不要把公司网所在的网卡改成这个地址。登录：

```sh
ping -c 3 192.168.1.110
ssh nvidia@192.168.1.110
```

这根网线不承担板子的上网。板子的默认网关仍是 `192.168.1.106`，只有 `eth0` 接在机器狗上时才能出去。

## 4. 核对

2026-11-24 在板子上核对。`ip route get 223.5.5.5` 的下一跳为 `192.168.1.106`，源地址为 `192.168.1.110`。`eth0` 上不再有 `192.168.88.2`。

```sh
python3 - << 'PY'
import socket, urllib.request
socket.setdefaulttimeout(15)
print(socket.getaddrinfo("www.baidu.com", 443)[0][4][0])
print(urllib.request.urlopen("https://www.baidu.com", timeout=15).status)
PY
```

域名能解析，`https://www.baidu.com` 返回 200。
