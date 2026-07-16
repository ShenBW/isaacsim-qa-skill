# 数字孪生与 Isaac Cortex

## 数字孪生应用场景

Isaac Sim 5.1.0 的数字孪生文档将各类机器人工具整合为协同工作的系统示例，包含三大板块：**仓储物流**（仓库生成、传送带工具、静态资产、cuOpt 路径优化）、**Cortex**（协作机器人决策框架的递进式教程，从概念到 Franka/UR10 实例）以及**地图制作**（占用栅格图生成）。另设专门的故障排除页，按 Warehouse Creator（组件可见性、布局、纹理）、传送带（物理应用、碰撞、动画速度）、Cortex（decider network 初始化、资产加载路径）、占用图（碰撞几何缺失、raycast 参数、分辨率）分类给出诊断方法，强调检查控制台日志与资产可用性。

来源：https://docs.isaacsim.omniverse.nvidia.com/5.1.0/digital_twin/index.html 、digital_twin/troubleshooting.html

## Warehouse Creator 仓库生成

`omni.warehouse_creator` 扩展用模块化资产构建自定义形状仓库。经 Window > Extensions 安装后，从 Tools > Modular Warehouse Creator 启动，点击 Build Warehouse 进入视口绘制模式：按**逆时针**放点，点自动对齐仓库瓷砖尺寸，回到起点闭合轮廓。之后可在属性面板为墙体选择瓷砖样式（装卸台、通道板等，按直墙/内外角/中心件分类，需将选择模式切为 Component），并通过 Edit Column Placement 启用/禁用柱子（禁用显示半透明绿色，支持 Flip All）。注意：避免线条交叉、点距过近；建议本地下载 Isaac 资产以加速创建。

来源：digital_twin/warehouse_logistics/ext_omni_warehouse_creator.html

## 传送带工具

传送带扩展可将刚体变为传送带，提供两种方式：**基础节点**——选中刚体/网格后经 Create > Isaac Sim > Warehouse Items > Conveyor 创建 OmniGraph 节点，可配置速度、方向、曲线标志、纹理动画（曲线段为真时施加角速度而非线速度）；**Conveyor Track Builder**（Tools 菜单）——从预置资产库拼装完整传送带系统，提供 8 项配置（滚筒/皮带/双层样式、轨道类型、曲率、高程、变体），内置 49+ 种轨道变体，并支持自定义数据集（USD 需设默认 prim、原点沿 X 轴对齐、端点位于 Z=0、独立材质）。

来源：digital_twin/warehouse_logistics/ext_isaacsim_asset_gen_conveyor.html

## 静态仓库资产

教程演示用 NVIDIA 资产浏览器（Window > Browsers > NVIDIA Assets）搭建仓库：导入 Warehouse 建筑、货架、货物堆（如 WarehousePile_A04）。关键点：美术团队制作的 NVIDIA 资产常以厘米为单位，需缩放 0.01；纯视觉资产需手动添加刚体与碰撞体才能参与仿真，推荐在新 Stage 中制作带物理配置的变体后保存复用；**SimReady 资产**自带语义标签与物理配置，可直接用于数字孪生。

来源：digital_twin/warehouse_logistics/tutorial_static_assets.html

## cuOpt 物流优化集成

NVIDIA cuOpt 是路径优化求解服务，支持成本矩阵（欧氏距离）与加权路网图（适合室内）两种问题表示。集成靠三个扩展：`omni.cuopt.service`（与服务通信、数据预处理）、`omni.cuopt.examples`（四个递进示例：创建网络、简单成本矩阵、简单路网图、仓库内运输演示）、`omni.cuopt.visualization`（路网与语义区域可视化）。流程为：建节点/边构成路网 → 加载订单、车队与语义区域（标记高成本规避区）→ 调用 cuOpt 求解 → 在视口可视化最优路线。

来源：digital_twin/warehouse_logistics/logistics_tutorial_cuopt.html

## 占用栅格图生成

Occupancy Map 扩展基于场景物理碰撞几何，在给定高度生成"占用/空闲"二值地图。经 Tools > Robotics > Occupancy Map 打开，参数含：Origin（必须位于未占用位置，其 Z 值决定采样高度）、上下边界、Cell Size（米/像素）。可用 CENTER TO SELECTION / BOUND SELECTION 自动定位，CALCULATE 生成，VISUALIZE IMAGE 预览并自定义占用/空闲/未知颜色，支持 Stage 与 ROS 坐标转换及 180° 旋转。前提：几何体必须启用碰撞。配套 Block World Generator 可将占用图反向生成 3D 可碰撞环境。

来源：digital_twin/ext_isaacsim_asset_generator_occupancy_map.html

## Cortex 决策框架

**概览**：Isaac Cortex 旨在让协作机器人开发"像游戏开发一样简单"，核心是区分机器人内部的 belief 仿真与外部现实（仿真或实体），以 60Hz 执行六阶段流水线：感知 → 世界建模（写入 USD）→ 逻辑状态监控 → 决策（decider network）→ 命令 API → 底层控制同步。**Commander** 抽象按关节子集（如 Franka 的手臂与夹爪）暴露任务级命令接口。教程面向 standalone Python 工作流。

来源：cortex_tutorials/tutorial_cortex_1_overview.html

**Decider network 概念**：由 `DfDecider` 节点构成的有向无环图（`DfNetwork`），每个周期从根节点经 `decide()` 逐层选择子节点直至叶节点；节点激活/失活时调用 `enter()`/`exit()`。`DfContext` 承载逻辑状态监控变量与机器人命令 API。与状态机（`DfState.step()`）可通过 `DfStateMachineDecider` 互相嵌套，`DfStateSequence` 实现顺序状态链，从而构建比传统状态机更具反应性的分层行为。

来源：cortex_tutorials/tutorial_cortex_2_decider_networks.html

**Franka 积木堆叠**：4 块彩色积木按预定顺序堆塔，用户随时移动积木系统也能自适应。网络分三层：顶层依塔完成度调度取/放/回家；中层用 RLDS（鲁棒逻辑动力系统）决定取哪块、放哪里（Pick 为开夹爪→抓取→伸向积木的优先序列，Place 为到位→放置）；底层是原子 pick/place 状态机，以网络锁防止抓取中被抢占。`BuildTowerContext` 含五个监控器：同步积木位置信念、推断塔构型、检测夹爪持物、抑制交互碰撞、诊断输出。运行：`isaac_python franka_examples_main.py --behavior=block_stacking_behavior`。

来源：cortex_tutorials/tutorial_cortex_4_franka_block_stacking.html

**UR10 料箱堆叠**：UR10 配吸盘夹爪，将传送带上的料箱码到托盘，方向不对的先送翻转台。与 Franka 的深层网络不同，此例是**浅层** Dispatch 节点依逻辑条件（堆叠是否完成、料箱是否吸附/需翻转）分发到 PickBin、FlipBin、PlaceBin、回家等顺序状态机，各自用 `DfSetLockState` 保护原子序列。特色：`ObstacleMonitor` 按任务阶段自动开关隐形碰撞障碍以引导 RMPflow 运动；放置阶段持续监测料箱对齐并反应式修正末端目标，支持重试直至码放成功。

来源：cortex_tutorials/tutorial_cortex_5_ur10_bin_stacking.html
