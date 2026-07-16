# Isaac Sim 6.0.1 官方文档整理

> 来源：https://docs.isaacsim.omniverse.nvidia.com/6.0.1/index.html
> 整理日期：2026-07-16。全站约 250+ 页、13 个大板块，由 10 个研读代理分板块通读后汇总为以下 10 份中文摘要。
> Isaac Sim 6.0.1 为当前最新 GA 版本（Kit SDK 110.1.2）。

## 6.0 相对 5.x 的核心变化（先读这里）

1. **Newton 物理引擎**（实验性，Warp + MuJoCo Warp）与引擎无关的 `isaacsim.core.experimental` API；新增 `Robot_Multiphysics` 资产目录、Newton Actuators 执行器体系
2. **传感器全面迁移**：`isaacsim.sensors.camera/rtx/physics/physx` 全部弃用 → `isaacsim.sensors.experimental.rtx/physics`；新增声学传感器、结构光相机、多 tick 渲染、Physics Raycast
3. **ROS 2 OmniGraph 破坏性变更**：Publish TF / Joint State 需前置 Compute Transform Tree / Read Joint State 节点；`frameSkipCount` 弃用改 `omni:sensor:tickRate`
4. **弃用大户**：Lula 运动栈（→ 实验性 Motion Generation / cuMotion / PINK）、Cortex、Robot Wizard、GUI 扩展模板生成器、旧 Core API
5. **新工具**：MCP Server、Python Server 远程执行、Asset Transformer、Robot Inspector/Poser、Self-Collision Detector、RTSP 流、Teleoperation SDG、LLM 行为树生成
6. **环境变化**：pip 安装改 **Python 3.12**、不再支持 Win10、OpenUSD 25.11、默认渲染器 RTX Real-Time 2.0

## 文件索引

| 文件 | 内容 |
|---|---|
| [01-overview-installation-migration.md](01-overview-installation-migration.md) | 版本要点、**8 篇 6.0 迁移指南详解（旧→新 API 对照）**、系统需求、安装、FAQ |
| [02-assets-quickstart.md](02-assets-quickstart.md) | 资产库（Robot_Multiphysics、NuRec）、资产结构、快速入门、工作流 |
| [03-importers-robot-setup.md](03-importers-robot-setup.md) | URDF/MJCF 导入（Newton schema）、Asset Transformer、Robot Inspector/Poser、13 篇搭建教程、OpenUSD 调参最佳实践 |
| [04-dev-tools-python-omnigraph.md](04-dev-tools-python-omnigraph.md) | MCP Server、Python Server、Experimental API 现状、Core API 教程、OmniGraph（含 IPC 节点） |
| [05-ros2.md](05-ros2.md) | Bridge、6.0 节点迁移、压缩图像、Radar、heightmap 导航、仿真控制服务、端到端教程 |
| [06-isaac-lab-app-template.md](06-isaac-lab-app-template.md) | Isaac Lab、策略部署、Cloner、Instanceable、App Template 并入主仓库 |
| [07-robot-simulation-sensors-physics.md](07-robot-simulation-sensors-physics.md) | 控制器、实验性运动栈（cuMotion/PINK）、Newton Actuators、五类传感器（experimental）、Newton 后端与物理数据流 |
| [08-synthetic-data-generation.md](08-synthetic-data-generation.md) | Replicator 函数式 API、SDG Workflows、IRA 1.x、行为树生成、遥操作 SDG、MobilityGen |
| [09-digital-twin-cortex.md](09-digital-twin-cortex.md) | 新版 Warehouse Creator、RTSP 流、占用图、Cortex（已弃用） |
| [10-utilities-usd-reference.md](10-utilities-usd-reference.md) | CLI 扩展模板、调试/Tracy、渲染模式、性能手册、Robot/Sensor Schema、C++ USD 插件 |

## 关键速查

- **单位/坐标**：米-千克-秒，右手系 +Z 上 +X 前；Core/USD 四元数 (w,x,y,z)，PhysX 为 (x,y,z,w)；USD 属性用度、物理计算用弧度
- **安装**：pip 需 **Python 3.12** + 先装 `torch==2.11.0`，`pip install "isaacsim[all,extscache]==6.0.1.0" --extra-index-url https://pypi.nvidia.com`；容器 `nvcr.io/nvidia/isaac-sim:6.0.1`
- **最低硬件**：Ubuntu 22.04/24.04 或 Win 11，32GB 内存，RTX GPU（A100/H100 不支持），驱动 Linux 595.58.03+
- **Newton 启用**：`isaac-sim.newton.sh` 或 `SimulationManager.switch_physics_engine("newton")`；资产须无闭链、质量/惯量非零
- **新代码规范**：一律用 `isaacsim.core.experimental` 与 `isaacsim.sensors.experimental.*`；旧 API 均在弃用通道
