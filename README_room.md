# 房间建图：LIGO 输入接口

本文只说明 2026-11-13 房间建图这一次喂给 LIGO 的输入：订阅哪些话题、消息有没有在外部转换、哪些参数修改后传给了节点、哪些没有传入。不评价地图几何。

运行对象是已安装的 `ligo_mapping`，节点名为 `laserMapping`。参数由私有句柄读取（`ros::NodeHandle nh("~")`），因此下文的键都在 `/laserMapping` 下。配置文件是 `config/x30_1113.yaml` 与 `config/x30_1113.launch`。播放时不做话题重映射。

程序内默认值指 `parameters.cpp` 里 `nh.param` 的第三个参数。键不存在时使用该默认值。

## 1. 订阅

只订阅两个话题。`lidar_type = 1`（`AVIA`）时点云走 Livox 回调，不走 `PointCloud2` 回调。

| 话题 | 消息 | 队列 | 回调 |
| --- | --- | --- | --- |
| `/livox/lidar` | `livox_ros_driver2/CustomMsg` | 200000 | `livox_pcl_cbk` |
| `/imu/data` | `sensor_msgs/Imu` | 200000 | `imu_cbk` |

话题名来自 `common/lid_topic` 与 `common/imu_topic`，均带前导 `/`，按全局名订阅。

数据包里还有 `/leg_odom`（`nav_msgs/Odometry`，22899 条）。没有订阅，没有重映射，不进入估计。

`/livox/lidar` 1145 条，约 10 Hz。首帧：坐标系 `livox_frame`，`lidar_id = 0`，`point_num = 79968`，点数组长度 79969。`line` 只取 0、1、2、3。`offset_time` 为纳秒，首帧从 0 到 100334567，跨度 100.3 ms。消息 MD5 为 `e4d6829bdfe657cb6c21a746c86b21a6`，与节点链接的 `livox_ros_driver2/CustomMsg` 一致。

`/imu/data` 22899 条，约 200 Hz，坐标系 `imu_link`。静止时线加速度约为 `(0.037, -0.029, 9.786)` m/s²，角速度约为 0.001 rad/s。消息本身没有改单位。

另外设置了全局参数 `/use_sim_time = true`，并用 `rosbag play --clock -r 1.0` 发布 `/clock`。这不是 `laserMapping` 的私有参数。

GNSS 与 NMEA 的使能为 false，对应的星历、观测和 NMEA 话题都没有订阅。

## 2. 转换

没有外部转换节点。`rosbag play` 按包内原话题、原消息发出。`scripts/cloud_relay.py` 这次没有启动。

点云进入 `avia_handler` 之后，由 LIGO 自己做下面的变换。这不是本次额外插入的步骤。

- 循环范围是 `i = 1 .. point_num - 1`。下标 0 不使用。首帧点数组比 `point_num` 多 1，多出的点也不使用。
- 保留条件：`line < scan_line`，且 `(tag & 0x30)` 为 `0x00` 或 `0x10`，且距离在 `[blind, det_range]` 内，且与前一点坐标差大于 `1e-7`。
- `reflectivity` 写入 `intensity`。
- `curvature = offset_time / 1000000`。`offset_time` 以纳秒计，`curvature` 以毫秒计。首帧最大约 100.33 ms。
- `point_filter_num = 1`，通过上述条件的点全部保留。

`timestamp_unit` 不参与上述除法。该参数只给 `PointCloud2` 的时间字段使用。本次点云不是 `PointCloud2`。

IMU 消息不做字段转换。加速度只在滤波器内按 `acc * 9.81 / acc_norm` 缩放。`acc_norm = 9.81` 时，m/s² 数值进入滤波器时不再乘 9.81。

## 3. 修改后传入的参数

下表是相对 `parameters.cpp` 默认值作了修改、并且节点会读到的键。

| 键 | 默认值 | 传入值 | 说明 |
| --- | --- | --- | --- |
| `common/imu_topic` | `/livox/imu` | `/imu/data` | 改为包内机体 IMU |
| `mapping/acc_norm` | `1.0` | `9.81` | 消息已是 m/s²。默认值 `1.0` 会把加速度再乘 9.81 |
| `mapping/gravity` | 空 | `[0, 0, -9.81]` | 提供重力模长。`init_with_imu` 为 true 时，方向由实测加速度取反，不使用 `gravity_init` |
| `mapping/gravity_init` | 空 | `[0, 0, -9.81]` | 已传入。仅当 `init_with_imu` 为 false 时作为重力先验。本次为 true，初始化不用它 |
| `mapping/extrinsic_T` | 空 | `[-0.011, -0.02329, 0.04412]` | 沿用 Mid-360 配置，不是本段房间数据的标定 |
| `mapping/extrinsic_R` | 空 | 单位阵 | 同上 |
| `preprocess/scan_line` | `16` | `4` | 与 `line` 的取值 0–3 一致 |
| `preprocess/blind` | `1.0` | `0.5` | 预处理距离下限，单位 m |
| `preprocess/det_range` | `1.0` | `1000` | 预处理距离上限，单位 m。不是 `mapping/det_range` |
| `mapping/plane_thr` | `0.05` | `0.1` | |
| `mapping/lidar_meas_cov` | `0.1` | `0.01` | |
| `mapping/acc_cov_output` | `0.1` | `500` | |
| `mapping/gyr_cov_output` | `0.1` | `1000` | |
| `mapping/ivox_grid_resolution` | `0.2` | `1.0` | |
| `ivox_nearby_type` | `18` | `6` | `6` 表示 `NEARBY6` |
| `init_map_size` | `100` | `10` | 由 launch 传入 |
| `point_filter_num` | `2` | `1` | 由 launch 传入 |
| `filter_size_surf` | `0.5` | `0.3` | 由 launch 传入 |
| `filter_size_map` | `0.5` | `0.3` | 由 launch 传入 |
| `publish/scan_bodyframe_pub_en` | `true` | `false` | |
| `gnss/gnss_extrinsic_T` | 空 | `[0.011, 0.03155, 0.02365]` | 节点会读。GNSS 关闭，不参与估计 |
| `gnss/gnss_extrinsic_R` | 空 | 单位阵 | 同上 |
| `nmea/ppp_anc` | 空 | `[0, 0, 0]` | 节点会读。NMEA 关闭，不参与估计 |

其中只有前三项是针对本包相对板上 Mid-360 示例配置改动的：`imu_topic`、`acc_norm`、`gravity_init`。其余修改值来自该示例配置，由本次 yaml 或 launch 原样传入。

`mapping/gravity_init` 虽然改成了 `[0, 0, -9.81]` 并写入参数，初始化路径不使用它。

## 4. 传入且与默认值相同的参数

这些键出现在 yaml 或 launch 中，节点会读到，取值等于程序默认值。

| 键 | 传入值 |
| --- | --- |
| `common/lid_topic` | `/livox/lidar` |
| `common/con_frame` | `false` |
| `common/con_frame_num` | `1` |
| `preprocess/lidar_type` | `1` |
| `preprocess/timestamp_unit` | `1` |
| `mapping/imu_en` | `true` |
| `mapping/init_with_imu` | `true` |
| `mapping/imu_time_inte` | `0.005` |
| `mapping/lidar_time_inte` | `0.1` |
| `mapping/satu_acc` | `3.0` |
| `mapping/satu_gyro` | `35` |
| `mapping/b_acc_cov` | `0.0001` |
| `mapping/b_gyr_cov` | `0.0001` |
| `mapping/imu_meas_acc_cov` | `0.1` |
| `mapping/imu_meas_omg_cov` | `0.1` |
| `mapping/match_s` | `81` |
| `prop_at_freq_of_imu` | `true` |
| `check_satu` | `true` |
| `space_down_sample` | `true` |
| `odometry/publish_odometry_without_downsample` | `false` |
| `publish/path_en` | `true` |
| `publish/scan_publish_en` | `true` |
| `runtime_pos_log_enable` | `false` |
| `pcd_save/pcd_save_en` | `false` |
| `pcd_save/interval` | `-1` |
| `gnss/gnss_enable` | `false`（yaml 与 launch 都写了） |
| `gnss/gt_file_type` | `1` |
| `gnss/gnss_ekf_noise` | `0.01` |
| `nmea/nmea_enable` | `false` |

`preprocess/timestamp_unit` 被读入并保存，`avia_handler` 不使用它。

## 5. 没有传入的参数

下列键不在 yaml 中，也不在 launch 中。节点使用默认值。

| 键 | 因此采用的默认值 | 与本次输入的关系 |
| --- | --- | --- |
| `mapping/det_range` | `300` | 与已传入的 `preprocess/det_range = 1000` 不是同一个量 |
| `mapping/fov_degree` | `180` | 未改 |
| `mapping/acc_cov_input` | `0.1` | 未改 |
| `mapping/vel_cov` | `20` | 未改 |
| `mapping/gyr_cov_input` | `0.1` | 未改 |
| `mapping/dyn_filter` | `true` | 未改 |
| `mapping/dyn_filter_resolution` | `0.1` | 未改 |
| `preprocess/scan_rate` | `10` | 见下一节。yaml 里的 `common/scan_rate` 不会填到这个键 |

没有传入第二路雷达话题，也没有传入第二路外参。本次程序没有这些键。

## 6. 写进配置但节点没有读的内容

`common/scan_rate: 10` 写在 yaml 的 `common` 段。节点只读 `preprocess/scan_rate`。因此 `common/scan_rate` 没有进入程序。`preprocess/scan_rate` 又属于上一节的未传入项，实际保持默认值 10。数值上与 yaml 中写下的 10 相同，但不是从该键读到的。

`gnss/gnss_enable` 为 false 时，`if (GNSS_ENABLE)` 内的参数不读取。yaml 里这些 GNSS 键没有进入估计，包括星历与观测话题、门限、噪声和窗口。无条件读取、因而仍然生效的 GNSS 相关键只有第 3、4 节列出的 `gnss_enable`、`gt_file_type`、`gnss_ekf_noise`、`gnss_extrinsic_T`、`gnss_extrinsic_R`。

`nmea/nmea_enable` 为 false 时，`if (NMEA_ENABLE)` 内的参数不读取。yaml 中的 `posit_odo_topic`、`ppp_std_thres`、`pos_noise`、`nmea_weight`、`nmea_local_time_diff`、`ppp_file_name` 都没有进入程序。`nmea/ppp_anc` 在该判断之外，已按第 3 节传入。

`/leg_odom` 没有对应参数，也没有订阅。
