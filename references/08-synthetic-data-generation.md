# 合成数据生成（SDG）

## SDG 总览与 Replicator 定位

Isaac Sim 5.1 的 SDG 分四大块：感知数据生成（Replicator）、行为与事件数据生成、抓取 SDG、MobilityGen。Replicator 基于 `omni.replicator.core`（`rep`），核心概念：**Randomizer**（对材质/位姿/光照等做程序化随机）、**Annotator**（提取 RGB、语义/实例分割、2D/3D 包围盒、光流等标注）、**Writer**（BasicWriter、KittiWriter、CocoWriter、PoseWriter 等落盘格式）、**Trigger**（`on_frame`/`on_custom_event` 控制触发时机）、**Orchestrator**（调度整个采集流程）。三种工作模式：GUI（Synthetic Data Recorder）、Python 脚本/Script Editor、YAML 配置驱动（转为 OmniGraph）。注意 5.1.0 已停止支持。

来源：synthetic_data_generation/index.html、replicator_tutorials/index.html、replicator_tutorials/tutorial_replicator_overview.html

## Synthetic Data Recorder（GUI 工具）

菜单 Tools > Replicator > Synthetic Data Recorder。Writer 面板配置 Render Products（相机路径+分辨率）、Writer 参数（默认 BasicWriter，自定义 writer 用 JSON 配置）、输出目录（支持 S3）、可保存/加载 GUI 配置。Control 面板：Start/Stop/Pause、帧数（0=无限）、RTSubframes（弱光/快速运动时增大以减少伪影）、Control Timeline、Verbose。内部用 `orchestrator.step()` 保证帧同步；资产需有语义标签。

来源：replicator_tutorials/tutorial_replicator_recorder.html

## 核心 API 入门套路

典型流程：`rep.create.render_product()`/`rep.create.camera()`/`rep.create.light()` → `rep.trigger.on_frame()` 或 `on_custom_event()`（配合 `rep.utils.send_og_event()` 手动触发）→ `rep.WriterRegistry.get("BasicWriter")` + `writer.initialize()` + `writer.attach(rp)` → `rep.orchestrator.set_capture_on_play(False)`，循环 `rep.orchestrator.step(rt_subframes=N)`，结束 `wait_until_complete()`。Script Editor 用异步 `step_async()`，standalone 用同步 `step()`。低分辨率下建议 DLSS 设 Quality 模式（`rtx/post/dlss/execMode=2`）。

来源：replicator_tutorials/tutorial_replicator_getting_started.html

## 场景级 / 物体级 / 环境级（Infinigen）SDG

- **场景级**：仓库+叉车+托盘场景。先 `SimulationApp` 再 import，`open_stage()` 加载环境，`prims.create_prim()` 生成带语义标签资产；`rep.randomizer.register()` 注册自定义随机器（`scatter_2d(check_for_collisions=True)` 撒箱、`rep.randomizer.materials()`、灯光/相机随机），先物理落箱再循环 `orchestrator.step()`。支持 Basic/Kitti/Coco writer，运行 `./python.sh standalone_examples/replicator/scene_based_sdg/scene_based_sdg.py --config xxx.yaml`。
- **物体级**：封闭工作区+隐形碰撞墙，训练资产+形状/网格干扰物悬浮碰撞；混合三类随机化——物理（速度、拉回中心）、USD API 直接改属性（`set_transform_attributes`、球面采样相机位姿）、Replicator（`rep.create.group()`+`rep.randomizer.color()`、dome 背景）；常配 PoseWriter。
- **Infinigen 环境级**：用 Infinigen 按 seed 批量生成程序化室内场景，`--omniverse` 导出 USD；SDG 脚本循环加载各环境、加碰撞体、生成标注资产与干扰物，先拍悬浮帧、物理沉降后再拍，配置全由 YAML 驱动。

来源：replicator_tutorials/tutorial_replicator_scene_based_sdg.html、tutorial_replicator_object_based_sdg.html、tutorial_replicator_infinigen_sdg.html

## 域随机化实战：AMR 导航与 UR10 码垛

- **AMR 导航**：Nova Carter 双目相机采 LdrColor（1024×1024），机器人接近目标（dolly）时基于距离回调触发采集；随机化目标位姿、每 N 帧切换环境、球灯颜色随机。可用临时 render product（`rp.hydra_texture.set_updates_enabled()`）省显存。`./python.sh standalone_examples/replicator/amr_navigation.py --use_temp_rp`。
- **UR10 码垛**：`PalletizingSDGDemo` 监听 timeline 事件，用 `get_physx_scene_query_interface().overlap_box()` 检测料箱碰撞事件→暂停仿真→跑 SDG→恢复，保证"SDG 不改变仿真结果"（材质用 `UsdShade.MaterialBindingAPI` 缓存后还原）。展示两种取数方式：annotator `get_data()` 手动存图 vs BasicWriter 自动写盘；`rep.trigger.on_frame(interval=4)` 控制不同随机化频率。

来源：replicator_tutorials/tutorial_replicator_amr_navigation.html、tutorial_replicator_ur10_palletizing.html

## Cosmos 视频生成

用 **CosmosWriter** + `DiskBackend` 为 Cosmos Transfer（Multi-ControlNet）录制五路同步模态：RGB(vis)、深度、分割、着色分割、Canny 边缘（`canny_threshold_low/high` 可调，`use_instance_id`/`segmentation_mapping` 控制分割）。循环 `rep.orchestrator.step(pause_timeline=False)` 采帧，`cosmos_writer.next_clip()` 分片，输出 clip 目录+每模态 mp4，直接作为 Cosmos 的控制输入。

来源：replicator_tutorials/tutorial_replicator_cosmos.html

## 数据增强

支持 NumPy(CPU) 与 Warp(GPU) 自定义增强：`rep.annotators.Augmentation.from_function()` 创建、`rep.AnnotatorRegistry.register_augmentation()` 注册、`rep.annotators.augment()`/`annotator.augment()` 包装 annotator、`augment_compose()` 串联、`writer.augment_annotator()` 用于 writer 侧。示例：RGB 红蓝通道互换、深度加不同 sigma 高斯噪声。

来源：replicator_tutorials/tutorial_replicator_augmentation.html

## 自定义随机化片段（Isaac Randomizers）

当内置 randomizer 不够用时的写法集合：直接用 USD `GetAttribute().Set()` 随机灯光；OmniPBR 材质/纹理随机（`UsdShade.MaterialBindingAPI`）；链式依赖随机（先随机托盘再稳定摆放料箱、球面采样相机）；物理填充体积（刚体+碰撞墙+临时改摩擦力）；SimReady 资产变体（`GetVariantSets()`）。

来源：replicator_tutorials/tutorial_replicator_isaac_randomizers.html

## 在线生成与位姿估计训练

- **在线生成**（已弃用）：数据不落盘直接喂 PyTorch，把生成包进 `IterableDataset`，`__next__` 中 `orchestrator.step()` → `rgb.get_data(device="cuda")`/`bbox_2d_tight`/`instance_seg` 取数 → 组 batch，训练 Mask-RCNN（ShapeNet 资产）。
- **位姿估计**（已弃用）：生成 MESH（大量飞行干扰物+碰撞盒）与 DOME（dome 光照背景）两类数据集，YCB 目标物；writer 支持 **DOPE / CenterPose / YCBVideo(PoseCNN)**；关键 API：`get_world_pose_from_relative()`、`apply_body_force()`。运行 `pose_generation.py --num_mesh 30 --num_dome 30 --writer DOPE`。

来源：replicator_tutorials/tutorial_replicator_online_generation.html、tutorial_replicator_pose_estimation.html

## Agent SDG（人/机器人行为数据）

`isaacsim.replicator.agent`（IRA，旧名 omni.replicator.character/agent）模拟人物与 Nova Carter/iw.hub 机器人，服务人员检测/跟踪训练。YAML 配置含 global（seed、时长）、scene、sensor（相机数与摆放）、character/robot（数量、资产、命令、出生区）、response/incident、replicator(writer)。流程：启用扩展→载配置→Set Up Simulation→生成/编辑命令→采数。无头运行：`./python.sh tools/actor_sdg/sdg_scheduler.py -c config.yaml`。输出多相机同步 RGB+2D/3D 框+姿态。

来源：action_and_event_data_generation/index.html、tutorial_replicator_agent.html

## Object SDG（IRO）

`isaacsim.replicator.object`：零代码 YAML 描述随机化场景。核心概念 **Mutables**（逐帧随机的物体/灯光/相机）、**Harmonizers**（约束多个 mutable 联动）、可随机属性分布（range/folder 等）。逐帧流程：解析符号→更新 USD→物理仿真→采图→存状态（支持场景复原重采样）。UI 或 Docker（`--/config/file=`）运行；输出 RGB(JPG)、分割 PNG、2D/3D 框、元数据。

来源：action_and_event_data_generation/tutorial_replicator_object.html

## VLM 场景描述与事件生成

- **VLM Captioning**：基于 3D 真值构建场景图（节点=物体、边=空间关系），经 NVIDIA NIM 的 LLM（默认 llama3-8b/70b-instruct，可本地）生成图文对；YAML 配置相机、caption 风格（brief/full、全局/QA）、场景图裁剪；输出 JSON 场景图、caption、可视化叠加、点云。
- **事件生成（IRI）**：三类自发事件——物体倾倒、起火、液体泼洒；流程：Event Scene Tagger 标记 loose/flammable/spillable → YAML 定义事件（target、time 触发、参数）→ Record Events；输出语义标签（如 `incident_toppled_item`）、事件日志 YAML、SDG 数据，可与 IRA 联动生成人物响应。

来源：action_and_event_data_generation/tutorial_replicator_caption.html、tutorial_replicator_incident.html

## RTX 传感器摆放标定

`isaacsim.sensors.rtx.placement`：按覆盖率需求与场景约束自动优化相机位姿；标定功能导出位姿、朝向、FOV 多边形等元数据为 JSON，并可视化每台相机覆盖范围。适合仓库/零售多相机部署规划。

来源：action_and_event_data_generation/tutorial_sensors_rtx_placement.html

## 抓取 SDG 与 MobilityGen

- **抓取 SDG**：`GraspingManager` API + UI（Tools > Replicator > Grasping）。配置夹爪 USD/关节/预抓取位与多阶段抓取相位；**antipodal 采样器**生成候选抓取位姿（旋转数、standoff、开口、扰动、seed 可配）；对每个候选按相位跑物理仿真并记录成败指标，YAML 保存配置可批处理。
- **MobilityGen**：移动机器人数据采集工具，支持差速（Jetbot/Carter）、四足（Spot）、人形（H1）。流程：Occupancy Map 扩展生成占据图（PNG+YAML）→ UI 录制轨迹（键盘/手柄遥操或随机加速/随机路径）→ `replay_directory.py --render_interval N` 离线回放渲染 RGB、深度、语义/实例分割、法线。

来源：synthetic_data_generation/tutorial_replicator_grasping_sdg.html、tutorial_replicator_mobility_gen.html

## 排障要点

深度噪声→关抗锯齿；材质加载不及时/残影→`rt_subframes≥2` 并按需增大；黑图→`--reset-user` 重启；掉帧→启动加 `--/exts/isaacsim.core.throttling/enable_async=false`；尽量避免 `rep.new_layer()`；Windows 写 S3 用环境变量存凭证；旧扩展名统一迁移到 `isaacsim.replicator.agent`。

来源：replicator_tutorials/troubleshooting.html
