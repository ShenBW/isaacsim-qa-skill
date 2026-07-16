# 概述、安装与 6.0 迁移指南

## Isaac Sim 6.0.1 是什么与版本要点

Isaac Sim 是 NVIDIA 基于 USD/Omniverse 的机器人仿真器：可导入 URDF、MJCF、Onshape CAD、USD；用 PhysX 或 Newton 物理引擎仿真；支持 RTX 与物理传感器、Replicator 合成数据、Isaac Lab 训练准备、ROS 2 验证。6.0.1 GA 将 Kit SDK 从 110.1.1 升到 110.1.2（6.0.0 为 110.0.0→110.1.1）。
来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/index.html

**相对 5.x 的重大变化**（overview/release_notes.html）：

- **Newton 物理引擎**（实验性）与引擎无关的 `isaacsim.core.experimental` API；URDF/MJCF 导入器改用 `NewtonArticulationRootAPI`、`NewtonMimicAPI`，不再写 `PhysxArticulationAPI`/`PhysxMimicJointAPI`（破坏性变更）
- **传感器架构重构**：`isaacsim.sensors.camera`/`rtx`/`physics`/`physx` 全部弃用，统一到 `isaacsim.sensors.experimental.rtx` 与 `isaacsim.sensors.experimental.physics`；新增 RTX Acoustic 超声传感器；多 tick 渲染让相机/RTX Lidar 按物理时间调度
- **SDG 扩展**：Replicator Functional API、Actor Simulation（`isaacsim.replicator.agent` 1.x，行为树控制器）、`isaacsim.replicator.object`/`incident`/`teleop`/`episode_recorder`（HDF5）、`omni.ai.behavior_tree_gen`
- **ROS 2**：Windows 通过 Pixi 全面支持；张量后端节点支持任意物理引擎；新增 `OgnROS2RtxRadarHelper`、仿真控制服务 `GetEntityBounds`/`SpawnEntities`/`GetSpawnables`
- **其他**：PINK 逆运动学（`isaacsim.robot_motion.pink`）、SimReady 内容浏览器、FANUC/Comau 资产、Docker Compose + WebRTC 网页查看器、RTSP 流（`isaacsim.streaming.rtsp`）、DGX Spark 支持
- **主要弃用**：`isaacsim.core.api/prims/utils`→`core.experimental`；`robot.manipulators`/`wheeled_robots`/`robot_motion.lula`→experimental 对应包；Cortex 系列（6.1 给迁移指引）；`omni.isaac.ml_archive`→自行装 PyTorch
- **移除**：所有 `omni.isaac.*` 兼容垫片、`isaacsim.asset.browser`（→`omni.simready.content.browser`）、`isaacsim.app.selector`

## 6.0 迁移指南详解

索引: migration_guides/isaac_sim_6_0/index.html

### Camera Sensors（sensors_camera_to_experimental_rtx.html）

`isaacsim.sensors.camera`→`isaacsim.sensors.experimental.rtx`。`Camera`→`CameraSensor`+`RtxCamera`（运行时/作者分离）；`CameraView`→`TiledCameraSensor`（正则改显式路径列表）；`SingleViewDepthSensor`→`SingleViewDepthCameraSensor`。参数复数化：`position=`→`positions=[[...]]`（N×3）、`orientation=`→`orientations=[[...]]`（N×4）；`frequency`/`dt`→`tick_rate`（Hz）；`name` 移除；畸变经 `schemas=`/`attributes=` 字典。数据访问：`add_rgb_to_frame()+get_current_frame()`→构造时 `annotators=["rgb"]`，`data, info = sensor.get_data("rgb")`；不再需要 `.initialize()`。

### RTX Sensors（sensors_rtx_to_experimental_rtx.html）

`LidarRtx`→`Lidar.create()`+`LidarSensor()`；雷达为 `Radar.create()`+`RadarSensor()`；超声更名 Acoustic（`Acoustic.create()`+`AcousticSensor()`，写 `OmniAcoustic` prim + `OmniSensorGenericAcousticWpmAPI`）。不再用 `omni.kit.commands.execute("IsaacSensorCreateRtxLidar")`。方法对照：`get_current_frame()`→`get_data("generic-model-output")`（返回 `(wp.array, dict)`）；`attach_annotator()`→构造参数 `annotators=[...]`；`enable/disable_visualization()`→`attach/detach_writer("draw-point-cloud")`（由 `isaacsim.sensors.rtx.nodes` 注册）；`initialize/pause/resume` 改由 `omni.timeline` 驱动；`get_gmo_data(rawPtr)`→`parse_generic_model_output_data(data)`；`decode_stable_id_mapping()`→`parse_stable_id_map_data()`。Radar 需开 `/renderer/raytracingMotion/enabled=True`。RTX IDS 无替代。config 参数：`config_file_name=`→`config=`；目前每传感器仅支持 N=1。

### Physics Sensors（sensors_physics_to_experimental_physics.html）

`isaacsim.sensors.physics`（及 physx）→`isaacsim.sensors.experimental.physics`。五类：`IMU/IMUSensor`、`Contact/ContactSensor`、`Raycast/RaycastSensor`、`Effort/EffortSensor`、`JointState/JointStateSensor`。命令式创建（`IsaacSensorCreateImuSensor`）→`IMUSensor(IMU.create(...))`。`translation=Gf.Vec3d`→`translations=np.array([[...]])`；`positions`（世界系）与 `translations`（局部系）互斥，同传报 `ValueError`。3.0.0 移除 `ImuSensorBackend` 等后端类与 `get_current_frame()`；用 `get_data()`（结构化 dict）/`get_sensor_reading()`（原始 struct）；阈值设置移到作者对象如 `sensor.contact.set_min_threshold(v)`。

### PhysX Lidar→Physics Raycast（sensors_physx_lidar_to_physics_raycast.html）

`from isaacsim.sensors.physx import Lidar`→`from isaacsim.sensors.experimental.physics import RaycastSensor`。`rotationRate`→`rayTimeOffsets`（周期=1/rotation_rate 内分布；rotationRate=0 则省略）；水平/垂直分辨率→显式 `rayDirections`；`minRange`/`maxRange` 语义不变；`drawLines`→Debug Draw RayCast OmniGraph 节点；`_range_sensor` 接口→`get_sensor_reading()`（深度、命中点、法线）；`enable_semantics`→`reportHitPrimPaths` 属性。

### PhysX Generic→Physics Raycast（sensors_physx_generic_to_physics_raycast.html）

`sensor_pattern`（方位/天顶角）→`rayDirections` 笛卡尔向量（`dx=cos(zenith)*cos(azimuth)` 等）；`origin_offsets`→`rayOrigins`；`send_next_batch()`/`set_next_batch_rays()` 不再需要，创建时定义全部射线，`rayTimeOffsets` 自动分批。

### PhysX Lightbeam→Physics Raycast（sensors_physx_lightbeam_to_physics_raycast.html）

配置为"光幕"：`numRays`→`rayOrigins`/`rayDirections` 数组长度；`curtainLength`+`curtainAxis`→沿轴分布的 `rayOrigins`；`forwardAxis`→统一 `rayDirections`。示例：`isaacsim.sensors.physics.examples/raycast_sensor.py`；GUI：Robotics Examples > Sensors > Physics Raycast Sensor。

### ROS 2 OmniGraph 节点（ros2_omnigraph_migration.html）

从"隐式 prim 解析"转向"显式张量 API"：

- **ROS2 Publish Transform Tree**：不再直连 prim，前置 **Isaac Compute Transform Tree**（`isaacsim.core.nodes`）节点，设 `targetPrims`（关节树自动展开）、`parentPrim`；连接 `execOut/parentFrames/childFrames/translations/orientations` 到发布节点同名输入。
- **ROS2 Publish Joint State**：前置 **Isaac Read Joint State**（`isaacsim.sensors.physics.nodes`），设 `prim` 为 articulation root；连接 `jointNames/jointPositions/jointVelocities/jointEfforts/jointDofTypes/stageMetersPerUnit/sensorTime`。
- 另：**ROS2 Camera Helper / Camera Info Helper 的 `frameSkipCount` 输入弃用**，改在相机 prim 上设 `omni:sensor:tickRate`（多 tick 渲染）。

### Replicator Agent 0.x→1.x（ext_isaacsim_replicator_agent_migration_guide.html）

配置架构重写：`global.seed`→根级 `seed`；`simulation_length`（帧）→`simulation_duration`（秒，=帧/30）；`scene.asset_path`→必填 `environment.base_stage_asset_path`（支持 URL/相对路径），新增 `prop_asset_paths`；角色/机器人/相机改**命名 groups**；外部命令 `.txt` 文件→内联 YAML `routines`（wander/patrol/stop/halt）+ `triggers`（event/time/collision）；动画后端 `omni.anim.people`→`omni.anim.behavior.core`；相机布置策略 `aim_at_targets`/`maximum_coverage`（`num:-1` 自动计算）；`replicator.writer+parameters`→`writers` 字典，支持多写入器、`start_frame`/`start_time`、`sensor_prim_list`，新增 `CosmosIRAWriter`/`SceneGraphWriter`。入口脚本：`tools/actor_sdg/sdg_scheduler.py`→`tools/actor_sdg/actor_sdg.py`。USD schema：`IRACharacterAPI`、`AnimRobotAPI`；1.3.0+ 支持 `behavior_tree` JSON。

### MobilityGen 录制（mobility_gen_recordings_migration.html）

状态存储由 pickle 的 `state/common/*.npy` 改为 NumPy 命名数组 `*.npz`；未转换的录制回放会"报错并产生 0 步"。转换命令：
`./python.sh standalone_examples/replicator/mobility_gen/migrate_recordings.py ~/MobilityGenData/recordings --recursive`（原地转换）。

### Surface Gripper 绑定（surface_gripper_bindings_import.html）

编译模块移入 `bindings` 子包，仅导入路径变化：`import isaacsim.robot.surface_gripper._surface_gripper`→`from isaacsim.robot.surface_gripper import _surface_gripper`；直接导入符号用 `from isaacsim.robot.surface_gripper.bindings._surface_gripper import ...`。`acquire_surface_gripper_interface()`、`GripperStatus` 等 API 不变。

## 系统需求（installation/requirements.html）

- OS：Ubuntu 22.04/24.04、Windows 11（**不再支持 Win10**）、DGX OS 7（仅 DGX Spark aarch64）
- GPU：最低 RTX 4080 16GB / 良好 RTX 5080 / 理想 RTX PRO 6000 Blackwell 48GB；**无 RT Core 的 A100/H100 不支持**
- 驱动：Linux 595.58.03+，Windows 595.97+
- CPU 4/8/16 核；内存 32GB 起（推荐 64GB）；存储 50GB SSD 起，推荐 500GB–1TB NVMe；容器仅 Linux；DGX Spark 不支持 cuRobo/cuMotion

## 各安装方式关键步骤

**Quick Install/工作站**（quick-install.html、install_workstation.html）：下载 `isaac-sim-standalone-6.0.1-linux-x86_64.zip`（或 aarch64/windows），解压到 `~/isaacsim` 或 `C:\isaacsim`，运行 `./post_install.sh`（建 extension_examples 软链），`./isaac-sim.sh` / `isaac-sim.bat` 启动；首启 5–10 分钟预热 shader 缓存；`--reset-user` 重置配置；`clear_caches.sh` 清缓存。资产完整包分 5 卷 `isaac-sim-assets-complete-6.0.1.001~005.zip`（download.html）。

**容器**（install_container.html）：`docker pull nvcr.io/nvidia/isaac-sim:6.0.1`；需 Docker + NVIDIA Container Toolkit；预建 `~/docker/isaac-sim/{cache/main,cache/computecache,config,data,logs,pkg}` 并 `chown 1234:1234`；`docker run --gpus all -e "ACCEPT_EULA=Y" -e "PRIVACY_CONSENT=Y" --network=host -u 1234:1234 ...`；容器内 `./runheadless.sh -v` 无头流式；端口 TCP 49100/UDP 47998/TCP 8210；Docker Compose：`docker compose -p isim -f tools/docker/docker-compose.yml up --build -d`。

## Python 与 ROS 2 环境

**pip**（install_python.html）：需 **Python 3.12**、Linux GLIBC 2.35+。先装 PyTorch：`pip install torch==2.11.0 --index-url https://download.pytorch.org/whl/cu128`（或 cu130）；再 `pip install "isaacsim[all,extscache]==6.0.1.0" --extra-index-url https://pypi.nvidia.com`。启动：`isaacsim`；无头流：`isaacsim isaacsim.exp.full.streaming --no-window`。须设 `OMNI_KIT_ACCEPT_EULA=YES`；Windows 需开启长路径；Notebook 中 `SimulationApp.close()` 会中断 kernel。

**ROS 2**（install_ros.html）：支持 Jazzy（Ubuntu 24.04 推荐）与 Humble（22.04）；Windows 11 仅 Humble（Pixi 工作流）。可用系统 ROS、内置库或 Docker；内置 Jazzy 库在未 source 其他 ROS 时自动加载。关键变量：`ROS_DISTRO`、`RMW_IMPLEMENTATION`（`rmw_fastrtps_cpp` 默认，可选 cyclonedds/zenoh）、`LD_LIBRARY_PATH=$isaac_sim_path/exts/isaacsim.ros2.core/jazzy/lib`。工作区：github.com/isaac-sim/IsaacSim-ros_workspaces，`rosdep install -i --from-path src --rosdistro jazzy -y && colcon build && source install/local_setup.bash`。多机通信配 `fastdds.xml` + `FASTRTPS_DEFAULT_PROFILES_FILE`。

## FAQ 与排障要点

来源: installation/install_faq.html、overview/troubleshooting.html、overview/known_issues.html、overview/faq_index.html

- **路径**：日志 `~/.nvidia-omniverse/logs/Kit/Isaac-Sim`；shader 缓存 `~/.cache/ov/Kit`；配置 `~/.local/share/ov/data/Kit/Isaac-Sim`（容器内前缀 `/isaac-sim` 或 `/root`）
- **资产根优先级**：`ISAACSIM_ASSET_ROOT` 环境变量 > `--/persistent/isaac/asset_root/default=path` > `.kit` 文件 > 云端默认；离线用完整资产包
- **动画卡顿**（固定步长默认开启）：`--/app/player/useFixedTimeStepping=false --/app/runLoops/main/manualModeEnabled=false --/exts/isaacsim.core.throttling/enable_manualmode=false`
- **已知问题**：Windows 黑屏加 `--vulkan`；多 GPU RTX Lidar 可致 CUDA error 700，加 `--/renderer/multiGpu/enabled=false`；SDG 用 DLSS Quality 模式防低分辨率伪影；SDG 跳帧加 `--/exts/isaacsim.core.throttling/enable_async=false`、`rt_subframes>=2`；Windows 缺 MSVC v143 build tools 导致 torch 加载失败；Surface gripper 不能抓含 articulation root 的对象；Nav2 多机器人低于 25 FPS 可能相撞
- **其他**：降日志 `--/log/level=error`；性能用 Tracy 分析；GPU 超频软件（MSI Afterburner）可致 "int divide by zero"；tkinter 不受支持；Nucleus/Launcher 自 2025-10-01 起弃用
