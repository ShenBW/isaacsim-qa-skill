---
name: isaacsim-601-qa
description: Use when the user asks about NVIDIA Isaac Sim 6.0.x (6.0.0/6.0.1, the latest version) or asks Isaac Sim questions WITHOUT specifying a version — installation (Python 3.12 pip/container), Newton physics backend, Newton Actuators, experimental sensors API (camera, RTX Lidar/Radar/Acoustic, raycast), URDF/MJCF import, Asset Transformer, robot rigging, joint tuning, ROS 2 bridge, Nav2, MoveIt, Isaac Lab, Replicator SDG, teleoperation SDG, cuMotion/PINK motion, MCP Server, migrating from 5.x to 6.0, errors/crashes (仿真、导入、报错、迁移、性能). For questions explicitly about Isaac Sim 5.1, use isaacsim-qa instead.
---

# Isaac Sim 6.0.1 问答

## 核心原则

回答 Isaac Sim 6.0.x 问题时，**先查本 skill 目录下的 `references/` 摘要库，本地不足再网络搜索**。禁止只凭训练记忆回答——6.0 相对 5.x 有大量破坏性变更（传感器/ROS 2 节点/Core API 全面迁移），记忆中的旧 API 极易出错，版本细节必须有出处。

> `references/` 的完整路径 = skill 加载时显示的 "Base directory for this skill" + `/references/`。
> 用户问题明确指向 Isaac Sim 5.1 时，改用 `isaacsim-qa` skill；未指明版本时按 6.0.1（最新版）回答，并在答案中注明所依据的版本。

## 流程

1. **定位**：按下表选 1–2 个摘要文件，用 Read 读取（`references/README.md` 含总索引、6.0 核心变化速览与关键速查，不确定时先读它）。
2. **回答**：能解决就直接答，引用摘要中保留的官方文档 URL 作为出处。
3. **判断是否足够**：以下情况视为"本地不足"→ 进入第 4 步：
   - 摘要只有一句概述，用户需要具体步骤/完整 API 签名/报错原文匹配
   - 问题涉及摘要未覆盖的页面（如小教程、具体扩展参数）
   - 用户报错信息在摘要排障小节中找不到
4. **网络兜底**：先 `date '+%Y-%m-%d'` 取当前日期；搜索词必须含 "Isaac Sim 6.0" 与年份。优先信源：`docs.isaacsim.omniverse.nvidia.com/6.0.1`（摘要中的来源 URL 可直接 WebFetch 原页）、forums.developer.nvidia.com、github.com/isaac-sim/IsaacSim（6.0 起开源，可直接查源码）。
5. **答案末尾标注来源**：本地摘要（哪个文件）/ 官方文档原页 / 论坛-GitHub，混合来源分别标注。

## 主题 → 文件速查（均位于 `references/`）

| 问题主题 | 读取文件 |
|---|---|
| 安装、硬件需求、容器、pip（Python 3.12）、驱动、**5.x→6.0 迁移（8 篇指南）** | 01-overview-installation-migration.md |
| 官方资产、Robot_Multiphysics、NuRec、环境、工作流选择 | 02-assets-quickstart.md |
| URDF/MJCF 导入（Newton schema）、Asset Transformer、Robot Inspector/Poser、rig、关节调参、OpenUSD 调参最佳实践 | 03-importers-robot-setup.md |
| Core Experimental API、python.sh、MCP Server、Python Server、OmniGraph（含 IPC 节点）、Carb 设置 | 04-dev-tools-python-omnigraph.md |
| ROS 2 桥、6.0 节点迁移、压缩图像、Nav2/heightmap、MoveIt、仿真控制服务 | 05-ros2.md |
| Isaac Lab、RL 策略部署、Cloner、Instanceable | 06-isaac-lab-app-template.md |
| 控制器、cuMotion/PINK 运动栈、Newton Actuators、传感器（experimental/acoustic/raycast）、Newton 物理后端 | 07-robot-simulation-sensors-physics.md |
| Replicator、SDG Workflows、IRA、行为树生成、遥操作 SDG、MobilityGen | 08-synthetic-data-generation.md |
| 仓库生成、传送带、RTSP 流、占用图、Cortex（已弃用） | 09-digital-twin-cortex.md |
| CLI 扩展模板、调试/Tracy、渲染模式、性能优化、Robot/Sensor Schema、约定 | 10-utilities-usd-reference.md |

## 常见错误

- ❌ 跳过本地摘要直接搜网：慢、且搜索结果常混入 4.x/5.x 旧版答案。
- ❌ 凭记忆报 API 名：6.0 中 `isaacsim.sensors.camera/rtx/physics/physx`、Lula、Cortex、旧 Core API 全部弃用，新代码应使用 `isaacsim.core.experimental` 与 `isaacsim.sensors.experimental.*`。
- ❌ 用 5.x 的 ROS 2 图结构回答：Publish TF / Joint State 在 6.0 需前置 Compute Transform Tree / Read Joint State 节点。
- ❌ 网络搜索不带版本号和年份。
- 用户从 5.x 升级遇到问题时，优先查 01 文件的迁移指南小节做新旧 API 对照。
