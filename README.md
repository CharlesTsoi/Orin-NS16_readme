# X30 数据包话题与 LIGO 输入的对齐

本文记录两次无头建图试验。两次均使用板上已安装的 LIGO，源码路径为 `/home/nvidia/ligo_ws/src/LIGO.-main`，不重新编译。下文对输入约定的说明均对应该源码。

第一次使用 2026-10-13 的已解析点云，见第 1–9 节。脚本位于 `config/x30.yaml`、`config/x30.launch`、`scripts/`，栅格与轨迹位于 `results/`。

第二次使用 2026-11-13 机器狗上的单台 Mid-360 原始消息，见第 10 节。参数位于 `config/x30_1113.yaml` 与 `config/x30_1113.launch`，启动脚本为 `scripts/run_1113.sh`。占据栅格为 `results/20261113/map.pgm`。供后端优化使用的接口说明见 [README_room.md](README_room.md)，只记录话题订阅、转换和实际传入的参数。网口对接步骤见 [README_network.md](README_network.md)。

## 1. 试验条件

| 项目 | 取值 |
| --- | --- |
| 平台 | 自制 Jetson，Ubuntu 20.04，ROS Noetic，无图形界面 |
| 程序 | 已安装的 `ligo_mapping`，可执行文件 `/home/nvidia/ligo_ws/devel/lib/ligo/ligo_mapping` |
| 数据包 | `mapping_20261013_222228_13.bag` |
| 时长 | 29.3 s（2026-10-13 22:28:51.18 至 22:29:20.49） |
| 播放 | `rosbag play --clock -r 1.0`，`/use_sim_time` 为 true |
| 工作目录 | 板上 `/home/nvidia/x30_ligo_test` |

播放返回码为 0。结果包覆盖 26.8 s，自 22:28:53.71 至 22:29:20.48。起始时刻晚于数据包约 2.5 s，对应 IMU 初始化完成之后才开始发布注册点云。

## 2. 数据包中的话题

| 话题 | 类型 | 条数 | 本次是否进入 LIGO |
| --- | --- | --- | --- |
| `/lidar_points` | `sensor_msgs/PointCloud2` | 294 | 是，经字段转发 |
| `/livox/lidar` | `livox_ros_driver2/CustomMsg` | 293 | 否 |
| `/livox/imu_192_168_2_202` | `sensor_msgs/Imu` | 5862 | 是，重映射为 `/livox/imu` |
| `/livox/imu_192_168_2_203` | `sensor_msgs/Imu` | 5860 | 否 |
| `/livox/imu_192_168_2_204` | `sensor_msgs/Imu` | 5861 | 否 |
| `/livox/imu_192_168_2_205` | `sensor_msgs/Imu` | 5861 | 否 |
| `/imu/data` | `sensor_msgs/Imu` | 5862 | 否 |
| `/leg_odom` | `nav_msgs/Odometry` | 5866 | 否 |
| `/tf` | `tf2_msgs/TFMessage` | 1694 | 否 |
| `/nlink_linktrack_nodeframe3` | `nlink_parser/LinktrackNodeframe3` | 1466 | 否 |

点云约 10 Hz（294 帧 / 29.3 s）。四路 Livox IMU 与 `/imu/data` 均约 200 Hz（约 5860 条 / 29.3 s）。

同一录制中有两路点云。`/livox/lidar` 为 Livox CustomMsg，抽查首帧约 80065 点，坐标系 `livox_frame`，`lidar_id` 为 0。`/lidar_points` 为已经解析为 `sensor_msgs/PointCloud2` 的同一测量，抽查首帧 79872 点，坐标系 `lidar_link`，`width = 79872`，`height = 1`。两路点数不完全相同。试验使用已解析的 `PointCloud2`，不再解析 Livox 自定义消息。

`/lidar_points` 的 `point_step` 为 26。按字段偏移读取如下。

| 字段 | 偏移（字节） | 类型 |
| --- | --- | --- |
| `x` | 0 | `float32` |
| `y` | 4 | `float32` |
| `z` | 8 | `float32` |
| `intensity` | 12 | `float32` |
| `tag` | 16 | `uint8` |
| `line` | 17 | `uint8` |
| `timestamp` | 18 | `float64` |

单帧 `timestamp` 的最大值与最小值之差约为 100。该跨度与 10 Hz 扫描的 100 ms 周期一致，因此将该 `float64` 视为扫描内的绝对毫秒时间，而不是秒或纳秒。转发时减去本帧最早时刻，得到以 0 为起点的毫秒偏移。

## 3. LIGO 接受的输入

LIGO 只订阅两个与本次对齐有关的话题，名称由参数给出，默认值如下（`parameters.cpp`）：

- `common/lid_topic`，默认 `/livox/lidar`
- `common/imu_topic`，默认 `/livox/imu`

订阅回调由 `lidar_type` 决定（`laserMapping.cpp`）：

```cpp
ros::Subscriber sub_pcl = p_pre->lidar_type == AVIA ?
    nh.subscribe(lid_topic, 200000, livox_pcl_cbk) :
    nh.subscribe(lid_topic, 200000, standard_pcl_cbk);
ros::Subscriber sub_imu = nh.subscribe(imu_topic, 200000, imu_cbk);
```

`LID_TYPE` 的取值为 `AVIA = 1`、`VELO16 = 2`、`OUST64 = 3`、`HESAIxt32 = 4`（`preprocess.h`）。只有 `AVIA` 走 `livox_pcl_cbk`，其消息类型是 `livox_ros_driver2/CustomMsg`。其余三种走 `standard_pcl_cbk`，消息类型是 `sensor_msgs/PointCloud2`，再按雷达型号选择不同的点字段。

因此存在两条互斥的接法：

1. 直接使用 `/livox/lidar`，设置 `lidar_type: 1`。LIGO 读取 CustomMsg 的 `line`、`tag` 和点内偏移时间。
2. 使用 `/lidar_points`。该话题是 `PointCloud2`，必须设置 `lidar_type` 为 2、3 或 4，并且点字段名、类型与对应解析函数一致。

本次采用第 2 条。`/lidar_points` 已经是解析后的点云，不再经过 Livox 自定义消息。

### 3.1 三种 PointCloud2 解析函数要求的字段

`pcl::fromROSMsg` 按字段名拷贝，不要求 ROS 点的内存布局与 PCL 结构体的对齐填充一致。字段名或类型不匹配时，对应成员保持默认值，随后的时间计算会失去扫描内偏移。

| `lidar_type` | 解析函数 | 时间字段 | 线号字段 |
| --- | --- | --- | --- |
| 2 `VELO16` | `velodyne_handler` | `time`，`float32` | `ring`，`uint16` |
| 3 `OUST64` | `oust64_handler` | `t`，`uint32` | `ring`，`uint8` |
| 4 `HESAIxt32` | `hesai_handler` | `timestamp`，`float64` | `ring`，`uint16` |

`/lidar_points` 的时间字段名是 `timestamp`，类型是 `float64`，与 `HESAIxt32` 一致，与 Velodyne 的 `time`/`float32`、Ouster 的 `t`/`uint32` 不一致。线号字段名是 `line` 而不是 `ring`，类型是 `uint8` 而不是 `uint16`。因此不能把 `/lidar_points` 原样交给上述任一解析函数，需要转发并改名。

选择类型 4 而不是类型 2，是因为源时间字段已经是 `float64`。类型 4 只需修改字段名，并把 `line` 提升为 `uint16`，时间保持双精度。类型 2 还要求把时间改名为 `time` 并收窄为 `float32`。

### 3.2 时间单位与曲率

`PointCloud2` 进入预处理后，先由 `timestamp_unit` 得到换算到毫秒的系数 `time_unit_scale`（`preprocess.cpp`）：

| `timestamp_unit` | 枚举 | 系数 |
| --- | --- | --- |
| 0 | `SEC` | 1000 |
| 1 | `MS` | 1 |
| 2 | `US` | 0.001 |
| 3 | `NS` | 1e-6 |

LIGO 点的 `curvature` 以毫秒为单位。Hesai 解析函数的计算为：

```text
given_offset_time = (points[last].timestamp > 0)
time_head         = points[0].timestamp
curvature         = (points[i].timestamp - time_head) * time_unit_scale
```

该式对点的存放顺序有两个要求。

- `points[0]` 必须是本帧最早的点。否则 `time_head` 不是最小时间，更早的点会得到负的 `curvature`。
- `points[last].timestamp > 0` 必须成立。否则 `given_offset_time` 为 false，函数放弃点内时间，改用 `ring` 作为线号下标，按水平角估算偏移。

后一条同时限制了 `ring` 的取值。无时间戳的分支执行 `is_first[ring]`，而 `is_first` 的长度为 `N_SCANS`。`N_SCANS` 来自参数 `preprocess/scan_line`。`ring >= scan_line` 时该分支越界。本次配置 `scan_line: 4`，而实测 `line` 的最大值为 15。因此本次运行依赖点内时间有效，不能进入按 `ring` 估算时间的分支。

### 3.3 IMU

IMU 回调只认 `common/imu_topic` 上的 `sensor_msgs/Imu`。与本次相关的参数沿用板上 Mid-360 配置：

- `mapping/imu_time_inte: 0.005`，对应 200 Hz，与包内 IMU 条数一致。
- `mapping/acc_norm: 1.0`。该参数把加速度换到算法使用的尺度；板上 Mid-360 配置以 Livox IMU 的 g 为单位，故取 1.0。若改接 `/imu/data`，其首帧加速度约为 `(3.481, 0.401, 10.225)`，模长约 10.8，与 m/s² 一致，此时应将 `acc_norm` 改为 9.81。本次不使用 `/imu/data`。
- `mapping/gravity: [0.0, 0.0, -9.81]`，`mapping/gravity_init: [0.0, 9.810, 0.0]`，`mapping/init_with_imu: true`。
- `mapping/extrinsic_T: [-0.011, -0.02329, 0.04412]`，`extrinsic_R` 为单位阵。这是单台 Mid-360 与其自带 IMU 的外参，不是四台雷达相对机体的标定。

## 4. 点云转发

转发节点为 `scripts/cloud_relay.py`。它订阅 `/lidar_points`，发布 `/lidar_points_ligo`。LIGO 的 `common/lid_topic` 设为后者。

对每个点执行下列变换。

1. 按第 2 节的偏移读取 `x, y, z, intensity`、`line` 和 `timestamp`，丢弃非有限值。`tag` 不进入 LIGO 的 Hesai 点类型，予以丢弃。
2. 用本帧 `timestamp` 的跨度判断单位。跨度大于 `1e6` 时视为纳秒并除以 `1e6`；跨度不大于 10 时视为秒并乘以 1000；否则视为已经是毫秒。本次首帧落入第三支。
3. 按时间升序排序，再减去排序后的第一个时间，使 `timestamp[0] = 0`，`timestamp[last]` 等于本帧跨度。
4. `line` 提升为 `uint16`，字段名改为 `ring`。

输出仍是 `sensor_msgs/PointCloud2`，`point_step = 26`，字段如下。

| 字段 | 偏移（字节） | 类型 |
| --- | --- | --- |
| `x, y, z, intensity` | 0, 4, 8, 12 | `float32` |
| `timestamp` | 16 | `float64`，本帧内毫秒偏移 |
| `ring` | 24 | `uint16`，来自 `line` |

首帧日志为：

```text
relay points=79872 time_span_ms=100.24 line_max=15
```

`time_span_ms = 100.24` 使 `points[last].timestamp > 0`，Hesai 解析进入点内时间分支。`timestamp_unit: 1` 使系数为 1，`curvature` 即该毫秒偏移。`line_max = 15` 只被写入 `ring`；由于没有进入无时间戳分支，它不被用作 `scan_line = 4` 的下标。

## 5. IMU 话题对齐

四路 Livox IMU 的话题名带有设备地址，LIGO 只订阅一个 IMU 话题。播放时做话题重映射，不改消息内容：

```text
rosbag play --clock -r 1.0 mapping_13.bag /livox/imu_192_168_2_202:=/livox/imu
```

选择 `192.168.2.202` 是因为它与 `/livox/lidar` 首帧的 `lidar_id = 0` 对应，是四台设备中的第一路。`/lidar_points` 单帧约 8 万点，数量约为单台 Mid-360（约 20 万点/秒、10 Hz）的四倍，与四台设备合并后的一帧相符。外参和 IMU 却只有第一路的 Mid-360 参数。其余三路雷达的点被放进同一扫描，但没有各自的外参和 IMU。这是本次几何结果的主要限制，不是话题未对齐。

`config/x30.yaml` 中 `common/imu_topic` 为 `/livox/imu`，与重映射后的名称一致。

## 6. 运行时未接入的数据

下列话题留在数据包中，播放时会被发布，但没有节点订阅，也不参与状态估计。

- `/livox/lidar`：与 `/lidar_points` 同源。若要改用它，应设置 `lidar_type: 1` 并直接订阅该话题，同时删除本次转发。两条路径不应同时作为 `lid_topic`。
- `/livox/imu_192_168_2_203`、`_204`、`_205`：另外三台 Livox 的 IMU。当前程序没有多 IMU 输入。
- `/imu/data`：机体 IMU。接入时需要把 `acc_norm` 改为 9.81，并重新确认 `gravity_init` 与外参，不能与 `acc_norm: 1.0` 混用。
- `/leg_odom`、`/tf`、`/nlink_linktrack_nodeframe3`：LIGO 本次配置不读取里程计、坐标变换或 UWB。

GNSS 与 NMEA 在启动参数中关闭（`gnss/gnss_enable: false`，`nmea/nmea_enable: false`）。不启动 RViz。不保存 PCD（`pcd_save/pcd_save_en: false`），以避免该配置在保存点云时的内存问题。盲区 `preprocess/blind` 为 0.5 m，量测距离上限 `det_range` 为 1000 m。

## 7. 启动顺序

`scripts/run_x30.sh` 在板上按下列顺序执行。ROS 环境为 `/opt/ros/noetic/setup.bash` 与 `/home/nvidia/ligo_ws/devel/setup.bash`。

1. 启动 `roscore`，设置 `/use_sim_time` 为 true。
2. 以 500 ms 间隔启动 `tegrastats`，输出 `tegrastats.log`。
3. 录制 `/cloud_registered`、`/aft_mapped_to_init`、`/path`，压缩格式为 lz4。
4. 以 `config/x30.launch` 启动 `ligo_mapping`，并启动 `cloud_relay.py`。
5. 等待订阅建立后，按第 5 节的重映射以 1 倍速播放数据包。
6. 播放结束后等待 15 s，再向建图、录制和转发进程发送 `SIGINT`。

`config/x30.launch` 加载 `x30.yaml`，并显式关闭 GNSS，设置 `point_filter_num = 1`、`filter_size_surf = 0.3`、`filter_size_map = 0.3`。

## 8. 输出

LIGO 在该配置下发布并被录制的话题如下。

| 话题 | 类型 | 条数 | 含义 |
| --- | --- | --- | --- |
| `/cloud_registered` | `sensor_msgs/PointCloud2` | 262 | 注册到地图坐标系的扫描 |
| `/aft_mapped_to_init` | `nav_msgs/Odometry` | 262 | 地图坐标系下的位姿 |
| `/path` | `nav_msgs/Path` | 262 | 与位姿对应的路径 |

262 帧相对 294 帧输入少 32 帧，时间上对应数据包开头约 2.5 s。此区间用于 IMU 初始化，不发布地图。

### 8.1 轨迹

`results/trajectory.csv` 的列为 `t, x, y, z`，取自 `/aft_mapped_to_init` 的位置，共 262 行，时间跨度 26.1 s。

| 轴 | 最小值 (m) | 最大值 (m) |
| --- | --- | --- |
| x | -1.19 | 0.78 |
| y | 0.00 | 0.06 |
| z | -0.02 | 1.18 |

x 在约 2 m 的区间内多次往复。y 的变化不超过 0.06 m。z 先上升至约 1.18 m，结束时约为 0.9 m。三维折线长度约 10.9 m；该长度主要来自 x 的往复，不是沿单一方向的 10.9 m 位移。

![轨迹](results/trajectory.png)

### 8.2 占据栅格

`scripts/make_map.py` 在建图进程退出后读取结果包。栅格只保留地图坐标 z 位于 [-0.4, 1.6] m 的点，分辨率 0.05 m，向点集外包 1 m。图像行方向与 y 轴相反。占用为 0，空闲为 255，格式为 P5。原始文件为 `results/map.pgm`，范围见 `results/map_meta.txt`。

| 量 | 值 |
| --- | --- |
| x | -33.54 m 至 78.70 m |
| y | -8.50 m 至 1.56 m |
| 宽 × 高 | 2246 × 203 |
| 落入该高度层的抽样点 | 28972 |

点云主体位于 y 约 -6 m 的一条结构上。轨迹位于原点附近，在整幅栅格中只占很小一段。该分布与“机器人位置变化约 2 m、侧向观测到一条延伸结构”相符。它不表示四台雷达已经用各自外参拼接到同一车体坐标系。

![占据栅格](results/map.png)

下图左侧为同一高度层内的抽样点与轨迹，右侧为 y 位于 [-1, 1] m 的点以及轨迹在 x–z 平面上的投影。

![点云与轨迹](results/cloud_traj.png)

### 8.3 处理器与内存

采样来自 `tegrastats --interval 500`，原始记录为 `results/tegrastats.log`。图中三条竖线依次为播放开始、播放结束和发送停止信号，时刻见 `results/events.log`。CPU 百分比是处于运行状态的核的平均值。`tegrastats` 显示 8 个逻辑核中有 4 个为 `off`，计算时不把这 4 个核计入平均。

![处理器与内存](results/cpu_gpu.png)

播放进行的约 31 s 内：

| 量 | 值 |
| --- | --- |
| 在线核的平均占用 | 41% |
| 在线核的峰值占用 | 61% |
| GPU `GR3D_FREQ` | 全程 0% |
| 内存 | 由约 0.58 GB 升至最高 0.90 GB（物理内存 15503 MB） |

播放结束时刻 CPU 占用回到接近空闲，而不是在之后继续维持高占用。据此判断，1 倍速播放期间建图没有形成需要在播放结束后长时间排空的队列。GPU 频率保持为 0，说明该 LIGO 建图进程没有使用 GPU。停止进程后内存回落到约 0.77 GB。

## 9. 结论与边界

话题对齐本身成立：`/lidar_points` 经字段转发后满足 `lidar_type = 4` 的 `timestamp`/`ring` 约定，首帧扫描内偏移为 100.24 ms；`/livox/imu_192_168_2_202` 经重映射进入 `common/imu_topic`。在此输入下，`ligo_mapping` 以 1 倍速完成 29.3 s 数据包，并连续发布 262 帧注册点云、里程计和路径。

几何结果受输入组合限制。点云是多台设备合并后的扫描，IMU 与外参只对应其中地址为 `192.168.2.202` 的一台，且外参取自 Mid-360 的既有配置。因此栅格和轨迹用于确认程序能够在该无头平台上运行，不作为多雷达标定后的地图。若后续改为单台雷达的 CustomMsg，应使用 `lidar_type: 1` 并去掉转发；若改为机体 IMU，应同时把 `acc_norm` 改为 9.81 并核对 `gravity_init`。在点内时间失效时，不能保留现在的 `scan_line: 4`，因为实测线号达到 15，无时间戳分支会按下标访问长度为 4 的数组。

第 10 节记录了随后一次按上述两条建议执行的试验。该数据包没有已解析点云，只有单台雷达的 CustomMsg，IMU 为以 m/s² 为单位的 `/imu/data`。运行使用 `lidar_type: 1` 与 `acc_norm: 9.81`，不再经过点云转发。

## 10. 2026-11-13 单台 Mid-360

本节记录第二次试验。输入改为机器狗当日录制的原始 Livox 消息，不再使用第 4 节的字段转发，也不再使用第 5 节的 IMU 话题重映射。占据栅格为 `results/20261113/map.pgm`。

### 10.1 试验条件

| 项目 | 取值 |
| --- | --- |
| 平台 | 与第 1 节相同的自制 Jetson，Ubuntu 20.04，ROS Noetic，无图形界面 |
| 程序 | 已安装的 `ligo_mapping`，未重新编译 |
| 数据包 | `LIO_2026-11-13-16-32-56.bag` |
| 源路径 | 机器狗 `~/jy_cog/system/record/bags/mapping/` |
| 时长 | 114.5 s（2026-11-13 16:32:56.41 至 16:34:50.90） |
| 大小 | 1,767,271,815 字节，无压缩，1146 个 chunk |
| 播放 | `rosbag play --clock -r 1.0`，`/use_sim_time` 为 true |
| 工作目录 | 板上 `/home/nvidia/x30_ligo_test` |
| 参数 | `config/x30_1113.yaml`，由 `config/x30_1113.launch` 加载 |
| 启动 | `scripts/run_1113.sh` |

当日该录制目录下只有这一个数据包。文件先经 Mac 拷到板上，板上原有的 `mapping_13.bag` 未替换。板子 `eth0` 保持 `192.168.88.2/24`。

播放返回码为 0，墙钟 116.2 s。结果包覆盖 111 s，自 16:32:58.92 至 16:34:50.90，与数据包结尾对齐。起始时刻晚于数据包约 2.5 s，仍对应 IMU 初始化。

### 10.2 数据包中的话题

| 话题 | 类型 | 条数 | 本次是否进入 LIGO |
| --- | --- | --- | --- |
| `/livox/lidar` | `livox_ros_driver2/CustomMsg` | 1145 | 是，直接订阅 |
| `/imu/data` | `sensor_msgs/Imu` | 22899 | 是，直接订阅 |
| `/leg_odom` | `nav_msgs/Odometry` | 22899 | 否 |

该包没有 `/lidar_points`，也没有 `/livox/imu_*`。因此第 2 节的两条点云路径中，只有 CustomMsg 这一条可用。点云约 10 Hz（1145 帧 / 114.5 s）。`/imu/data` 为 22899 / 114.5 s，约 200 Hz。

板上的 LIGO 编译时包含 `livox_ros_driver2/CustomMsg`（`preprocess.h`、`parameters.h`）。包内该消息的 MD5 为 `e4d6829bdfe657cb6c21a746c86b21a6`，与板上 `livox_ros_driver2/CustomMsg` 的 MD5 相同，订阅能够建立。

抽查首帧：坐标系 `livox_frame`，`lidar_id` 为 0，`point_num` 为 79968，点数组长度为 79969。`line` 只取 0、1、2、3。`offset_time` 最小为 0，最大为 100334567，跨度 100.3 ms，单位为纳秒。这是单台 Mid-360 的一帧，不是第 2 节那种约 8 万点、线号达到 15 的多雷达合并扫描。

### 10.3 点云接法

`lidar_type` 取 `AVIA = 1`。`laserMapping.cpp` 在该取值下把 `common/lid_topic` 交给 `livox_pcl_cbk`，消息类型为 `livox_ros_driver2/CustomMsg`。`config/x30_1113.yaml` 中该话题为 `/livox/lidar`，与包内名称一致，播放时不做重映射。

`avia_handler`（`preprocess.cpp`）不读取 `timestamp_unit`。该参数只作用于 PointCloud2 的时间字段，见第 3.2 节。CustomMsg 的扫描内时间由下列赋值直接得到，`curvature` 的单位仍是毫秒：

```text
plsize    = msg->point_num
curvature = points[i].offset_time / 1000000
```

循环下标从 1 开始，到 `point_num` 为止。首帧因此不使用点数组中多出的那一个点，也不使用下标 0 的点。`offset_time` 为 100334567 纳秒时，`curvature` 为 100.33 ms，与 10 Hz 扫描的周期一致。

点被保留还需同时满足：

```text
points[i].line < N_SCANS
(points[i].tag & 0x30) == 0x10  或  (points[i].tag & 0x30) == 0x00
距离位于 [blind, det_range]
与前一点的坐标差超过 1e-7
```

`N_SCANS` 来自 `preprocess/scan_line`。实测 `line` 最大为 3，故 `scan_line: 4` 使 0–3 全部落入范围，同时不会把线号当作越界下标。这与第 3.2 节的约束不同：第 3.2 节的 `line` 最大为 15，`scan_line: 4` 只在点内时间有效时才安全；本次线号本身小于 4。

`tag` 的过滤使用比特 4–5。首帧 79968 个由 `point_num` 计数的点中，约 79526 个满足 `0x00` 或 `0x10`。计数值最高的是 `tag = 0`（70167）和 `tag = 16`（4444）。`blind` 为 0.5 m，`det_range` 为 1000 m，与第一次试验相同。

本次没有运行 `scripts/cloud_relay.py`。`/lidar_points` 不存在，Hesai 字段转发没有输入。

### 10.4 IMU

`common/imu_topic` 设为 `/imu/data`。播放命令不包含话题重映射。

静止段前几帧的线加速度约为 `(0.037, -0.029, 9.786)` m/s²，模长 9.786；角速度约为 0.001 rad/s。坐标系为 `imu_link`。这与 Livox 驱动以 g 为单位发布的 IMU 不同。

状态更新中的加速度换算（`Estimator.cpp`）为：

```text
acc_avr * G_m_s2 / acc_norm
```

`G_m_s2` 为 9.81。消息已经是 m/s² 时，`acc_norm` 必须为 9.81，换算结果才回到消息本身的数值。若沿用第一次试验的 `acc_norm: 1.0`，同一测量会被再乘 9.81。`config/x30_1113.yaml` 因此取 `acc_norm: 9.81`。`imu_time_inte` 仍为 0.005 s，对应 200 Hz。

`init_with_imu` 为 true。初始化时重力取实测加速度的反方向，模长为 `gravity` 向量的范数（`laserMapping.cpp`）。配置为 `gravity: [0.0, 0.0, -9.81]`。静止时加速度指向 IMU 的 +z，估计出的重力指向 −z。`gravity_init: [0.0, 0.0, -9.81]` 只在 `init_with_imu` 为 false 时作为先验；本次该分支不执行，仍写成与估计方向一致的值，避免关闭该开关后沿用第一次试验里指向 +y 的先验。

外参仍取板上 `config/mid360.yaml` 的 Mid-360 数值：`extrinsic_T: [-0.011, -0.02329, 0.04412]`，`extrinsic_R` 为单位阵。这不是本段录制的现场标定。GNSS 与 NMEA 关闭，不启动 RViz，不保存 PCD。

`/leg_odom` 会随数据包发布，没有节点订阅。

### 10.5 启动顺序

`scripts/run_1113.sh` 与 `scripts/run_x30.sh` 的差别只有输入：

1. 启动 `roscore`，设置 `/use_sim_time` 为 true。
2. 以 500 ms 间隔启动 `tegrastats`。
3. 录制 `/cloud_registered`、`/aft_mapped_to_init`、`/path`，压缩格式为 lz4。
4. 以 `config/x30_1113.launch` 启动 `ligo_mapping`。不启动点云转发。
5. 播放 `LIO_2026-11-13-16-32-56.bag`，速率 1.0，带 `--clock`，不做话题重映射。
6. 播放结束后等待 15 s，再向建图和录制进程发送 `SIGINT`。

`x30_1113.launch` 同样设置 `point_filter_num = 1`、`filter_size_surf = 0.3`、`filter_size_map = 0.3`，并关闭 GNSS。

标准输出见 `results/20261113/ligo.log`。其中 `lidar_type: 1`，`gnss enable:0`，`nmea enable:0`。日志没有报错。

### 10.6 输出

| 话题 | 类型 | 条数 | 含义 |
| --- | --- | --- | --- |
| `/cloud_registered` | `sensor_msgs/PointCloud2` | 1121 | 注册到地图坐标系的扫描 |
| `/aft_mapped_to_init` | `nav_msgs/Odometry` | 1121 | 地图坐标系下的位姿 |
| `/path` | `nav_msgs/Path` | 1121 | 与位姿对应的路径 |

1121 帧相对 1145 帧输入少 24 帧，时间上对应数据包开头约 2.5 s 的 IMU 初始化。结果包在板上为 lz4，压缩后 124 MB。

#### 10.6.1 轨迹

`results/20261113/trajectory.csv` 的列为 `t, x, y, z`，共 1121 行，时间跨度 112.0 s。

| 轴 | 最小值 (m) | 最大值 (m) |
| --- | --- | --- |
| x | -0.08 | 10.62 |
| y | -9.68 | 0.23 |
| z | -0.08 | 0.13 |

三维折线长度 58.6 m，水平投影 56.7 m。z 的全程变化不超过 0.21 m。起点与终点都在原点附近，平面轨迹闭合。

![轨迹](results/20261113/trajectory.png)

#### 10.6.2 占据栅格

栅格由与第 8.2 节相同的程序从结果包生成：只保留地图坐标 z 位于 [-0.4, 1.6] m 的点，分辨率 0.05 m，向点集外包 1 m。占用为 0，空闲为 255，格式为 P5。文件为 `results/20261113/map.pgm`，范围见 `results/20261113/map_meta.txt`。

| 量 | 值 |
| --- | --- |
| x | -15.89 m 至 15.65 m |
| y | -22.87 m 至 5.31 m |
| 宽 × 高 | 632 × 565 |
| 落入该高度层的抽样点 | 241455 |
| 占用格 | 43712 |

轨迹位于房间轮廓内部，并回到起点附近。注册点云在高度上主要位于 0 m 至约 2.4 m；z 的 5% 分位为 -0.45 m，中位数为 1.42 m，95% 分位为 2.43 m。抽样中存在更远的离群点，未进入上述高度层。

![占据栅格](results/20261113/map.png)

下图左侧为同一高度层内的抽样点与轨迹，右侧为 y 位于 [-1, 1] m 的点以及轨迹在 x–z 平面上的投影。右侧轨迹接近水平。

![点云与轨迹](results/20261113/cloud_traj.png)

#### 10.6.3 处理器与内存

采样来自 `tegrastats --interval 500`，原始记录为 `results/20261113/tegrastats.log`。图中三条竖线依次为播放开始、播放结束和发送停止信号，时刻见 `results/20261113/events.log`。8 个逻辑核中有 4 个为 `off`，CPU 百分比是其余 4 个核的平均值。物理内存为 15503 MB。

![处理器与内存](results/20261113/cpu_gpu.png)

播放进行的 116 s 内：

| 量 | 值 |
| --- | --- |
| 在线核的平均占用 | 25.5% |
| 在线核的峰值占用 | 35.5% |
| GPU `GR3D_FREQ` | 全程 0% |
| 内存 | 由约 0.63 GB 升至约 0.89 GB，播放期间平均 0.86 GB |

播放结束时刻 CPU 占用回到接近空闲。1 倍速播放期间建图没有形成需要在播放结束后长时间排空的队列。GPU 频率保持为 0。停止进程后内存回落到约 0.78 GB。

### 10.7 与第一次试验的差别

| 项目 | 2026-10-13 | 2026-11-13 |
| --- | --- | --- |
| 点云话题 | `/lidar_points`，经转发 | `/livox/lidar`，直接订阅 |
| `lidar_type` | 4，Hesai 字段 | 1，Livox CustomMsg |
| 线号 | 最大 15，多台合并 | 0–3，单台 |
| IMU | `/livox/imu_192_168_2_202`，`acc_norm` 1.0 | `/imu/data`，`acc_norm` 9.81 |
| 时长 | 29.3 s，输出 262 帧 | 114.5 s，输出 1121 帧 |
| 轨迹 | x 约 2 m 往复，z 上升约 1.2 m | 平面闭合，路径 58.6 m，z 变化不超过 0.21 m |
| 在线核平均占用 | 41% | 25.5% |

外参两次都取自同一份 Mid-360 配置，都不是现场标定。第一次的几何限制是多台雷达共用一路 IMU 和外参。第二次的输入与 `lidar_type: 1` 的字段约定一致，轨迹在平面内闭合且高度稳定；外参仍未在本段数据上重新估计。
