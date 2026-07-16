---
name: isaacsim-qa
description: Use when the user asks about NVIDIA Isaac Sim 5.1.0 or hits confusion/errors while using it — installation, pip/container/cloud setup, URDF/MJCF import, robot rigging, articulation, joint tuning, ROS 2 bridge, Nav2, MoveIt, Isaac Lab, RL policy deployment, Replicator, synthetic data (SDG), sensors (camera, RTX Lidar, Radar, IMU, contact), PhysX physics, OmniGraph, Cortex, performance, crashes (仿真、导入、报错、性能、机器人).
---

# Isaac Sim 5.1.0 问答

## 核心原则

回答 Isaac Sim 5.1.0 问题时，**先查本 skill 目录下的 `references/` 摘要库，本地不足再网络搜索**。禁止只凭训练记忆回答——版本细节（API 名、参数、命令）必须有出处。

> `references/` 的完整路径 = skill 加载时显示的 "Base directory for this skill" + `/references/`。

## 流程

1. **定位**：按下表选 1–2 个摘要文件，用 Read 读取（`references/README.md` 含总索引与速查，不确定时先读它）。
2. **回答**：能解决就直接答，引用摘要中保留的官方文档 URL 作为出处。
3. **判断是否足够**：以下情况视为"本地不足"→ 进入第 4 步：
   - 摘要只有一句概述，用户需要具体步骤/完整 API 签名/报错原文匹配
   - 问题涉及摘要未覆盖的页面（如小教程、具体扩展参数）
   - 用户报错信息在摘要排障小节中找不到
4. **网络兜底**：先 `date '+%Y-%m-%d'` 取当前日期；搜索词必须含 "Isaac Sim 5.1" 与年份。优先信源：`docs.isaacsim.omniverse.nvidia.com/5.1.0`（摘要中的来源 URL 可直接 WebFetch 原页）、forums.developer.nvidia.com、github.com/isaac-sim。
5. **答案末尾标注来源**：本地摘要（哪个文件）/ 官方文档原页 / 论坛-GitHub，混合来源分别标注。

## 主题 → 文件速查（均位于 `references/`）

| 问题主题 | 读取文件 |
|---|---|
| 安装、硬件需求、容器、pip、驱动、启动报错 | 01-overview-installation.md |
| 官方资产、机器人型号、环境、SimReady、工作流选择 | 02-assets-quickstart.md |
| URDF/MJCF 导入导出、rig、关节调参、Gain Tuner、资产优化 | 03-importers-robot-setup.md |
| Core API、python.sh、VS Code/Jupyter、OmniGraph、Carb 设置 | 04-dev-tools-python-omnigraph.md |
| ROS 2 桥、话题发布、Nav2、MoveIt、TF、QoS | 05-ros2.md |
| Isaac Lab、RL 策略部署、Cloner、Instanceable | 06-isaac-lab-app-template.md |
| 控制器、Lula/RMPflow/cuRobo、夹爪、传感器、PhysX 物理 | 07-robot-simulation-sensors-physics.md |
| Replicator、合成数据、域随机化、MobilityGen | 08-synthetic-data-generation.md |
| 仓库生成、传送带、cuOpt、占用图、Cortex | 09-digital-twin-cortex.md |
| 性能优化、单位/坐标/四元数约定、扩展重命名、USD、调试 | 10-utilities-usd-reference.md |

## 常见错误

- ❌ 跳过本地摘要直接搜网：慢、且搜索结果常混入其他版本的答案。
- ❌ 凭记忆报 API 名：4.5 起 `omni.isaac.*` 已改名 `isaacsim.*`，记忆极易过时。
- ❌ 网络搜索不带版本号和年份：Isaac Sim 各版本差异大。
- 提醒用户时机：涉及升级/弃用问题时，说明 5.1.0 已 EOL（修复只进新版本，6.0 移除全部弃用扩展）。
