# Isaac Sim 5.1.0 官方文档整理

> 来源：https://docs.isaacsim.omniverse.nvidia.com/5.1.0/index.html
> 整理日期：2026-07-16。全站约 200 页、13 个大板块，由 10 个研读代理分板块通读后汇总为以下 10 份中文摘要。
> ⚠️ 注意：官方已标注 **5.1.0 停止支持（EOL）**，缺陷修复与新功能只进后续版本；6.0 将彻底移除所有已弃用扩展。

## 总览

Isaac Sim 是构建在 Omniverse Kit (107.3.3) 上的机器人仿真参考应用，核心链路为：
**资产导入（URDF/MJCF/CAD → USD）→ 机器人搭建（articulation/关节驱动/传感器）→ 仿真（PhysX + RTX 渲染）→ 应用（ROS 2 / 强化学习 Isaac Lab / 合成数据 Replicator / 数字孪生）**。
四种工作流：GUI、Extension（应用内热重载）、Standalone Python（`python.sh`，手动控制步进，适合 RL/SDG）、Jupyter。

## 文件索引

| 文件 | 内容 | 对应文档板块 |
|---|---|---|
| [01-overview-installation.md](01-overview-installation.md) | Isaac Sim 定位、5.1 版本要点、系统需求、工作站/容器/云端/pip 安装、ROS 2 环境、FAQ 与排障 | What Is Isaac Sim / Release Notes / Installation / Help & FAQ |
| [02-assets-quickstart.md](02-assets-quickstart.md) | 官方 USD 资产库（机器人/传感器/环境/Nova Carter/SimReady/NuRec）、资产目录结构规范、快速入门教程、示例体系、参考架构与工作流对比 | Isaac Sim Assets / Quick Start / Concepts |
| [03-importers-robot-setup.md](03-importers-robot-setup.md) | URDF/MJCF 导入与 URDF 导出、Robot Wizard、Gain Tuner、Robot Assembler、13 篇机器人搭建教程（rig/调参/优化/腿式）、资产校验 | Importers and Exporters / Robot Setup |
| [04-dev-tools-python-omnigraph.md](04-dev-tools-python-omnigraph.md) | VS Code/Jupyter/Script Editor/Carb 设置、Core API 体系（World/Scene/Task/SimulationApp）、python.sh 环境、代码片段、Core API 教程系列、OmniGraph、GUI 参考 | Development Tools / Python Scripting / Omnigraph / GUI |
| [05-ros2.md](05-ros2.md) | ROS 2 Bridge（Humble/Jazzy）、OmniGraph 发布订阅、时钟、相机/Lidar/TF/里程计、关节与 Ackermann 控制、Nav2 单机多机、MoveIt 2、仿真控制服务、Isaac ROS | ROS 2 |
| [06-isaac-lab-app-template.md](06-isaac-lab-app-template.md) | Isaac Lab 定位、RL 策略部署全流程、经 ROS 2 跑策略、Cloner 并行克隆、Instanceable 资产、Kit App Template | Isaac Lab / Application Template |
| [07-robot-simulation-sensors-physics.md](07-robot-simulation-sensors-physics.md) | Articulation 控制、移动机器人控制器、运动生成栈（Lula RMPflow/RRT/轨迹生成/cuRobo）、Surface Gripper、Grasp Editor、五类传感器（相机/深度/RTX Lidar-Radar/物理/PhysX）、物理仿真基础与 PhysX 限制 | Robot Simulation / Sensors / Physics |
| [08-synthetic-data-generation.md](08-synthetic-data-generation.md) | Replicator（randomizer/annotator/writer/trigger）、Synthetic Data Recorder、场景级/物体级/Infinigen SDG、域随机化实战、Cosmos、Agent/Object SDG、VLM captioning、抓取 SDG、MobilityGen | Synthetic Data Generation |
| [09-digital-twin-cortex.md](09-digital-twin-cortex.md) | Warehouse Creator、传送带、静态资产、cuOpt 物流优化、占用栅格图、Cortex 决策框架（decider network、Franka/UR10 案例） | Digital Twin |
| [10-utilities-usd-reference.md](10-utilities-usd-reference.md) | 浏览器、扩展模板、VS Code 调试与 Tracy 性能分析、性能优化手册、基准测试、单位与坐标系约定、术语表、4.5 扩展重命名、OpenUSD 与 Robot Schema、许可证 | Utilities / Reference Information / Omniverse and USD / Common |

## 关键速查

- **单位/坐标**：米-千克-秒，右手系 +Z 上 +X 前；Core/USD 四元数 (QW,QX,QY,QZ)，PhysX 为 (QX,QY,QZ,QW)；USD 属性用度、物理计算用弧度。
- **安装**：工作站 zip 解压即用；容器 `nvcr.io/nvidia/isaac-sim:5.1.0`；pip `pip install isaacsim[all,extscache]==5.1.0 --extra-index-url https://pypi.nvidia.com`（必须 Python 3.11）。
- **最低硬件**：Ubuntu 22.04/24.04 或 Win 10/11，32GB 内存，RTX GPU（A100/H100 无 RT Core 不支持），驱动 580.65.06+。
- **控制铁律**：同一关节同一时刻只能一种控制模式；位置驱动 stiffness>0，速度驱动 stiffness=0；articulation API 用弧度。
- **扩展命名**：4.5 起 `omni.isaac.*` → `isaacsim.*`；Core API 将被 Core Experimental API 取代，建议新代码尽早迁移。
