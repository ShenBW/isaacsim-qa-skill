# 概述与安装

## Isaac Sim 是什么

NVIDIA Isaac Sim™ 是构建在 NVIDIA Omniverse (Kit 107.3.3) 上的参考应用，用于在物理精确的虚拟环境中开发、仿真、测试 AI 驱动的机器人。核心能力：

- **设计**：支持 Onshape、URDF、MJCF 导入，场景基于 USD 格式
- **仿真**：GPU 加速的 PhysX 物理引擎；相机、Lidar、IMU、接触等多传感器仿真；Replicator 合成数据生成；配合 Isaac Lab 做强化学习
- **部署**：ROS 2 接口、Isaac ROS、数字孪生
- **工作流**：GUI 应用、Python 脚本、ROS 2、Jupyter Notebook；插件式可扩展架构，多数组件开源

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/index.html

## 5.1 版本要点

- Kit 升级至 107.3.3，新增 DGX Spark (aarch64) 支持；Compatibility Checker 集成进主程序，不再独立
- 所有已弃用扩展将在 6.0 完全移除
- 物理：改进 articulation 碰撞接触求解，减少夹爪穿透；新增夹爪关节调参教程
- 机器人资产：新增 5 款 Schunk 机器人、Booster T1、G1 手部变体、UR10e 跟随目标示例
- 传感器：RTX Sensor 支持语义分割 (Object ID)、非视觉材质 USD 属性、IMU 设备端处理
- ROS 2：支持 Simulation Interfaces v1.1.0（世界管理服务）、rclpy executor 性能优化、Humble/Jazzy 库管理改进
- 容器：多架构支持，默认以 rootless 用户 (UID 1234) 运行；aarch64 暂不支持 livestream
- 合成数据：新增 Cosmos SDG 教程，更新 Actor SDG 的 Navigation Mesh API

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/overview/release_notes.html

## 系统需求

- **操作系统**：Ubuntu 22.04/24.04；Windows 10/11（Win10 支持至 2025-10-14）
- **硬件 (x86_64)**：

| 项目 | 最低 | 推荐 | 理想 |
|---|---|---|---|
| CPU | i7 7代 / Ryzen 5, 4核 | i7 9代 / Ryzen 7, 8核 | i9 / Ryzen 9, 16核 |
| 内存 | 32GB | 64GB | 64GB |
| 存储 | 50GB SSD | 500GB SSD | 1TB NVMe |
| GPU | RTX 4080 (16GB) | RTX 5080 (16GB) | RTX PRO 6000 Blackwell (48GB) |

- **驱动**：Linux 580.65.06 / Windows 580.88；aarch64 仅限 DGX Spark（驱动 580.95.05）
- 无 RT Core 的 GPU（A100、H100）不兼容；容器仅支持 Linux；需联网获取资产

来源: https://docs.isaacsim.omniverse.nvidia.com/5.1.0/installation/requirements.html

## 安装方式对比与关键步骤

**下载**：提供 `isaac-sim-standalone-5.1.0-{windows-x86_64|linux-x86_64|linux-aarch64}.zip`、WebRTC 流客户端 1.1.5（Linux/Windows/macOS）、资产包（机器人/材质/环境或完整包 3 部分，含 MD5）。注意 Omniverse Launcher、Nucleus Workstation/Cache 自 2025-10-01 起弃用。
来源: installation/download.html

**工作站安装（推荐入门，约 3 小时内）**：解压 zip 到 `C:/isaacsim` 或 `~/isaacsim` → 运行 `post_install.sh`/`.bat`（创建教程符号链接）→ 启动 App Selector（`./isaac-sim.selector.sh` / `isaac-sim.selector.bat`）选 "Isaac Sim Full" → START。首次启动需编译着色器缓存，5-10 分钟窗口空白属正常。也可直接运行 `./isaac-sim.sh`；`--reset-user` 重置配置；`clear_caches.sh` 清缓存。无需 Nucleus/Hub。
来源: installation/quick-install.html, installation/install_workstation.html

**容器安装（仅 Linux）**：安装 Docker + NVIDIA Container Toolkit，`docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi` 验证 → `docker pull nvcr.io/nvidia/isaac-sim:5.1.0` → 运行时设 `ACCEPT_EULA=Y`、`PRIVACY_CONSENT=Y`，挂载 `~/docker/isaac-sim/` 下缓存/日志/配置卷（需 `chown -R 1234:1234`）→ 容器内 `./runheadless.sh -v` 启动 headless 流，看到 "Isaac Sim Full Streaming App is loaded" 后用 WebRTC 客户端按 IP 连接。
来源: installation/install_container.html

**云端部署**：支持 Isaac Launchable、NVIDIA Brev、AWS、Azure、GCP、腾讯云、阿里云、火山引擎、百度云及远程工作站；Isaac Automator 可自动化部署（SSH、网页 VNC、远程桌面），支持 AWS/Azure/GCP/阿里云。
来源: installation/install_cloud.html

**Livestream 客户端**：官方推荐 WebRTC Streaming Client。服务端启动：二进制 `./isaac-sim.streaming.sh`（Win: `isaac-sim.streaming.bat`）、容器 `./runheadless.sh`、pip 版 `isaacsim isaacsim.exp.full.streaming --no-window`。公网连接需加 `--/app/livestream/publicEndpointAddress=<公网IP> --/app/livestream/port=49100`，开放 UDP 47998、TCP 49100。限制：每实例同时仅一种流方式、一个客户端；A100 与 aarch64 不支持。
来源: installation/manual_livestream_clients.html

## Python 与 ROS 2 环境

**Python (pip)**：必须 Python 3.11。建议先建虚拟环境（venv 或 `conda create -n env_isaacsim python=3.11`），然后：

```bash
pip install isaacsim[all,extscache]==5.1.0 --extra-index-url https://pypi.nvidia.com
```

Linux 需 GLIBC 2.35+；Windows 需开启长路径。启动 GUI 用 `isaacsim`，跑脚本用 `python script.py`；需先 `export OMNI_KIT_ACCEPT_EULA=YES` 接受协议。
来源: installation/install_python.html

**ROS 2**：支持 Humble 与 Jazzy（Ubuntu 24.04→Jazzy，22.04→Humble/Jazzy，Windows→WSL2+Humble）。因 Isaac Sim 仅兼容 Python 3.11 而系统 ROS 用 3.10/3.12，故有两条路：用系统 ROS + Isaac Sim 内置 Python 3.11 桥接库（默认接口），或自行用 Python 3.11 编译自定义包。内置库示例：

```bash
export ROS_DISTRO=humble
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$isaac_sim_package_path/exts/isaacsim.ros2.bridge/humble/lib
```

单机推荐 Fast DDS 共享内存；跨机/Docker 用 UDP；也可选 Cyclone DDS。WSL2 需转发端口 7400、7410、9387。
来源: installation/install_ros.html

## 常见问题与排障要点

- **路径**：配置数据在 Linux `~/.local/share/ov/data/Kit/Isaac-Sim`，Windows `%userprofile%\AppData\Local\ov\data\Kit\Isaac-Sim`，Docker `/root/.local/share/ov/data`
- **离线资产**：下载完整资产包解压至如 `~/isaacsim_assets/Assets/Isaac/5.1`，启动加 `--/persistent/isaac/asset_root/default="<本地路径>"`
- **首次加载慢**：材质/着色器编译所致，属正常；运行慢则简化场景与碰撞几何、调仿真步长
- **性能**：用 Tracy 分析瓶颈；CPU 占用高可启用 Fabric；降日志开销 `--/log/level=error`；多 GPU 用 `--/renderer/multiGpu/enabled=true`
- **已知问题**：Windows 黑屏加 `--vulkan`；视口分辨率超 VRAM 会崩溃；SDG 建议 DLSS Quality 模式；URDF 同名材质只生成一个、命名须符合 USD 规范；ROS 2 样例长时间运行可能内存泄漏；Windows 缺 MSVC v143 构建工具会导致模块加载失败；Linux headless 退出时的段错误可忽略
- **驱动**：Ubuntu 22.04.5 + 内核 6.8.0-48 建议驱动 ≥535.216.01，问题时改用 Production Branch `.run` 安装；禁用超频软件（如 MSI Afterburner）可解决除零错误；Isaac Sim 的 Python 不支持 tkinter
- **求助渠道**：Omniverse 论坛、Discord (discord.gg/nvidiaomniverse)、GitHub isaac-sim 仓库；注意 5.1.0 已停止支持，官方建议升级到新版本

来源: installation/install_faq.html, overview/faq_index.html, overview/troubleshooting.html, overview/known_issues.html, overview/help.html
