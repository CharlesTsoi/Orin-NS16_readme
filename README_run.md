# 让机器狗按 NS16 导航行走

本文是在机器狗网络上启动 NS16 定位与导航、发送目标并确认速度命令的操作步骤。程序在板子 `/home/nvidia/gong_ws`，2026-11-23 已编译。按本文命令执行，不改参数、不改地图。

NS16 算出的速度发布在 `/cmd_vel_corrected`。`/home/nvidia/gong_ws` 内没有把该话题转发到运动主机 `192.168.1.103` 的 UDP `43893` 的程序。发出目标后必须先看这个话题有没有订阅者。没有订阅者时，路径可以规划出来，腿不会动。

## 1. 地址

| 设备 | 地址 | 作用 |
| --- | --- | --- |
| NS16 `eth0` | `192.168.1.110/24`，网关 `192.168.1.106` | 运行定位与导航 |
| 运动主机 | `192.168.1.103` | 腿部控制，UDP `43893` |
| 感知主机 | `192.168.1.105` | 感知 |
| 记录主机 | `192.168.1.106` | ROS master、雷达与 IMU 驱动 |

NS16 的登录用户名为 `nvidia`，密码为 `nvidia`。记录主机的用户名为 `ysc`，主机名为 `ysc-perception`。

网线一端接入板子 `eth0`（RJ45），另一端接入机器狗网口。从机器狗网络执行：

```sh
ping -c 3 192.168.1.110
ssh nvidia@192.168.1.110
```

应收到应答。主机名为 `ubuntu`。主机密钥不一致时，在发起连接的计算机上执行 `ssh-keygen -R 192.168.1.110` 后再连接。ECDSA 指纹为 `SHA256:WgOsBiCTREUrUwPE4pxxbW4pnG2vXKlXEDRcv1fILfk`。

不要使用 `/home/nvidia/gong_ws/scripts/x30_start.sh`。该脚本要求网口永久地址为 `3c:6d:66:0c:95:ff`。本板 `eth0` 的永久地址是 `4c:bb:47:ac:01:5f`，脚本的网口检查不会通过。

## 2. 在 NS16 上进入狗的 ROS

下面每个终端都先执行一次：

```sh
source /opt/ros/noetic/setup.bash
source /home/nvidia/gong_ws/devel/setup.bash
unset ROS_HOSTNAME
export ROS_MASTER_URI=http://192.168.1.106:11311
export ROS_IP=192.168.1.110
export DISABLE_ROS1_EOL_WARNINGS=1
```

然后确认雷达和 IMU 已在 master 上发布：

```sh
rostopic hz /livox/lidar
rostopic hz /imu/data
```

两路都应持续有输出。`/livox/lidar` 应为 Livox `CustomMsg`，`/imu/data` 应为 `sensor_msgs/Imu`。任一话题没有发布者时，先恢复记录主机上的传感器驱动，不要继续后面的步骤。

再看雷达消息的时间戳是否接近本机时间：

```sh
rostopic echo -n 1 /livox/lidar/header
date +%s
```

两处秒数应在同一年。记录主机时钟若仍在 1970 年附近，代价地图会报 `transform timeout`，本次不要发导航目标。

## 3. 停止记录主机上的原定位和原导航

在记录主机上执行：

```sh
bash /home/ysc/jy_cog/system/scripts/kill_localization.sh
bash /home/ysc/jy_cog/system/scripts/kill_nav.sh
```

停完后回到 NS16，再次执行第 2 节的两个 `rostopic hz`。传感器话题消失时，先把雷达和 IMU 驱动拉起来，并确认原来的定位节点没有再次发布 `map` 或 `camera_init`。两套定位同时发布 TF 时，导航会跳动。

## 4. 启动里程计、定位和导航

使用三个终端，每个终端都带第 2 节的环境变量。地图目录必须写成 `/home/nvidia/gong_ws`。不写 `workspace_dir` 时，launch 会到 `/home/nvidia/map` 找图，该目录不存在。

`scene:=default` 与 `scene:=futian_offline` 是同一张图，约 24 m × 17 m，原点在 `[-3.91, -2.34]`，覆盖大约 x ∈ [-3.9, 20.1] m、y ∈ [-2.3, 14.6] m。

终端 1，里程计：

```sh
roslaunch ligo mapping_astri.launch rviz:=false
```

等到 `/aft_mapped_to_init` 有输出，并且下面的命令能连续打印位姿：

```sh
rosrun tf tf_echo camera_init base_link
```

终端 2，全局定位：

```sh
roslaunch localization localize.launch \
  workspace_dir:=/home/nvidia/gong_ws scene:=default rviz:=false
```

该 launch 把初始位姿放在地图原点。狗如果不站在原点，先发布当前位姿，再看 TF。把 `X`、`Y`、`W` 换成狗在地图中的实际位置和朝向：

```sh
rostopic pub -1 /initialpose geometry_msgs/PoseWithCovarianceStamped \
  "{header: {frame_id: 'map'}, pose: {pose: {position: {x: X, y: Y, z: 0.0}, orientation: {w: W, x: 0.0, y: 0.0, z: Z}}}}"
```

朝向只用绕竖直轴的四元数。面朝地图 x 轴正方向时 `W=1`、`Z=0`。

确认全局定位已经接上里程计：

```sh
rosrun tf tf_echo map base_link
```

输出应连续、位置与狗的实际站位相符，不能来回跳。全局自动重定位是关闭的，位姿不对时只能再发 `/initialpose`。

终端 3，导航：

```sh
roslaunch navigation navigate.launch \
  workspace_dir:=/home/nvidia/gong_ws scene:=default rviz:=false
```

等到 `/move_base/global_costmap/costmap` 和 `/move_base/local_costmap/costmap` 都有输出，终端里不再持续刷 `transform timeout`。全局代价地图第一次更新最多约 1 秒。

有显示器时，把对应 launch 的 `rviz:=false` 改成 `rviz:=true`。没有 `DISPLAY` 时保持 `false`。

## 5. 先确认速度话题，再发短目标

发目标之前执行：

```sh
rostopic info /cmd_vel_corrected
```

`Publishers` 里应出现 `/move_base`。`Subscribers` 为空时，不要发目标：规划器只会在 NS16 上发布 `geometry_msgs/Twist`，运动主机收不到命令。

有订阅者后，发一个位于自由栅格内、距离当前位姿约 1 m 的目标。`X` 和 `Y` 必须落在第 4 节的地图范围内：

```sh
rostopic pub -1 /move_base_simple/goal geometry_msgs/PoseStamped \
  "{header: {frame_id: 'map'}, pose: {position: {x: X, y: Y, z: 0.0}, orientation: {w: 1.0}}}"
```

同时看速度：

```sh
rostopic echo /cmd_vel_corrected
```

跟踪路径时 `linear.x` 或 `angular.z` 应为非零，并低于第 7 节的限幅。人留在急停旁边。第一次只走这一小段，不要直接跑完整巡检。

有图形界面时，也可以在 RViz 里用 2D Nav Goal 点选。导航已经启动时，这个点击会直接成为行驶目标。

## 6. 沿已保存的巡检点行走

短距离目标确认腿能跟上之后，才使用已保存的 6 个点。这些点都在 `default` 这张图内，坐标系为 `map`。

| 名称 | x (m) | y (m) | yaw (rad) |
| --- | --- | --- | --- |
| pt1 | -0.967 | 10.383 | 1.609 |
| pt2 | 4.400 | 11.217 | 0 |
| pt3 | 8.100 | 5.750 | 0 |
| pt4 | 7.253 | 0.992 | -1.608 |
| pt5 | 0.490 | 0.691 | 3.012 |
| pt6 | -1.020 | 5.802 | 1.696 |

在第 4 节的导航已经运行时，另开终端执行第 2 节的环境变量，然后：

```sh
roslaunch route run_inspection.launch rviz:=false
rosservice call /route/start "data: false"
```

`data: false` 走一轮。`data: true` 为循环，第一次现场试验不要用。停止当前轮次：

```sh
rosservice call /route/stop "{}"
```

不要用 `record_points.launch` 或默认的 `inspection_route.launch` 叠加到已经运行的导航上。二者会再启动一个 `map_server`，与导航里的地图节点重名。

## 7. 本配置会发出的速度

导航实际加载的限幅如下。合速度不超过 0.8 m/s。

| 项目 | 取值 |
| --- | --- |
| 前进采样上限 | 1.0 m/s |
| 合速度上限 | 0.8 m/s |
| 横向速度上限 | 0.2 m/s |
| 角速度 | ±0.5 rad/s |
| 到达位置容忍 | 0.20 m |
| 到达朝向容忍 | 0.15 rad |
| 机体足迹 | 0.70 m × 0.44 m |
| 障碍膨胀 | 0.65 m |

局部代价地图只把机体坐标系下高度在 -0.8 m 到 0.2 m 之间、距离 2.5 m 以内的点标成障碍。

## 8. 停止

立即停住规划并发送 1 秒零速度：

```sh
bash /home/nvidia/gong_ws/src/navigation/scripts/emergency_stop.sh
```

该脚本同时向 `/cmd_vel_corrected` 和 `/cmd_vel` 发零速度。然后在三个 launch 终端里按 Ctrl-C。

零速度只持续 1 秒。若 `/cmd_vel_corrected` 仍有其他发布者，停完后继续看这个话题，直到速度保持为零。

## 9. 启动后仍不能走时的检查

| 现象 | 处理 |
| --- | --- |
| `/livox/lidar` 或 `/imu/data` 没有数据 | 留在第 2 节，不启动导航 |
| 雷达时间戳与 `date` 相差数年 | 不发目标。代价地图无法按墙钟取 TF |
| `tf_echo map base_link` 失败 | 里程计或 HDL 未起来。先看终端 1，再看终端 2 |
| 位姿离实际站位很远或来回跳 | 再发 `/initialpose`。不要发目标 |
| `transform timeout` 持续出现 | 不发目标。先核对时间戳和两段 TF |
| `/cmd_vel_corrected` 没有订阅者 | 规划可以成功，腿不会动。本板没有 UDP 转发 |
| `/cmd_vel_corrected` 有非零速度但腿不动 | 订阅者没有把 `Twist` 送到 `192.168.1.103:43893` |
| 目标一点就报在障碍里 | 换一个自由栅格内的点，或先核对初始位姿 |
