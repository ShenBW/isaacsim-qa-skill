# 数字孪生与 Isaac Cortex（6.0.1）

## 1. 仓储物流工具集

**Warehouse Creator**（`omni.warehouse.creator.ui` + 无头 API 包 `omni.warehouse.creator.api`）：将 2D 网格平面图转换为 USD 仓库场景，基于 Modular Warehouse 资产包。提供俯视 2D 网格编辑器（自由绘制/直线/框选填充、对称、布尔运算合并/减除/翻转/旋转）、单元分组（每组生成独立 root prim）、生成后可切换墙面/中心瓦片变体（装卸码头、窗户等）并可开关立柱。工作流：块库选资产 → 网格上绘制 → 分组 → Generate Warehouse。**相对 5.1：这一 2D 草图式生成器为 6.0 重做的新版工具（5.1 为旧式参数化 Warehouse Creator）。**
来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/digital_twin/warehouse_logistics/ext_omni_warehouse_creator.html

**传送带**（`isaacsim.asset.gen.conveyor.ui`）：将刚体一键转为传送带（Create > Isaac Sim > Warehouse Items > Conveyor），生成 OmniGraph 节点控制速度方向/大小、贴图动画，Curved 模式下角速度沿 Direction 向量（旋转轴）施加。附 Conveyor Track Builder，内置 50+ 预制轨道件（直段/弯道/分流/坡道），支持 JSON 自定义件库。另有 `standalone_examples/conveyor_belt` 基于 Warp 直接计算摩擦力的替代方案，适合多带接触场景。与 5.1 基本一致。
来源: digital_twin/warehouse_logistics/ext_isaacsim_asset_gen_conveyor.html

**静态仓库资产**：Window > Browsers > NVIDIA Assets → Industrial > Buildings > Warehouse，拖拽即以引用方式加载。注意多数资产为厘米单位，需将缩放设为 0.01（可在父 prim 上批量设置）；资产仅视觉，物理属性（刚体/碰撞体）须自行添加，或改用带物理与语义标注的 SimReady 资产。推荐通过 USD 引用 + 本地增量做定制。
来源: digital_twin/warehouse_logistics/tutorial_static_assets.html

**cuOpt 路径优化**：通过 `omni.cuopt.service`（封装服务、格式化请求）与 `omni.cuopt.visualization`（路点图、语义区域可视化）集成，需自行部署 cuOpt 服务器并启用 `omni.cuopt.examples`。四个示例：创建路点网络、简单代价矩阵求解、JSON 加权路点图、含高代价语义区域的仓内运输。
来源: digital_twin/warehouse_logistics/logistics_tutorial_cuopt.html

## 2. RTSP 相机流（6.0 新增）

扩展 `isaacsim.streaming.rtsp`：将场景相机以 RTSP 直播，内置进程内 RTSP 服务器，**无需外部 mediamtx**；用 NVENC 硬编 H.264，VLC/ffplay/GStreamer/OpenCV 均可拉流。OmniGraph 用法：`OnPlaybackTick → IsaacCreateRenderProduct → RTSPCameraHelper`，配置相机 prim、端口（默认 8554）与挂载路径（默认 `/stream`），Play 后访问 `rtsp://localhost:8554/stream`；也可用 og.Controller 以 Python 搭建，或经 `RTSPStreamWriter` 走 Replicator 流程。编码：H.264（默认，支持逐帧 SEI 仿真时间戳）/ Raw RGBA（`useRawEncoding=true`，不支持 SEI）。限制：每路流需独立端口；首帧渲染后服务器才启动；编码失败进入 failed 状态需重启时间线。**此功能为 6.0 新增，5.1 无对应能力。**
来源: digital_twin/rtsp_camera_streaming.html

## 3. 占用图（Mapping）

扩展 `isaacsim.asset.gen.omap`（默认启用，与 5.1 相同）：基于物理碰撞几何，在指定高度切片生成 2D 二值占用图。参数：Origin（必须位于空闲区）、上下边界、Cell Size（米/像素）；提供 CENTER TO SELECTION / BOUND SELECTION 快捷定位，可切换 PhysX 碰撞近似或原始三角网格。CALCULATE 计算后 VISUALIZE IMAGE 预览，可自定义占用/空闲/未知配色、旋转与坐标系，导出 PNG/YAML（可直接用于 ROS Nav2）。支持 Python/standalone 编程调用。排障：场景需有碰撞几何，漏区检查 raycast。
来源: digital_twin/ext_isaacsim_asset_generator_occupancy_map.html
排障页: digital_twin/troubleshooting.html（覆盖 Warehouse Creator 缺件/贴图、传送带穿透与速度、Cortex 资产加载、占用图缺区等）

## 4. Cortex 框架与两个案例

**重要变化：Cortex 自 Isaac Sim 6.0.0 起已弃用，将在后续版本移除**（5.1 中仍为正常功能），官方建议迁移至 py_trees（行为树）或 transitions（状态机）。

**框架概述**：围绕"信念世界"（belief world）的 60Hz 六阶段流水线：感知 → USD 世界建模 → 逻辑状态监控 → 决策（Decider Network）→ 命令 API → 底层控制；信念仿真与真实世界分离，可先全仿真验证再接实机。核心概念：决策器网络是有向无环图，根节点每周期经 `decide()` 返回 `DfDecision` 逐层下溯到叶节点；Monitor 每周期更新逻辑状态；上下文继承 `DfLogicalState` 持有命令 API。关键 API：`DfNetwork`、`DfDecider`（enter/decide/exit）、`DfStateMachineDecider`、`DfStateSequence`。
来源: cortex_tutorials/tutorial_cortex_1_overview.html、tutorial_cortex_2_decider_networks.html

**案例一：Franka 堆方块**：按序堆四色方块成塔，人为挪动方块时可反应式重规划。顶层 `BlockPickAndPlaceDispatch` 按"塔是否完成/夹爪是否持块"分派 pick / place / go home；pick 与 place 各为 RLDS 优先级序列（`DfRldsDecider`，鲁棒逻辑动力系统），原子抓放动作用锁定状态机防中断；`BuildTowerContext` 监控感知、塔构型、夹爪与碰撞抑制。运行：`isaac_python franka_examples_main.py --behavior=block_stacking_behavior`。
来源: cortex_tutorials/tutorial_cortex_4_franka_block_stacking.html

**案例二：UR10 吸盘码垛**：UR10 + 吸盘将传送带上的料箱码上托盘，需翻面的先过翻转站。顶层 Dispatch 分派 flip / pick / place / go home；障碍监视器按任务阶段自动开关隐形碰撞屏障，借 RMPflow 反应性整形运动；放置时末端动态纠偏，放置成功率约 100%。运行：`isaac_python demo_ur10_conveyor_main.py`。
来源: cortex_tutorials/tutorial_cortex_5_ur10_bin_stacking.html
