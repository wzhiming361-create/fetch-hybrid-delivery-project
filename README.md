# Fetch Hybrid Delivery System

这是一个基于 **Fetch Mobile Manipulator** 的混合架构移动抓取与递送系统。

## 项目架构

本项目采用了 **ROS 1 + ROS 2 混合架构**，以在保留 Fetch 机器人稳定驱动（ROS 1 Melodic）的同时，利用 ROS 2 的先进特性进行视觉感知。

* **宿主机 (ROS 1 Melodic)**: 运行硬件驱动、MoveIt! 运动规划、MoveBase 导航以及主控逻辑 (`fetch_controller.py`)。
* **Docker 容器 (ROS 2 Foxy)**: 运行基于 PCL 的视觉感知节点 (`fetch_perception`) 以及 `ros1_bridge`。
* **通信**: 容器与宿主机共享网络，通过 `ros1_bridge` 将 ROS 2 的感知结果 (`/box_target`) 转发给 ROS 1 控制器。

## 目录结构

```text
hybrid_fetch_project/
├── docker/                           # Docker 环境配置
│   ├── bridge_mapping.yaml           # 跨版本消息映射规则
│   ├── docker-compose.yml            # 容器启动配置
│   ├── Dockerfile                    # ROS 1 Noetic + ROS 2 Foxy 镜像构建
│   └── entrypoint.sh                 # 环境加载脚本
├── ros1_ws/                          # [宿主机] ROS 1 工作空间
│   └── src/
│       ├── fetch-delivery-system/    # 主系统 (已修改适配混合架构)
│       └── grasping/                 # 独立的测试与演示脚本
└── ros2_ws/                          # [Docker] ROS 2 工作空间
    └── src/
        ├── fetch_delivery_system_msgs/ # 消息定义 (用于桥接)
        └── fetch_perception/           # 核心感知节点
```
## 环境要求

* **硬件**: Fetch Mobile Manipulator (或安装了 `fetch_gazebo` 的仿真电脑)。
* **操作系统**: Ubuntu 18.04 (宿主机)。
* **软件**:
    * ROS Melodic (宿主机)
    * Docker & Docker Compose

## 安装与部署

### 1. 宿主机 (ROS 1) 准备

将 `ros1_ws` 中的代码部署到机器人或仿真电脑上。

```bash
# 假设你在项目根目录下
cd ros1_ws

# 编译 ROS 1 工作空间
catkin_make

# 使得环境生效
source devel/setup.bash
```

重要配置: 修改地图路径 打开 `ros1_ws/src/fetch-delivery-system/scripts/fetch_navigation.py`，找到 map_file 变量，将其修改为你机器上的绝对路径：

```Python3
# 示例
map_file = "/home/fetch/hybrid_fetch_project/ros1_ws/src/fetch-delivery-system/fetch_maps/fetch_traditional_maps.yaml"
```

### 2. Docker 环境 (ROS 2) 准备

构建包含 ROS 2 和 Bridge 的 Docker 镜像。

```bash
cd ../docker

# 构建并启动容器 (后台运行)
docker-compose up -d --build
```

### 3. 编译 ROS 2 代码

我们需要进入容器内部编译感知节点。

```bash
# 进入容器
docker exec -it fetch_hybrid_bridge bash

# --- 以下命令在容器内执行 ---
cd /root/ros2_ws

# 编译 (使用 symlink 安装以便开发)
colcon build --symlink-install

# 退出容器
exit
```

## 运行

请按照以下顺序在不同的终端 (Terminal) 中启动系统。

第一步：启动 ROS 1 基础系统 (宿主机)

在宿主机终端中

```bash
cd ros1_ws
source devel/setup.bash

# 启动 MoveIt, 导航配置 (已禁用了旧版感知节点)
roslaunch fetch_delivery_system grasping.launch
```

第二步：启动 ROS 2 感知节点 (Docker)

打开一个新的终端：

```bash
docker exec -it fetch_hybrid_bridge bash

# --- 容器内 ---
# 启动感知节点
ros2 launch fetch_perception perception.launch.py
```

第三步：启动 ROS 1 Bridge (Docker)

打开一个新的终端：
```bash
docker exec -it fetch_hybrid_bridge bash

# --- 容器内 ---
# 启动桥接，加载映射规则
# bash -lc "source /opt/ros/noetic/setup.bash && source /opt/ros/foxy/setup.bash && ros2 run ros1_bridge parameter_bridge /root/bridge_mapping.yaml"

# ros2 run ros1_bridge parameter_bridge /root/bridge_mapping.yaml

# bash -lc "source /opt/ros/noetic/setup.bash && source /opt/ros/foxy/setup.bash && ros2 run ros1_bridge dynamic_bridge"

source /opt/ros/noetic/setup.bash
source /opt/ros/foxy/setup.bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-topics \
  -m geometry_msgs/PoseStamped \
  -m visualization_msgs/Marker
```

第四步：启动主控制器 (宿主机)

打开一个新的终端：
```bash
cd ros1_ws
source devel/setup.bash

# 启动导航逻辑 (确保地图路径已修改)
rosrun fetch_delivery_system fetch_navigation.py &

# 启动抓取与递送逻辑
# 注意：不要使用第三方包 fetch_api 来运行控制器。
# 如果脚本或环境依赖于 `fetch_api`，请改用 `fetch_ros` 提供的 ROS 接口，
# 或在真实机器人（有 fetch 驱动）或安装了 `fetch_gazebo` 的仿真环境中运行。
rosrun fetch_delivery_system fetch_controller.py
```