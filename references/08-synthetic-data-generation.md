# 合成数据生成（6.0.1）

## 一、Replicator 感知数据生成

### 概览与基础工作流

Replicator 是基于 `omni.replicator` 的 SDG 框架，核心概念：**Annotator**（数据提取，需 prim 带语义标签）、**Writer**（输出，默认 `BasicWriter`）、**Randomizer**（随机化）、**Trigger**（触发）。提供 GUI 录制器、Replicator YAML（转为 OmniGraph 管线）、Python 脚本、Semantics Schema Editor 四类入口。
来源: replicator_tutorials/tutorial_replicator_overview.html

**Synthetic Data Recorder**（Tools > Replicator > Synthetic Data Recorder）：配置 Render Products（相机+分辨率）、annotator 勾选、输出目录（本地/S3）、帧数与 RTSubframes（减少动态场景伪影）；自定义 writer 须接受 `backend` 参数或 `**kwargs`；GUI 状态可存为 JSON。
来源: replicator_tutorials/tutorial_replicator_recorder.html

**Getting Started 脚本**：两种执行模式——Script Editor（async/await，`rep.orchestrator.step_async()`）与 Standalone（`SimulationApp` + 同步 `step()`）。核心模式：

```python
rep.orchestrator.set_capture_on_play(False)
writer = rep.writers.get('BasicWriter')
writer.initialize(backend=backend, rgb=True)
writer.attach(rp)
rep.orchestrator.step()
```

6.0 引入 `rep.functional.create/modify` 函数式 API、`rep.trigger.on_custom_event()` + `rep.utils.send_og_event()`、`wait_for_render=False` 与 write-to-fabric 性能优化。
来源: replicator_tutorials/tutorial_replicator_getting_started.html

### SDG Workflows（6.0 新增）

两个端到端完整范例：**Workflow 1** 物理沉降（持久场景，每次采集前重新掉落箱体由 PhysX 稳定）；**Workflow 2** 碰撞检测放置（按 `CAPTURES_PER_SCENE` 周期重建场景，`rep.functional.randomizer.scatter_2d(check_for_collisions=True)` 免物理堆叠，失败自动重试）。强调七项关键配置：Backend 体系（`rep.backends.get("DiskBackend")`）、关闭自动采集、DLSS 模式（`rtx/post/dlss/execMode=2` 抗鬼影）、种子随机（`rep.rng.ReplicatorRNG(seed=42)`，6.0 新 API）、`rp.hydra_texture.set_updates_enabled(False)` 在物理/随机化期间停渲染、`step(delta_time, rt_subframes, pause_timeline)` 参数、收尾 `rep.orchestrator.wait_until_complete()`。
来源: replicator_tutorials/tutorial_replicator_sdg_workflows.html

### 场景/物体/环境三类 SDG

- **Scene-based**：仓库场景，叉车+托盘+箱体（物理掉落 + scatter_2d）+ 三相机随机视角；BasicWriter+DiskBackend，支持 Kitti/Coco；运行：`./python.sh standalone_examples/replicator/scene_based_sdg/scene_based_sdg.py --config <json/yaml>`。来源: tutorial_replicator_scene_based_sdg.html
- **Object-based**：面向位姿估计/检测，物体在工作域内随机生成（部分悬浮、部分掉落）、随机速度、干扰物随机颜色尺度，PoseWriter 输出；`standalone_examples/replicator/object_based_sdg/object_based_sdg.py`。来源: tutorial_replicator_object_based_sdg.html
- **Environment-based（Infinigen）**：用 Infinigen 程序化生成室内场景（不同 seed 批量导出 USD），Isaac Sim 中加碰撞体、定义工作区，放置带标签 YCB 目标物+干扰物，悬浮/物理沉降两种采集；`standalone_examples/replicator/infinigen/infinigen_sdg.py`。来源: tutorial_replicator_infinigen_sdg.html

### 机器人任务随机化范例

- **AMR Navigation**：Nova Carter 经 OmniGraph 导航栈驶向随机 Xform 目标，随机化 dolly 位姿/灯光/堆叠道具；接近阈值时触发采集，LdrColor annotator 读双目相机；`standalone_examples/replicator/amr_navigation.py --use_temp_rp --num_frames --env_interval`。来源: tutorial_replicator_amr_navigation.html
- **UR10 Palletizing**：事件触发式采集——物理 overlap 查询监测 bin 接触翻转器/落到托盘两个时刻，暂停仿真、随机化灯光/材质/相机、采集后恢复状态继续；`rep.annotators.get()` 取 RGB+实例分割。来源: tutorial_replicator_ur10_palletizing.html

### Cosmos SDG

用 **CosmosWriter** 采集 Carter Nova 仓库导航的多模态数据（RGB/深度/分割/着色分割/边缘，按 clip 组织为 PNG+MP4），再经 **Cosmos Transfer**（Multi-ControlNet，vis/edge/depth/seg 分支权重 0–1）做 sim-to-real 照片级增强；`./python.sh standalone_examples/replicator/cosmos_writer_warehouse.py`。来源: tutorial_replicator_cosmos.html

### 增强、随机化片段与排错

- **Augmentation**：annotator 级（attach 前）与 writer 级两种；NumPy（CPU）/Warp（GPU 核，免数据搬运）；API：`rep.annotators.Augmentation.from_function()`、`rep.annotators.augment()`、`augment_compose()`、`writer.augment_annotator()`。来源: tutorial_replicator_augmentation.html
- **Randomization Snippets**：灯光、OmniPBR 纹理、链式随机（Fibonacci 球面相机分布）、物理填箱（临时围墙+施力沉降）、SimReady 资产 variant 选择；大量直接 USD API（`UsdGeom.Xformable`、`UsdShade.MaterialBindingAPI`、`UsdPhysics`）配合 `step_async()`。来源: tutorial_replicator_isaac_randomizers.html
- **Troubleshooting**：深度噪点→关抗锯齿；材质加载不及时/鬼影→提高 `rt_subframes`（≥2）；黑图→`--reset-user`；时间线停止时异步渲染丢帧→`--/exts/isaacsim.core.throttling/enable_async=false`；Windows standalone 首帧缺失→先 warmup `step()`。**迁移要点：`omni.replicator.character` / `omni.replicator.agent` 已更名为 `isaacsim.replicator.agent`**。来源: replicator_tutorials/troubleshooting.html

## 二、Action & Event 数据生成

### Actor SDG（IRA）

三扩展协作：**omni.metropolis.pipeline**（OMP，行为框架）、**isaacsim.replicator.agent**（IRA，仿真管理）、**isaacsim.anim.robot.core**（IAR，机器人动画）。零代码 YAML 驱动：UI 加载配置 → Set Up Simulation → Start Data Generation（默认输出 `~/IRA_output`）；Headless：`./python.sh tools/actor_sdg/actor_sdg.py -c <config>`。角色遵循 routine-trigger 循环（空闲时按权重随机行为，触发时执行反应序列）。**迁移：6.0 的 IRA 1.x 相对 5.1 的 0.x 为架构重写——USD 原生 schema、Pydantic v2 校验、行内行为定义、新增行为树支持。**

- 配置文件：根键 `isaacsim.replicator.agent` + 版本号，含 environment / character（groups: num、routines 如 wander/patrol/idle、triggers）/ robot / sensor（相机放置策略：瞄准目标、最大覆盖）/ replicator（IRABasicWriter、CosmosIRAWriter、CustomWriter；RGB/分割/包围盒/骨架 annotator）；`seed`、`simulation_duration` 等。
- Configuration Editor API（`isaacsim.replicator.agent.ui`）：`load/save_config_file()`、`get/update_config()`（点分路径如 `environment.base_stage_asset_path`）、`setup_simulation()`、`start_data_generation()`。
- CustomWriter 示例：`isaacsim.streaming.rtsp` 的 `RTSPStreamWriter`，每相机独立 writer/端口，`rtsp://localhost:8554/camera_01`，h264 硬编码或 raw。
- Animated Robot Controller（IAR）：动画回放替代实时物理，自带 Nova Carter/iw.hub/叉车配置，可扩展 YAML；架构：AnimRobot + StateMachine + Actions（`move_to/idle/turn/play_animation/sequence`）+ Drive（全向/差速）+ Behaviors（Wander/Patrol/Halt）。

来源: action_and_event_data_generation/tutorial_replicator_agent.html 及 ext_replicator-agent/ 各子页

### Behavior Tree Generation（6.0 新增）

LLM 将自然语言场景描述自动生成 actor 行为树。需要：场景上下文文件、节点目录、元数据 schema、模型配置、输出目录、**NVIDIA API key**。三个核心 API：`setup_workspace()`、`prepare_runtime()`、`generate_behavior_tree()`；支持 UI 示例窗口或纯 Python 脚本两种用法。来源: tutorial_behavior_tree_gen.html

### Object SDG（IRO）

`isaacsim.replicator.object`：面向无 3D 经验用户的零代码域随机化。YAML 描述文件三要素：**Mutables**（逐帧随机的对象+分布）、**Harmonizers**（多 mutable 协调约束）、**Settings**（物理/输出/帧数）。支持 UI、Docker/headless、内嵌快速预览；配套 **Chat IRO** 自然语言生成 YAML。输出 RGB、2D/3D 包围盒、分割，并记录帧状态可复现。来源: tutorial_replicator_object.html

### 其余扩展

- **VLM Scene Captioning**：从 3D ground truth 构建场景图后经 NVIDIA NIM LLM（默认 `meta/llama-3.1-8b-instruct`，可本地 NIM）生成图文对；支持 global/brief/qa 三类 caption、场景图剪枝（`pruning_ratio`）、`attach_label_to_usd` 自动打标。来源: tutorial_replicator_caption.html
- **Physical Space Event Generation（IRI）**：三类事件——箱体倾倒、火灾烟雾、液体泼洒；属性菜单标记物品（loose/flammable/spillable，支持 `$random_loose_item$`），YAML 定义触发（time/carb_event/physical_event）；输出事件语义标签 + `incidents_report.json`；Python API：`isaacsim.replicator.incident.core`。来源: tutorial_replicator_incident.html
- **RTX Sensors Placement & Calibration**：`isaacsim.sensors.rtx.placement`（按覆盖需求自动寻优相机位）+ `isaacsim.sensors.rtx.calibration`（导出位姿/FOV 标定元数据）。来源: tutorial_sensors_rtx_placement.html
- **Event Reactive Actors（6.0 新增）**：IRI 事件经 carb event bus 触发 IRA 角色反应，纯 YAML、无代码耦合——Actor 侧 `event_trigger.event` 字符串须匹配 `isaacsim.replicator.incident.core.events/<事件名>`；触发后按序执行行为列表再恢复 routine；目前仅 Fire/SpillEvent 派发 carb 事件（ToppleEvent 不行）。来源: example_event_reactive_actors.html
- **Omni Metropolis Pipeline（6.0 新增）**：Action/Event 生态的共享基础层——ConfigurationManager（单一 YAML 分发到各扩展的 async setup）、TriggersManager（time/carb-event/collision 三类内置触发）、AgentsManager（timeline 播放时发现并实例化 agent prim，统一 behavior+trigger 接口）。来源: tutorial_omni_metropolis_pipeline.html
- **Telemetry & Performance Tracking（6.0 新增）**：记录执行时间、资源占用、文件读写、资产加载等；`--/telemetry/mode=prod|test|dev`（headless 需显式 dev），日志在 `~/.nvidia-omniverse/logs/`；`--/telemetry/enableAnonymousData=false` 关上传。来源: tutorial_telemetry.html

## 三、专项 SDG

- **Grasping SDG**：自动生成抓取数据集——antipodal 采样候选抓取位姿 → 多阶段执行（pre-grasp/close/lift）→ 物理仿真评估 → 结果记入结构化 YAML（gripper 配置、关节状态、位姿、评估结果）。核心类 `GraspingManager`；UI（Tools > Replicator > Grasping）或 `./python.sh standalone_examples/api/isaacsim.replicator.grasping/grasping_workflow_sdg.py`；示例台 `sdg_grasping_xarm.usd`。来源: synthetic_data_generation/tutorial_replicator_grasping_sdg.html
- **MobilityGen**：移动机器人数据集工具，支持 Jetbot/Carter（差速）、Spot（四足）、H1（人形）；先生成占据栅格图（`~/MobilityGenData/maps/` YAML+PNG），录制模式含键盘 WASD/手柄遥操与 Random Acceleration / Random Path Following 自动模式；录制仅存轨迹与关节状态，再由 `replay_directory.py` 离线渲染 RGB/深度/分割/法线（需关多 GPU）。**6.0 迁移**：旧录制需用 `migrate_recordings.py` 转 `.npz` 格式。来源: tutorial_replicator_mobility_gen.html
- **Teleoperation SDG（6.0 新增）**：VR 遥操采集操纵演示用于模仿学习——CloudXR 头显（如 Quest 3）或无硬件 debug 模式（屏上拖拽标记）；支持单/双臂悬浮夹爪（xArm、Dex3）、IK 机械臂（UR3e）、移动底盘+机械臂；Episode Recorder 录 **HDF5**（链级世界位姿、手柄输入通道、OpenXR 头/瞄准位姿、相机轨迹与自定义 USD 属性），回放时经 Replicator writer 生成合成数据集。来源: tutorial_replicator_teleop_sdg.html

## 6.0 相对 5.1 的关键变化速览

1. 扩展改名：`omni.replicator.character`/`omni.replicator.agent` → `isaacsim.replicator.agent`；IRA 1.x 架构重写（USD schema、Pydantic v2、行为树）。
2. 新增教程/能力：SDG Workflows 端到端范例、Behavior Tree Generation（LLM 生成行为树）、Event Reactive Actors（事件↔角色联动）、Omni Metropolis Pipeline（统一配置/触发/agent 基础层）、Telemetry、Teleoperation SDG（VR+HDF5）。
3. 新 API 模式：`rep.functional.*` 函数式接口、`rep.backends`（DiskBackend）、`rep.rng.ReplicatorRNG` 种子随机、`hydra_texture.set_updates_enabled()` 渲染门控。
