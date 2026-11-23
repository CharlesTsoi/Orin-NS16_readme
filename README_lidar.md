# NS16 订阅 X30 雷达点云

本文说明在 NS16 本机订阅 X30 雷达点云，并确认消息已经到达。系统为 Ubuntu 20.04、ROS Noetic，无图形界面。确认方式是话题类型、频率和一帧中的点数字段。

使用本机数据包 `LIO_2026-11-13-16-32-56.bag`。其中的雷达话题为 `/livox/lidar`，消息类型为 `livox_ros_driver2/CustomMsg`。发布与订阅使用同一个 ROS 主节点，地址为 `127.0.0.1:11311`。

## 1. 登录

网线直连时：

```sh
ssh nvidia@192.168.88.2
```

经机器狗网络登录时：

```sh
ssh nvidia@192.168.1.110
```

| 项目 | 取值 |
| --- | --- |
| 用户名 | `nvidia` |
| 密码 | `nvidia` |
| 主机名 | `ubuntu` |

登录后工作目录中的数据包路径为 `/home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag`。

## 2. 每个终端的环境

发布和订阅使用不同终端时，每个终端都执行下列命令。未 source 工作空间的终端无法加载 `livox_ros_driver2/CustomMsg`。未导出下列变量的终端会使用另一个主节点地址或另一块网卡地址。

```sh
set +u
source /opt/ros/noetic/setup.bash
source /home/nvidia/ligo_ws/devel/setup.bash
set -u
export ROS_MASTER_URI=http://127.0.0.1:11311
export ROS_IP=127.0.0.1
unset ROS_HOSTNAME
```

消息定义来自 `/home/nvidia/ligo_ws`。核对类型指纹：

```sh
rosmsg md5 livox_ros_driver2/CustomMsg
```

输出为：

```text
e4d6829bdfe657cb6c21a746c86b21a6
```

该值与数据包内 `/livox/lidar` 的指纹一致。输出不同或命令失败时，订阅端无法解析这一帧，话题上不会出现点坐标。

`ROS_HOSTNAME` 必须为空。已启动的 `roscore` 和 `rosbag` 保留启动当时的地址；修改环境变量之后重新启动这两项。

播放本机数据包时，点云不经过 `eth0`。`ROS_IP` 使用 `127.0.0.1`。

## 3. 话题

`rosbag info /home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag` 的结果如下。

| 项目 | 取值 |
| --- | --- |
| 文件大小 | 1,767,271,815 字节 |
| 时长 | 114.5 s |
| 点云话题 | `/livox/lidar` |
| 点云类型 | `livox_ros_driver2/CustomMsg` |
| 点云条数 | 1145，约 10 Hz |
| IMU 话题 | `/imu/data`，`sensor_msgs/Imu`，22899 条，约 200 Hz |
| 里程计话题 | `/leg_odom`，`nav_msgs/Odometry`，22899 条 |

雷达数据在 `/livox/lidar`。`/imu/data` 是机体 IMU。`/leg_odom` 是里程计。

首帧点云：

| 字段 | 取值 |
| --- | --- |
| `header.frame_id` | `livox_frame` |
| `lidar_id` | `0` |
| `point_num` | `79968` |
| `points` 长度 | `79969` |
| `line` | `0`、`1`、`2`、`3` |
| `offset_time` | 纳秒，首帧从 `0` 到 `100334567` |

`points` 的长度比 `point_num` 多 1。读取坐标时使用下标 `1` 到 `point_num - 1`。每个点含 `x`、`y`、`z`、`reflectivity`、`tag`、`line`、`offset_time`。

该文件中的点云话题只有 `/livox/lidar`。订阅类型使用 `livox_ros_driver2/CustomMsg`。

## 4. 播放并订阅

确认点云时不要执行 `scripts/run_1113.sh` 或 `scripts/run_x30.sh`。二者在播放结束后关闭主节点。播放期间主节点也只监听 `127.0.0.1`。

本节不设置 `/use_sim_time`。该参数为 true 且没有 `/clock` 时，新节点停在初始化，终端没有后续输出。

先确认文件：

```sh
ls -lh /home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag
rosbag info /home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag
```

`topics` 中应有 `/livox/lidar`，类型为 `livox_ros_driver2/CustomMsg`，条数为 1145。

终端 A：

```sh
roscore
```

终端 B，在终端 A 打印 `started core service` 之后执行：

```sh
rosbag play -r 1.0 /home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag
```

终端 C，在终端 B 开始播放之后执行：

```sh
rostopic list
rostopic info /livox/lidar
rostopic hz /livox/lidar
rostopic echo -n 1 /livox/lidar/point_num
rostopic echo -n 1 /livox/lidar/header
```

合格输出：

| 命令 | 结果 |
| --- | --- |
| `rostopic list` | 含 `/livox/lidar` |
| `rostopic info /livox/lidar` | 类型为 `livox_ros_driver2/CustomMsg`，发布者地址以 `http://127.0.0.1:` 开头 |
| `rostopic hz /livox/lidar` | 平均频率约 10 Hz |
| `point_num` | `79968` |
| `header.frame_id` | `livox_frame` |

整帧 `rostopic echo /livox/lidar` 会打印约 8 万个点，终端会长时间滚动。确认收到数据时只打印 `point_num` 和 `header`。

打印一个点的坐标：

```sh
python3 - << 'PY'
import rospy
from livox_ros_driver2.msg import CustomMsg

def cb(msg):
    p = msg.points[1]
    print("frame_id=%s lidar_id=%s point_num=%s points=%s" % (
        msg.header.frame_id, msg.lidar_id, msg.point_num, len(msg.points)))
    print("point[1] x=%.3f y=%.3f z=%.3f reflectivity=%s tag=%s line=%s offset_time=%s" % (
        p.x, p.y, p.z, p.reflectivity, p.tag, p.line, p.offset_time))
    rospy.signal_shutdown("shown")

rospy.init_node("x30_cloud_check", anonymous=True)
rospy.Subscriber("/livox/lidar", CustomMsg, cb, queue_size=1)
rospy.spin()
PY
```

该命令在播放进行中执行。打印一行 `point_num` 和一行坐标后退出。`line` 为 0 到 3 中的一个值。`offset_time` 为纳秒。

程序内订阅使用同一话题和同一类型：

```python
rospy.Subscriber("/livox/lidar", CustomMsg, callback, queue_size=1)
```

`CustomMsg` 来自 `livox_ros_driver2.msg`。节点进程使用第 2 节的环境。

播放约 114.5 s 后结束，话题停止更新。需要再次查看时，重新执行终端 B 的播放命令。结束后在终端 A 按 Ctrl-C 停止 `roscore`。

## 5. 在另一台电脑上订阅

第 4 节的主节点和发布地址都是 `127.0.0.1`。操作电脑访问不到该地址。`ROS_MASTER_URI` 仍指向 `127.0.0.1` 时，操作电脑无法连接主节点。主节点改到 `192.168.88.2` 而发布者仍通告 `127.0.0.1` 时，操作电脑可以列出话题，频率为 0。

订阅端位于操作电脑时，先结束 NS16 上已有的 `roscore` 和 `rosbag`，再改用下面的地址。操作电脑网卡为 `192.168.88.1/24`，NS16 的 `eth0` 为 `192.168.88.2/24`。两台机器都能 `ping` 通对端之后再启动。

NS16 上每个相关终端：

```sh
set +u
source /opt/ros/noetic/setup.bash
source /home/nvidia/ligo_ws/devel/setup.bash
set -u
export ROS_MASTER_URI=http://192.168.88.2:11311
export ROS_IP=192.168.88.2
unset ROS_HOSTNAME
```

操作电脑上每个相关终端：

```sh
export ROS_MASTER_URI=http://192.168.88.2:11311
export ROS_IP=192.168.88.1
unset ROS_HOSTNAME
```

`roscore` 和 `rosbag play` 仍在 NS16 上执行，命令与第 4 节相同。操作电脑上执行 `rostopic hz` 和 `rostopic echo`。操作电脑上的 `livox_ros_driver2/CustomMsg` 指纹同样为 `e4d6829bdfe657cb6c21a746c86b21a6`。

发布者地址应为 `http://192.168.88.2:`。RViz 的 PointCloud2 显示项只接受 `sensor_msgs/PointCloud2`，不能显示 `/livox/lidar`。

## 6. 订阅 `sensor_msgs/PointCloud2`

`LIO_2026-11-13-16-32-56.bag` 提供的是 `livox_ros_driver2/CustomMsg`。需要 `sensor_msgs/PointCloud2` 时，改播本机另一份录制：

```text
/home/nvidia/x30_ligo_test/mapping_13.bag
```

原文件名为 `mapping_20261013_222228_13.bag`，时长 29.3 s。环境仍用第 2 节。播放：

```sh
rosbag play -r 1.0 /home/nvidia/x30_ligo_test/mapping_13.bag
```

已解析点云话题为 `/lidar_points`，类型 `sensor_msgs/PointCloud2`，294 条，约 10 Hz。首帧 `width = 79872`，`height = 1`，坐标系 `lidar_link`，`point_step = 26`。

| 字段 | 偏移（字节） | 类型 |
| --- | --- | --- |
| `x` | 0 | `float32` |
| `y` | 4 | `float32` |
| `z` | 8 | `float32` |
| `intensity` | 12 | `float32` |
| `tag` | 16 | `uint8` |
| `line` | 17 | `uint8` |
| `timestamp` | 18 | `float64` |

确认命令：

```sh
rostopic info /lidar_points
rostopic hz /lidar_points
rostopic echo -n 1 /lidar_points/width
rostopic echo -n 1 /lidar_points/header
```

`width` 为 `79872`，`frame_id` 为 `lidar_link`，频率约 10 Hz。该话题可以直接用 `sensor_msgs/PointCloud2` 订阅。

同一文件里还有 `/livox/lidar`。两份点云来自同一次测量，点数不完全相同。只订阅其中一路。

`timestamp` 在单帧内的跨度约为 100，单位是毫秒。`line` 的最大值为 15。这一份是多台设备合并后的扫描，与第 3 节单台 Mid-360 的 `line` 取值 0–3 不同。

## 7. 送入 LIGO 时的话题

只核对点云时停在第 4 节或第 6 节。节点 `ligo_mapping` 按 `lidar_type` 选择回调，两路不能同时作为 `common/lid_topic`。

| 数据 | `common/lid_topic` | `lidar_type` | 消息类型 |
| --- | --- | --- | --- |
| 第 3 节单台 Mid-360 | `/livox/lidar` | `1` | `livox_ros_driver2/CustomMsg` |
| 第 6 节已解析点云 | `/lidar_points_ligo` | `4` | `sensor_msgs/PointCloud2` |

`lidar_type = 1` 时，IMU 话题为 `/imu/data`，`mapping/acc_norm` 为 `9.81`。播放命令改为：

```sh
rosparam set /use_sim_time true
rosbag play --clock -r 1.0 /home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag
```

`/use_sim_time` 在 `roscore` 启动之后、节点启动之前设置。播放命令带 `--clock`。这一路不启动 `scripts/cloud_relay.py`。

`lidar_type = 4` 时，先播放 `mapping_13.bag`，并由 `scripts/cloud_relay.py` 把 `/lidar_points` 转为 `/lidar_points_ligo`。转发后的字段为 `x,y,z,intensity`（`float32`）、`timestamp`（`float64`，本帧内毫秒偏移）、`ring`（`uint16`，来自 `line`）。`timestamp_unit` 为 `1`。IMU 在播放时重映射：

```sh
rosbag play --clock -r 1.0 /home/nvidia/x30_ligo_test/mapping_13.bag \
  /livox/imu_192_168_2_202:=/livox/imu
```

这一路的 `common/imu_topic` 为 `/livox/imu`，`mapping/acc_norm` 为 `1.0`。`/imu/data` 与该取值不能混用。

LIGO、转发和播放进程的 `ROS_MASTER_URI` 与 `ROS_IP` 保持一致。

## 8. 收不到消息时的核对

播放进行中逐项执行。前一项未合格时，先处理后一项。

| 检查 | 合格结果 | 不合格时的处理 |
| --- | --- | --- |
| `echo "$ROS_MASTER_URI"` | `http://127.0.0.1:11311` | 按第 2 节重新导出，并重启 `roscore` 与 `rosbag play` |
| `echo "$ROS_IP"` | `127.0.0.1` | 同上 |
| `echo "${ROS_HOSTNAME-}"` | 空 | `unset ROS_HOSTNAME` 后重启 `roscore` 与 `rosbag play` |
| `rosmsg md5 livox_ros_driver2/CustomMsg` | `e4d6829bdfe657cb6c21a746c86b21a6` | 重新 source `/home/nvidia/ligo_ws/devel/setup.bash` |
| `rosbag info` 的话题 | 含 `/livox/lidar`，1145 条 | 改播第 1 节的文件路径 |
| `rostopic info /livox/lidar` 的类型 | `livox_ros_driver2/CustomMsg` | 订阅该类型。`sensor_msgs/PointCloud2` 收不到这一路 |
| 发布者地址 | `http://127.0.0.1:` | 结束当前 `roscore`，在第 2 节的环境中重新启动后再播放 |
| `rostopic hz /livox/lidar` | 约 10 Hz | 确认播放仍在进行。播放结束后频率停止 |
| 订阅的话题名 | `/livox/lidar` | 第 3 节的文件不发布 `/lidar_points` |
| `/use_sim_time` | 未设置，或为 false | 第 4 节保持该状态。已设为 true 时，播放命令加上 `--clock`，或执行 `rosparam set /use_sim_time false` 后重开订阅进程 |

`rostopic list` 能看到话题、`rostopic hz` 没有频率时，发布者地址与当前终端的 `ROS_IP` 不一致。结束主节点和播放进程，使全部终端回到第 2 节，再从终端 A 重新开始。

在操作电脑上查看时，改用第 5 节的地址。NS16 保持 `ROS_IP=127.0.0.1` 时，操作电脑收不到点云。

## 9. 地址与文件

| 项目 | 取值 |
| --- | --- |
| NS16 直连地址 | `eth0`，`192.168.88.2/24` |
| 操作电脑直连地址 | `192.168.88.1/24`，路由器留空 |
| NS16 在机器狗网段的地址 | `eth0`，`192.168.1.110/24`，该地址无网关 |
| 机器狗运动主机 | `192.168.1.103` |
| 机器狗感知主机 | `192.168.1.105` |
| 机器狗记录主机 | `192.168.1.106`，用户 `ysc`，主机名 `ysc-perception` |
| 数据包在记录主机上的目录 | `~/jy_cog/system/record/bags/mapping/` |
| 数据包在 NS16 上的路径 | `/home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag` |
| 已解析点云在 NS16 上的路径 | `/home/nvidia/x30_ligo_test/mapping_13.bag` |
| ROS 环境 | `/opt/ros/noetic/setup.bash`，`/home/nvidia/ligo_ws/devel/setup.bash` |
| LIGO 可执行文件 | `/home/nvidia/ligo_ws/devel/lib/ligo/ligo_mapping` |

从记录主机补拷文件时，NS16 的 `eth0` 使用 `192.168.1.110/24`，并先 `ping 192.168.1.106`。拷到第 1 节的路径后，订阅仍按第 2 节把 `ROS_IP` 设为 `127.0.0.1`。
