# M-Detector-ROS2

ROS2 Humble 版本的 M-detector（Moving Event Detection），用于对 LiDAR 点云进行动态事件检测与动态点输出。此工程不仅完成了从 ROS1 的完整迁移，还新增了 Livox/PointCloud2 兼容、传感器静止模式（无里程计依赖）、oneTBB 全局统一并行化、ROS2 参数/launch 规范化与多设备配置等工程落地化改造。

## 1. 简介

M-detector 基于遮挡原理，对每个到达的点进行快速动态判别，具备低延迟与较高泛化性。该实现面向工程落地，支持 Livox 设备与标准 PointCloud2 输入。

来源：
该工作源自 香港大学 MARS Lab（The University of Hong Kong），论文发表于 Nature Communications。


论文：
[Moving event detection from LiDAR point streams](https://www.nature.com/articles/s41467-023-44554-8)

视频：

<div align="center">
<a href="https://www.youtube.com/watch?v=SYaig2eHV5I" target="_blank"><img src="img/cover.bmp" alt="video" width="75%" /></a>
</div>

## 2. 兼容环境

- Ubuntu 22.04
- ROS2 Humble
- colcon
- PCL (>= 1.8)
- Eigen (>= 3.3.4)
- oneTBB (`libtbb-dev`)

安装示例：
```
sudo apt install libpcl-dev libeigen3-dev libtbb-dev
```

## 3. 工作空间结构

推荐结构：
```
ros2_ws/
  src/
    m_detector/
```

> Livox 驱动建议使用外部工作空间，运行前请先 source 该驱动的 install 环境（需包含 `livox_interfaces`）。

> [!IMPORTANT]
> 请确保 `livox_interfaces` 在当前环境中可被发现，否则编译或运行会失败。

## 4. 获取与构建

### 4.1 拉取源码

```
git clone https://github.com/BreCaspian/M-Detector-ROS2.git
```

### 4.2 构建编译

将仓库放入工作空间 `src` 下：
```
mkdir -p ~/ros2_ws/src
cp -r M-Detector-ROS2 ~/ros2_ws/src/m_detector
```

编译并加载环境：
```
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

## 5. 运行

> 如已编译完成，可直接进入运行步骤。

### 5.1 实时检测（示例）

```
ros2 launch m_detector detector_mid70.launch.py
```

> [!TIP]
> 若只发布 `sensor_msgs/PointCloud2`，请在配置中设置 `dyn_obj/use_livox_custom: false`，并把 `dyn_obj/points_topic` 指向真实话题名。

其他内置配置：
```
ros2 launch m_detector detector_mid360.launch.py
ros2 launch m_detector detector_horizon.launch.py
ros2 launch m_detector detector_avia.launch.py
```

### 5.2 RViz 可视化

默认使用 `rviz/demo.rviz`，显示 `/m_detector/*` 话题输出。
如不需要 RViz：
```
ros2 launch m_detector detector_mid70.launch.py rviz:=false
```

## 6. 输入与输出

### 6.1 输入

- Livox CustomMsg（默认）：
  - 话题：`/livox/lidar`
  - 参数：`dyn_obj/use_livox_custom: true`
- 标准 PointCloud2：
  - 话题：`dyn_obj/points_topic`
  - 参数：`dyn_obj/use_livox_custom: false`

里程计输入：
- `dyn_obj/odom_topic`（默认 `/aft_mapped_to_init`）

### 6.2 输出

- `/m_detector/point_out`：点级动态结果（PointCloud2）
- `/m_detector/frame_out`：帧级聚类后动态结果（PointCloud2）
- `/m_detector/std_points`：静态点云（PointCloud2）

可选输出（文件）：
- `dyn_obj/out_file` / `dyn_obj/out_file_origin`：保存标签
- `dyn_obj/cluster_out_file`、`dyn_obj/time_file`、`dyn_obj/time_breakdown_file`：统计输出

## 7. 静止模式（无里程计）

当 LiDAR 静止且点云坐标系稳定时，可不输入里程计：

```
"dyn_obj/use_odom": false
"dyn_obj/static_pos": [0.0, 0.0, 0.0]
"dyn_obj/static_quat": [1.0, 0.0, 0.0, 0.0]
```

说明：
- `use_odom=false` 时不订阅里程计话题。
- `static_pos/static_quat` 预留固定外参，四元数顺序为 `[w, x, y, z]`。
- 若传感器有运动，请保持 `use_odom=true`。

> [!CAUTION]
> 静止模式仅适用于传感器完全静止的场景，若存在运动会引入明显误检。

## 8. 参数与配置

参数文件位于 `config/`，采用 ROS2 `ros__parameters` 格式。
建议先使用对应设备的默认配置（如 `config/mid70/mid70.yaml`）。

关键输入参数：
```
"dyn_obj/points_topic": "/livox/lidar"
"dyn_obj/use_livox_custom": false
"dyn_obj/odom_topic": "/aft_mapped_to_init"
"dyn_obj/frame_dur": 0.1
```

## 9. 离线评估（可选）

```
ros2 launch m_detector cal_recall.launch.py \
  dataset:=0 dataset_folder:=/path/to/dataset \
  start_se:=0 end_se:=0 start_param:=0 end_param:=0 is_origin:=false
```

## 10. 常见问题

**RViz 没有显示动态点**
- 检查 `/m_detector/point_out` 是否有频率输出。
- 若 `/livox/lidar` 同时存在 CustomMsg 和 PointCloud2，请确认：
  - `dyn_obj/use_livox_custom` 与实际发布类型一致。

> [!TIP]
> 可用 `ros2 topic info --verbose /livox/lidar` 快速确认真实发布类型，再调整订阅配置。
**/livox/lidar 无法 echo**
- 话题类型冲突时可用：
  ```
  ros2 topic info --verbose /livox/lidar
  ```
  并改为订阅发布中的实际类型。

## 11. 许可证

This project is licensed under the **GNU General Public License v2.0 (GPL-2.0)**.

- See the `LICENSE` file for the full license text.
- This project follows the **copyleft** requirements of GPL-2.0.

## 12. 致谢与作者

原作者：
- Huajie Wu (wu2020@connect.hku.hk)
- Yihang Li (yhangli@connect.hku.hk)

本仓库为 ROS2 迁移与工程化版本，保留原作者署名与算法来源。
