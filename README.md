# YESLab Gazebo + VINS-Fusion + PX4 视觉定位飞行

## 一、实验环境

| 组件 | 版本 |
|---|---|
| 操作系统 | Ubuntu 20.04 LTS |
| ROS | Noetic (1.17.4) |
| PX4 Autopilot | v1.14.0 (SITL) |
| Gazebo | Classic 11 |
| MAVROS | noetic 版本 |
| VINS-Fusion | 开源版本（基于 HKU MARS 仓库） |
| 虚拟机 | VMware Workstation |

## 二、系统数据流

azebo Iris 相机 + IMU
↓
VINS-Fusion（视觉惯性里程计）
↓
/vins_estimator/odometry (ENU/FLU)
↓
vins_to_px4.py（坐标转换 ENU→NED + 原点对齐）
↓
/mavros/vision_pose/pose
↓
MAVROS
↓
PX4 EKF2 融合视觉定位
↓
Local Position 有效
↓
Position (POSCTL) 模式飞行

```

## 三、实际 Topic

| 功能 | 话题 | 类型 | 频率 |
|---|---|---|---|
| 相机图像 | /iris/front_camera/image_raw | sensor_msgs/Image | ~24Hz |
| IMU 数据 | /mavros/imu/data | sensor_msgs/Imu | ~50Hz |
| VINS 里程计 | /vins_estimator/odometry | nav_msgs/Odometry | ~10Hz |
| VINS 轨迹 | /vins_estimator/path | nav_msgs/Path | - |
| PX4 视觉输入 | /mavros/vision_pose/pose | geometry_msgs/PoseStamped | ~10Hz |
| 本地位置 | /mavros/local_position/pose | geometry_msgs/PoseStamped | ~30Hz |

## 四、完整启动顺序

### 终端1：roscore
```bash
roscore
```

### 终端 2：PX4 + Gazebo 仓库世界

```
cd ~/PX4-Autopilot
make px4_sitl_default gazebo-classic_iris__warehouse
```

### 终端 3：MAVROS

```
source /opt/ros/noetic/setup.bash
roslaunch mavros px4.launch fcu_url:="udp://:14540@"
```

### 终端 4：VINS-Fusion

```
source /opt/ros/noetic/setup.bash
source ~/catkin_ws/devel/setup.bash
rosrun vins vins_node ~/catkin_ws/src/VINS-Fusion/config/simulation/vins_config.yaml
```

### 终端 5：起飞 + 初始化 VINS

```
# 在 PX4 终端输入：commander takeoff
source /opt/ros/noetic/setup.bash
python3 ~/YESLab-Gazebo-VINS-PX4/scripts/vins_to_px4.py
# 先运行移动脚本让 VINS 初始化
python3 ~/move3.py
```

### 终端 6：PX4 视觉定位参数（在 nsh 终端输入）

```
param set EKF2_AID_MASK 1
param set EKF2_HGT_REF vision
param set EKF2_EVP_NOISE 0.05
param save
```

### 终端 7：进入 Position 模式

```
# 在 PX4 终端输入
commander arm -f
commander mode posctl
## 五、VINS 结果

RViz 不知道为啥没有显现，已通过ai排除了大部分因素
## 六、PX4 定位结果

EKF 状态：ekf2:0 EKF dt: 0.0080s, attitude: 1, local position: 1, global position: 1
## 七、Position 模式结果

成功进入 AUTO_LOITER / POSCTL 模式，悬停 30 秒稳定，前后左右移动正常。
## 八、问题记录

### 问题：VINS 初始化失败，持续报 "Not enough features or parallax"

**现象：**
在 empty.world 空世界中，VINS 始终无法初始化，vio.csv 为空。

**原因：**
空世界中相机画面几乎全灰，没有足够的视觉特征点，VINS 无法通过三角化初始化尺度。

**解决方法：**
换用 warehouse.world 仓库世界，货架、箱子、托盘提供了丰富的视觉特征，VINS 一次初始化成功。

**最终结果：**
vio.csv 输出连续轨迹数据，VINS 正常运行。

---

### 附加问题：VINS 偶发总线错误崩溃

**现象：**
VINS 运行一段时间后突然崩溃，报 "总线错误 (核心已转储)"。

**原因：**
虚拟机 CPU 性能不足，VINS 处理一帧时间过长（1.6 秒 vs 正常 0.05 秒），帧丢失导致预积分数值不稳定，最终堆内存损坏。

**解决方法：**
关闭 gzclient 界面（gzclient:=0），降低系统负载。
### 问题：RViz 中无法显示 VINS 连续轨迹

**现象：**
在 RViz 中添加 `/vins_estimator/path` 和 `/vins_estimator/odometry` 话题后，看不到轨迹线，Fixed Frame 选 `world` 时显示 "Unknown frame world" 错误。
