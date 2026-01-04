# IMU轨迹估计与实时显示系统

imu_traj_ws是一个ROS/C++项目，用于基于IMU传感器数据进行高精度轨迹估计与实时可视化。该项目支持多种滤波算法，包括滑动窗口滤波和误差状态卡尔曼滤波(ESKF)，并实现了分轴ESKF误差模型，能够充分利用IMU分轴标定数据提高估计精度。

项目订阅`/imu/data`话题，进行姿态、位置和速度估计，并通过RViz实时可视化轨迹。系统支持零速检测(ZUPT)、重力校准、自适应噪声调整等功能。

## 依赖项

- ROS Noetic
- C++11
- 以下ROS包：
  - roscpp
  - sensor_msgs
  - geometry_msgs
  - visualization_msgs
  - tf2
  - tf2_ros

## 项目结构

```
imu_trajectory/
├── CMakeLists.txt           # CMake配置文件
├── IM42652_imu_param.yaml   # 默认IMU标定参数文件
├── config.yaml              # 项目主配置文件
├── include/                 # 头文件目录
├── launch/                  # 启动文件目录
│   └── imu_trajectory.launch # ROS启动文件
├── package.xml              # ROS包配置文件
├── rviz/                    # RViz配置文件目录
│   └── imu.rviz             # RViz可视化配置文件
└── src/                     # 源代码目录
    └── imu_trajectory.cpp   # 主程序文件
```

## 编译方法

1. 确保已安装ROS Noetic和所有依赖项
2. 克隆或下载此项目到ROS工作空间的`src`目录
3. 编译项目：

```bash
cd /home/ubuntu20/imu_traj_ws
catkin_make
```

## 运行方法

1. 首先确保ROS环境已设置：

```bash
source /home/ubuntu20/imu_traj_ws/devel/setup.bash
```

2. 启动IMU轨迹显示节点：

```bash
roslaunch imu_trajectory imu_trajectory.launch
```

或者使用`rosrun`：

```bash
rosrun imu_trajectory imu_trajectory_node
```

3. 确保IMU数据正在发布到`/imu/data`话题

4. 启动RViz来查看轨迹：

使用默认RViz配置文件（推荐）：
```bash
rviz -d /home/ubuntu20/imu_traj_ws/src/imu_trajectory/rviz/imu.rviz
```

或手动配置RViz：
```bash
rviz
```

在RViz中：
- 设置Fixed Frame为`world`
- 添加`Marker`显示，选择`imu_trajectory`话题
- 可以添加`TF`显示来查看IMU坐标系

## 配置说明

项目使用`config.yaml`文件进行配置，主要参数说明如下：

### 1. 滤波器参数
```yaml
filter_type: "eskf"  # 滤波器类型: sliding_window, exponential, gaussian, eskf
```

### 2. 滑动窗口滤波参数（当filter_type为sliding_window时生效）
```yaml
window_size: 100               # 滑动窗口大小
weight_type: "gaussian"          # 权重类型: equal(等权重), exponential(指数加权), gaussian(高斯权重)
gaussian_sigma: 100000           # 高斯滤波的标准差（仅当weight_type为gaussian时生效）
```

### 3. ESKF参数
```yaml
# 状态协方差初始值 (位置, 速度, 姿态, 陀螺仪偏置, 加速度计偏置)
p0: 0.01      # 位置初始协方差
v0: 0.1       # 速度初始协方差
q0: 0.001     # 姿态初始协方差
bg0: 0.1      # 陀螺仪偏置初始协方差
bb0: 0.1      # 加速度计偏置初始协方差
```

### 4. 自适应噪声调整参数
```yaml
innovation_threshold: 0.01  # 新息阈值，超过此值增大噪声协方差
noise_update_factor: 5      # 噪声增大因子
noise_reduce_factor: 0.001  # 噪声减小因子
```

### 5. 重力参数
```yaml
gravity_mode: "auto"          # 重力计算模式: auto(自动), manual(手动)
gravity: 9.81                 # 手动模式下的重力值 (m/s²)
zupt_gravity_calibration_threshold: 100 # 零速检测触发重力重新计算的连续零速帧数阈值
```

### 6. IMU标定参数
```yaml
calibration_file_path: "/home/ubuntu20/imu_traj_ws/src/imu_trajectory/IM42652_imu_param.yaml"  # IMU标定文件路径
```

### 7. 零速检测参数
```yaml
# 初始强制零速检测参数
enable_initial_zupt: false     # 是否开启初始强制零速检测
initial_zupt_duration: 10      # 初始强制零速检测持续时间（秒）

# 普通零速检测（ZUPT）参数
zupt_accel_threshold: 0.6      # 加速度阈值（x和y轴），小于此值认为处于零速
zupt_accel_z_threshold: 0.9    # 加速度z轴与重力差值阈值，小于此值认为处于零速
zupt_gyro_threshold: 0.1       # 角速度阈值，小于此值认为处于零速
zupt_cov_reduction_factor: 1   # 零速时ESKF速度误差协方差减小因子
```

### 8. 时间间隔参数
```yaml
time_interval_min: 0.0001      # 最小时间间隔阈值
time_interval_max: 0.1         # 最大时间间隔阈值
```

## 功能说明

1. **多滤波算法支持**：
   - **滑动窗口滤波**：支持等权重、指数加权和高斯加权平均
   - **ESKF（误差状态卡尔曼滤波）**：实现分轴误差模型，充分利用IMU分轴标定数据
   - 可通过配置文件切换滤波算法

2. **IMU数据处理**：
   - 订阅`/imu/data`话题获取加速度计和陀螺仪数据
   - 使用四元数积分进行高精度姿态估计
   - 支持IMU分轴标定数据加载与应用
   - 实现基于分轴标定数据的ESKF误差模型

3. **零速检测(ZUPT)**：
   - 支持初始强制零速检测
   - 基于加速度和角速度阈值的自动零速检测
   - 零速检测时自动降低协方差，提高估计精度

4. **重力校准**：
   - 支持自动重力校准模式
   - 支持手动设置重力加速度值
   - 基于零速检测的重力方向校准

5. **轨迹可视化**：
   - 通过`visualization_msgs/Marker`发布轨迹
   - 轨迹显示为红色线条
   - 支持轨迹标记点显示

6. **TF变换**：
   - 发布IMU坐标系相对于世界坐标系的变换
   - 可以在RViz中查看IMU的实时姿态

7. **自适应噪声调整**：
   - 基于新息的分轴自适应噪声调整
   - 自动根据测量误差调整系统置信度

8. **标定数据支持**：
   - 支持加载IMU分轴标定文件
   - 自动应用陀螺仪和加速度计分轴噪声参数
   - 支持零偏校准

## 注意事项

1. **标定数据使用**：为了获得最佳性能，建议提供IMU分轴标定数据文件。默认标定文件路径为：`/home/ubuntu20/imu_traj_ws/src/imu_trajectory/IM42652_imu_param.yaml`

2. **算法选择**：ESKF算法通常提供更高的估计精度，特别是在长时间运行时。滑动窗口滤波适合实时性要求高的场景。

3. **初始条件**：位置估计假设初始位置为原点，初始速度为零。可以通过配置文件启用初始零速检测来优化初始估计。

4. **重力校准**：系统支持自动重力校准，但在首次使用时建议确保IMU静止一段时间以获得准确的重力估计。

5. **噪声调整**：系统实现了基于新息的自适应噪声调整机制，可以根据实际测量误差自动调整系统置信度。

6. **计算资源**：ESKF算法的计算复杂度高于滑动窗口滤波，在资源受限的平台上可以考虑使用滑动窗口滤波。

## 许可证

MIT License
