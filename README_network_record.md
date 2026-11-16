# NS16 网口转发与核对记录

本文保留地址转换配置及 2026-11-16 的核对结果。对接步骤见 [README_network.md](README_network.md)。

2026-11-16 在 macOS 上核对过。板子可以打开 `http://www.baidu.com`（HTTP 200）和 `http://www.gstatic.com/generate_204`（HTTP 204）。

## 1. 板上已经生效的配置

板侧 `eth0` 地址保持不变。NetworkManager 连接名为 `eth0`，IPv4 为手动，自动连接，忽略自动 DNS，IPv6 关闭。

| 项目 | 取值 |
| --- | --- |
| 登录 | `ssh nvidia@192.168.88.2`，密码 `nvidia`，`sudo` 密码同为 `nvidia` |
| 主机名 | `ubuntu` |
| 系统 | Ubuntu 20.04，aarch64，内核 `5.10.216-tegra` |
| 网口 | `eth0`，别名 `enP8p1s0`，MAC `4c:bb:47:ac:01:5f` |
| 地址 | `192.168.88.2/24` |
| 网关 | `192.168.88.1`，即操作电脑上接入该板的网卡 |
| DNS | `223.5.5.5` |

`l4tbr0` 是 `192.168.55.1/24`，`ipv4.never-default` 为 yes，当前无载波。路由表里可能还有一条 metric 更大、状态为 `linkdown` 的默认路由 `192.168.55.100`。上网走 `eth0` 上 metric 更小的 `192.168.88.1`。

板上没有 `curl`。检查网页用第 5 节的 `python3`。

## 2. 接线

1. 千兆 USB 以太网卡插到操作电脑，网线另一端接到板子的 `eth0`（RJ45）。链路正常时为 1000baseT 全双工。
2. 电脑上找出刚出现载波的那块网卡。接口名会随插拔变化，以当时的设备名为准。2026-11-16 这次是 macOS 网络服务 “USB 10/100/1000 LAN 2”，设备 `en8`，适配器 MAC `00:e0:4c:cb:e9:11`，对端 MAC `4c:bb:47:ac:01:5f`。

机器狗的 USB 网线网段为 `192.168.1.0/24`。公司局域网也使用 `192.168.1.0/24`。板子 `eth0` 保持 `192.168.88.2/24`。`192.168.1.100` 不配置到 `eth0`，机器狗所在网段不作为默认路由。

## 3. 操作电脑的直连地址

接入板子的那块网卡使用静态地址 `192.168.88.1/24`。

该网卡不填写路由器。操作电脑的上网仍走原来的 Wi-Fi 或有线，全局默认路由保持原来的网关。下文把电脑用来上网的接口记为 `IF_WAN`。2026-11-16 核对时 `IF_WAN` 是 `en0`。

不启用操作系统自带的互联网共享（Internet Connection Sharing）。该功能会改写 USB 网卡的网段（常见为 `192.168.2.0/24` 或 `192.168.137.0/24`），与板子网关 `192.168.88.1` 不一致。

### 3.1 macOS

系统设置 → 网络 → 该 USB 网卡：

- 配置 IPv4：手动
- IP 地址：`192.168.88.1`
- 子网掩码：`255.255.255.0`
- 路由器：留空

临时在当前接口上生效时，把 `en8` 换成第 2 节查出的设备名：

```sh
sudo ifconfig en8 inet 192.168.88.1 netmask 255.255.255.0
```

`networksetup -setmanual` 的参数里包含路由器。`10.21.31.1` 不填入这块网卡的路由器。系统配置里如果还留着以前的路由器 `10.21.31.1`，会在这块网卡上多出一条只作用于本接口的默认路由。`netstat -rn -f inet` 中类似：

```text
default    10.21.31.1    UGScIg    en8
```

标志里的 `I` 表示该路由只作用于这块网卡，不替换全局默认路由。板子转发出来的流量仍走全局默认路由。`10.21.31.1` 不用作板子网关。若它变成不带 `I` 的全局默认路由，删除该接口上的默认路由后再核对：

```sh
sudo route -n delete -ifscope en8 default
netstat -rn -f inet | awk '/^default|192\.168\.88/'
ping -c 2 192.168.88.2
```

全局 `default` 指向操作电脑用来上网的接口。`192.168.88` 指向 USB 网卡。

### 3.2 Linux

把 `enx` 换成插上后出现载波的接口。该接口不添加默认路由。

```sh
sudo ip link set enx up
sudo ip addr replace 192.168.88.1/24 dev enx
ping -c 2 192.168.88.2
```

### 3.3 Windows

把 USB 网卡设为 `192.168.88.1`，前缀长度 `24`，默认网关留空。地址在“网络连接”的 IPv4 属性里填写，或用管理员 PowerShell，把 `以太网` 换成该网卡在系统里的名称：

```powershell
New-NetIPAddress -InterfaceAlias "以太网" -IPAddress 192.168.88.1 -PrefixLength 24
```

## 4. 转发与源地址转换

板子的默认路由指向 `192.168.88.1`。操作电脑打开 IPv4 转发，把来自 `192.168.88.0/24` 的流量从 `IF_WAN` 送出，并做源地址转换。板侧不设置 `http_proxy` 或 `https_proxy`。

先确认上网接口：

```sh
route -n get default
```

Linux 上用 `ip route show default`。输出里的接口就是 `IF_WAN`。

### 4.1 macOS

`/etc/pf.conf` 里已有 `nat-anchor "com.apple/*"`。规则挂在这个锚点下，锚点名为 `com.apple/boardnat`，不必修改 `pf.conf`。

写入 `/etc/pf.anchors/com.boardnat`。下面的 `en0` 换成自己的 `IF_WAN`：

```text
nat on en0 inet from 192.168.88.0/24 to any -> (en0)
pass quick inet from 192.168.88.0/24 to any keep state
pass quick inet from any to 192.168.88.0/24 keep state
```

写入 `/usr/local/sbin/board-nat.sh` 并 `chmod 755`。文件属主为 root：

```sh
#!/bin/sh
sysctl -w net.inet.ip.forwarding=1
pfctl -E
pfctl -a com.apple/boardnat -f /etc/pf.anchors/com.boardnat
```

开机执行该脚本。`/Library/LaunchDaemons/com.board.nat.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.board.nat</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/sh</string>
    <string>/usr/local/sbin/board-nat.sh</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <false/>
</dict>
</plist>
```

```sh
sudo launchctl bootstrap system /Library/LaunchDaemons/com.board.nat.plist
```

该 plist 已经加载过时，直接再执行一次 `/usr/local/sbin/board-nat.sh`。2026-11-16 这台 Mac 上对应的守护进程标签是 `com.charlescai.board-nat`，调用的是同一个脚本。

### 4.2 Linux

把 `enx` 换成直连板子的接口，把 `IF_WAN` 换成上网接口。

```sh
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o IF_WAN -j MASQUERADE
sudo iptables -A FORWARD -i enx -o IF_WAN -j ACCEPT
sudo iptables -A FORWARD -i IF_WAN -o enx -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

上述规则在重启后消失。需要长期使用时，用该系统现有的方式保存，例如 `iptables-persistent` 或发行版自己的防火墙配置。

### 4.3 Windows

USB 网卡已经是 `192.168.88.1/24` 且没有默认网关之后，用管理员 PowerShell：

```powershell
New-NetNat -Name BoardNat -InternalIPInterfaceAddressPrefix 192.168.88.0/24
```

若系统提示不支持 `New-NetNat`，这台 Windows 没有可用的 WinNAT，改用第 4.1 节或第 4.2 节的系统。

## 5. 登录并检查上网

```sh
ssh nvidia@192.168.88.2
```

登录后在板上执行：

```sh
ping -c 1 -W 2 192.168.88.1
ping -c 1 -W 2 223.5.5.5
python3 -c "import urllib.request; print(urllib.request.urlopen('http://www.baidu.com', timeout=12).status)"
```

网关应通，`223.5.5.5` 应通，百度应返回 `200`。

操作电脑若只通过本机代理软件访问外网，且该软件不转发其他主机送来的数据包，板子无法经该代理外出。本次核对未在板上配置代理。地址转换从 `en0` 发出后，板上可以直接访问上述地址。

## 6. 2026-11-16 核对记录

| 项目 | 结果 |
| --- | --- |
| 电脑直连口 | 服务 “USB 10/100/1000 LAN 2”，`en8` = `192.168.88.1/24`，1000baseT 全双工 |
| 板子 `eth0` | `192.168.88.2/24`，网关 `192.168.88.1`，DNS `223.5.5.5` |
| 电脑转发 | `net.inet.ip.forwarding` 为 1 |
| 地址转换 | `192.168.88.0/24` 从 `en0` 做源地址转换 |
| 当时的全局默认路由 | `en0`，网关 `172.20.10.1` |
| USB 网卡上的残留路由 | `10.21.31.1`，标志带 `I`，未用作板子网关 |
| 板上 ping `192.168.88.1` | 通 |
| 板上 ping `223.5.5.5` | 通 |
| 板上 HTTP | 百度 200，`generate_204` 为 204 |
