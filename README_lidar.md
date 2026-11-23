# 接入机器狗后订阅 X30 点云

在 NS16 上执行。本机地址为 `192.168.1.110`。点云在 `/home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag`，话题为 `/livox/lidar`，类型为 `livox_ros_driver2/CustomMsg`。

## 1. 登录

```sh
ssh nvidia@192.168.1.110
```

用户名和密码均为 `nvidia`。

## 2. 环境

每个终端都执行：

```sh
set +u
source /opt/ros/noetic/setup.bash
source /home/nvidia/ligo_ws/devel/setup.bash
set -u
export ROS_MASTER_URI=http://127.0.0.1:11311
export ROS_IP=127.0.0.1
unset ROS_HOSTNAME
```

## 3. 查看

终端 A：

```sh
roscore
```

终端 B：

```sh
rosbag play -r 1.0 /home/nvidia/x30_ligo_test/LIO_2026-11-13-16-32-56.bag
```

终端 C：

```sh
rostopic hz /livox/lidar
rostopic echo -n 1 /livox/lidar/point_num
```

频率约 10 Hz。`point_num` 为 `79968`，坐标系 `livox_frame`。

程序内订阅同一话题、同一类型：

```python
from livox_ros_driver2.msg import CustomMsg
rospy.Subscriber("/livox/lidar", CustomMsg, callback, queue_size=1)
```
