# 机器人仿真、传感器与物理

基础 URL：`https://docs.isaacsim.omniverse.nvidia.com/5.1.0/`（下文来源为相对路径）

## 一、机器人仿真

### 1. Articulation 控制体系

Isaac Sim 将机器人控制分三层：关节级（Articulation Controller）、轮式机器人控制器、机械臂运动生成，以及更高抽象的 RL 策略控制。

- 关节级控制支持 position / velocity / effort 三种模式，角度单位为弧度（自动从 USD 的度转换）。
- 核心类：`ArticulationController`（需在仿真启动后基于 Articulation view 创建）与 `SingleArticulation`（高层封装，隐式创建控制器，`initialize()` 须在仿真开始后调用）。
- 指令通过 `ArticulationAction` 下发，字段：`joint_positions` / `joint_velocities` / `joint_efforts`，配合 `joint_indices` 或 `joint_names` 指定关节。**约束：同一关节同一时刻只能被一种控制方式驱动**。
- 提供 OmniGraph 节点接口：输入 `execIn`、targetPrim/robotPath、关节标识与指令数组；关节留空则控制全部关节，数组长度须匹配。

来源：robot_simulation/index.html、robot_simulation/articulation_controller.html

### 2. 移动机器人控制器

三类通用控制器，均有 Python API 与 OmniGraph 节点：

- **DifferentialController（差速）**：适用两轮机器人（如 Nova Carter），用左右轮速差实现线速度/角速度；参数：wheel radius、wheel distance、最大加减速、最大速度。
- **Holonomic Controller（全向）**：适用麦克纳姆轮机器人（如 Kaya），将问题建模为二次规划、最小化质心残余力，由前向/横向/偏航速度指令计算各轮驱动；参数：轮半径数组、轮位姿、mecanum 角、速度增益。
- **Ackermann Controller**：适用转向轮机器人（如 Leatherback），按轴距几何计算转向角与轮速；参数：wheel base、track width、轮半径、转向角限制、加速度。

来源：robot_simulation/mobile_robot_controllers.html

### 3. 运动生成栈（Lula / cuRobo）

Lula 是 CPU 端高性能运动生成库，包含四类工具；cuRobo 为 GPU 加速替代方案。选型：

| 需求 | 推荐 |
|---|---|
| 实时反应式避障（局部策略） | RMPflow |
| 静态环境全局路径规划 | RRT（RRT-Connect / JT-RRT） |
| 时间最优轨迹 | Lula Trajectory Generator |
| GPU 加速、批量无碰撞 IK / 规划 | cuRobo |

**运动学求解器**（三层架构）：

- `KinematicsSolver` 接口：任意 frame 的 FK/IK，内部维护独立于 USD 的机器人表示；`get_joint_names()`、`get_all_frame_names()`、`set_robot_base_pose()`（世界系换算）、`supports_collision_avoidance()`。
- `ArticulationKinematicsSolver` 适配器：自动读取关节状态，结果以 `ArticulationAction` 返回。
- `LulaKinematicsSolver` 实现：需 URDF + YAML robot description（声明主动关节、默认构型）。文档强调：IK 解直接发给机器人仅适合演示，实用场景须配合轨迹规划。配套 XRDF Editor / Robot Description Editor 生成配置。

**轨迹接口**：

- `Trajectory` 接口：c-space 位置对时间的连续函数，访问器 `start_time`/`end_time`/`active_joints`/`joint_targets(time)`。
- `ArticulationTrajectory`：桥接到 Articulation，`get_action_at_time()`、`get_action_sequence()`。
- `LulaCSpaceTrajectoryGenerator`：c-space 路点样条插值，生成速度/加速度/jerk 极限饱和的时间最优轨迹；`LulaTaskSpaceTrajectoryGenerator`：任务空间目标线性插值，内部用 Lula IK 转 c-space；均需 URDF + YAML（含加速度/jerk 限制）。

**RMPflow**：Riemannian Motion Policy = 加速度策略 + 权重矩阵，融合多个运动目标：C-Space Target、Target RMP（末端位置）、Axis Target（姿态轴）、Joint Limits、Joint Velocity Caps、Collision Avoidance、Damping。需三个文件：URDF、robot description YAML（含碰撞球）、RMPflow config YAML（各 RMP 参数）。调试：`visualize_collision_spheres()` 查看内部障碍表示，`set_ignore_state_updates(True)` 将策略与物理仿真解耦，区分是策略问题还是 PD 增益问题；经 `ArticulationMotionPolicy` 接入机器人。

**cuRobo / cuMotion**：cuRobo 是 NVIDIA Research 的 GPU 加速运动生成库，支持批量无碰撞 IK、无碰撞运动规划、基于 MPPI 的反应式控制、经 nvblox 接入深度传感器做在线障碍建图；直接对接 Isaac Sim（2022.2.1+）。cuMotion 为其产品化封装（Developer Preview），以 MoveIt 2 插件形式提供 ROS 2 集成。已知问题：NvBlox 示例存在 issue，aarch64 不支持该教程。

来源：manipulators/motion_generation_overview.html、manipulators/concepts/kinematics_solver.html、manipulators/concepts/trajectory_interface.html、manipulators/concepts/rmpflow.html、manipulators/manipulators_lula_kinematics.html、manipulators/manipulators_curobo.html

### 4. 表面吸附夹爪与 Grasp Editor

**Surface Gripper**：模拟吸盘式夹爪，在接触点用 D6 关节将父/子刚体绑定，解析 USD Surface Gripper Schema。关键属性：Attachment Points（D6 关节）、Max Grip Distance（闭合接受范围）、Shear/Coaxial Force Limits（超限则约束断开）、Retry Interval（尝试闭合时长）。附着关节要求：D6 类型、已启用、Body 0 一致、"Exclude from Articulation"=True。API：`acquire_surface_gripper_interface()`，`close_gripper()` / `open_gripper()` / `set_gripper_action()`（-1~1），`get_gripper_status()` 返回 Open/Closed/Closing；GUI 经 Create > Robots > Surface Gripper；另有 OmniGraph 节点。

**Grasp Editor**：UI 扩展，生成 `isaac_grasp` YAML（单文件存储某夹爪-物体对的抓取列表：相对位姿、开/闭关节构型、置信度）。流程：导入夹爪（独立或臂载）与物体 → 选参考 frame → 手动摆位、物理仿真闭合、可施加外力测试稳定性 → 导出；支持导入回放验证。提供 Python API 供下游（如运动规划）由物体位姿反算夹爪位姿；参考 frame 须与 URDF/视觉系统一致。

来源：robot_simulation/ext_isaacsim_robot_surface_gripper.html、robot_simulation/grasp_editor.html

### 5. RL 策略示例与调参 Tips

- **Policy Example 扩展**：部署 Isaac Lab 训练的 RL 策略，内置示例：Unitree H1（平地行走，方向键控制）、Boston Dynamics Spot（四足全向）、Franka（开抽屉）；另提供 ANYmal C。策略分发格式：`.pt` 策略网络 + `env.yaml`（环境参数）+ `agent.yaml`（网络结构）。standalone 脚本：`h1_standalone.py`（支持多机器人实例）、`spot_standalone.py`、`anymal_standalone.py`（带地形感知），经 `./python.sh` 运行。
- **仿真 Tips**：提速——减少场景物体数量/复杂度、减少仿真步数；车轮打滑——调整轮与地面的接触材料摩擦；抓取不稳——提高手指与物体摩擦、用 Physics Authoring Toolbar 的 Mass Distribution Tool 检查质量分布、提高夹爪关节 stiffness。

来源：robot_simulation/ext_isaacsim_robot_policy_example.html、robot_simulation/robot_simulation_tips.html

## 二、传感器

总览：分为相机、深度相机、RTX 传感器（Lidar/Radar/Annotators/非视觉材质）、物理传感器（关节/接触/Effort/IMU/接近）、PhysX SDK 传感器（Generic/Lidar/Lightbeam）五类。来源：sensors/index.html

### 1. 相机与深度相机

- 相机基于 Camera USD prim，数据经 render product 获取。Python：`isaacsim.sensors.camera.Camera(prim_path, position, frequency, resolution)`；GUI：Create > Camera。
- 标定支持多种畸变模型：OpenCV Pinhole（径向+切向）、OpenCV Fisheye、fisheye 多项式等，需内参矩阵（焦距、主点）与畸变系数。数据：RGB/RGBA、深度、运动矢量、图像↔世界坐标变换。Camera Inspector（Tools > Sensors）可多视口检查相机覆盖与位姿。
- 深度相机：`SingleViewDepthSensor` 封装 Camera prim，用后处理管线模拟**双目立体深度估计**（非真实双渲染，也不适用 ToF/结构光）。输出模式：`DepthSensorDistance`（模拟立体深度）与 `DistanceToImagePlane`（到像平面的真实垂直距离）。`SingleViewDepthSensorAsset` 可直接加载官方资产（如 RealSense D455）并自动管理 render product 与 annotator；自建传感器流程：导入几何→加标定相机 prim→对齐内外参→迭代调 schema 属性拟合实机输出。

来源：sensors/isaacsim_sensors_camera.html、sensors/isaacsim_sensors_camera_depth.html

### 2. RTX 传感器（Lidar / Radar / Annotators）

- 基于 RTX 渲染器的 RTX Sensor SDK 做真实光线追踪，能建模透明/反射表面（Lidar）与材料发射率/反射率（Radar），精度高于 PhysX 近似但计算开销更大。Motion BVH（默认关闭，经 `SimulationApp` 构造参数启用）用于 Lidar 运动补偿与 Radar 多普勒建模（后者必需）。
- **RTX Lidar**：运行时渲染的 `OmniLidar` prim。创建：`IsaacSensorCreateRtxLidar` 命令，或高层类 `LidarRtx`（自动挂 render product 与 annotator）。支持厂商预设配置（如 `HESAI_XT32_SD10`，可用简名/变体如 SICK picoScan150 的 `Normal_11`）。旧 JSON 配置有迁移工具转 USD prim。推荐用 Replicator Annotator 取数，主输出为 `GenericModelOutput` buffer。
- **RTX Radar**：`OmniRadar` prim + `OmniSensorGenericRadarWpmDmatAPI` schema，`IsaacSensorCreateRtxRadar` 创建，tick rate 默认 20 Hz；材质发射率/反射率影响回波；`force_camera_prim=True` 的旧 Camera prim 方式自 5.0 起弃用。
- **Annotators**：
  - `IsaacCreateRTXLidarScanBuffer`：累积整圈扫描，默认输出 float3 点云，可选 azimuth/elevation/distance/intensity/timestamp(uint64 ns)/emitterId/materialId/objectId（需启用 stable IDs）/法向/速度；
  - `IsaacComputeRTXLidarFlatScan`：2D lidar 的 depth+azimuth，要求发射器仰角为 0，不支持 radar；
  - `IsaacExtractRTXSensorPointCloudNoAccumulator`：逐帧处理 GenericModelOutput 不累积。
  - 挂载：Replicator API 或 `sensor.attach_annotator("IsaacCreateRTXLidarScanBuffer", outputTimestamp=True)`；需 timeline 播放中才有数据。

来源：sensors/isaacsim_sensors_rtx.html、sensors/isaacsim_sensors_rtx_lidar.html、sensors/isaacsim_sensors_rtx_radar.html、sensors/isaacsim_sensors_rtx_annotators.html

### 3. 物理传感器（isaacsim.sensors.physics）

基于 CPU 物理仿真、渲染后运行，输出物理引擎精确值（可后处理加噪声），输出率上限为物理频率（可插值）。含五种：Articulation Joint Sensors（关节力/力矩）、Contact、Effort（关节电机力）、IMU、Proximity。

- **Contact Sensor**：基于 PhysX Contact Report API，模拟压力/接触传感器，按附着对象过滤并可限定区域（如足端）。参数：`radius`、`enabled`、`min_threshold`/`max_threshold`、`sensor_period`（不能小于物理步长）。创建：GUI（Create > Sensors > Contact_sensor）、命令 `IsaacSensorCreateContactSensor`、类 `ContactSensor`。读数：`get_sensor_reading()`（CsSensorReading：有效性/时间戳/合力/接触状态）、`get_current_frame()`（力、接触数、body 对、位置、法向、冲量）、`get_contact_sensor_raw_data()`（不受阈值过滤的原始数据）。
- **IMU Sensor**：输出局部坐标系线加速度、角速度、姿态四元数、时间戳与物理步数。参数：`enabled`、sensor period、`angularVelocityFilterWidth` / `linearAccelerationFilterWidth` / `orientationFilterWidth`（插值缓冲为 max(2×filter width, 20)）。创建：命令 `IsaacSensorCreateImuSensor` 或类 `IMUSensor`；读数：`get_sensor_reading()`（可插值）、`get_current_frame()`、OmniGraph "Isaac Read IMU" 节点。注意：传感器在播放时动态初始化，改层级需先停止仿真。

来源：sensors/isaacsim_sensors_physics.html、sensors/isaacsim_sensors_physics_contact.html、sensors/isaacsim_sensors_physics_imu.html

### 4. PhysX SDK 传感器（isaacsim.sensors.physx）

用 PhysX raycast 测距，输出物理引擎精确值，默认输出率受渲染率限制。三种：Generic Sensor（通用 raycast）、PhysX Lidar、Lightbeam Sensor。

- **PhysX Lidar**：raycast 模拟，提供真值深度；与 RTX Lidar 区别——不感知非视觉材质、会穿透透明物体。参数：水平 FOV（默认 360°）/垂直 FOV、分辨率（度）、rotation rate（0 = 全部射线同时发射）、min/max range。数据 API：线性深度、zenith/azimuth 角、点云、语义分割（需启用）。目标物体须开启碰撞；附着到父几何可随动。

来源：sensors/isaacsim_sensors_physx.html、sensors/isaacsim_sensors_physx_lidar.html

## 三、物理仿真

### 1. 仿真基础

- **流程**：USD Physics schema 解析 → PhysX SDK 后端建对象 → 步进 → 状态写回 USD；运行时参数可传播修改。
- **时序**：物理步率与渲染帧率解耦，默认均 60；每渲染帧可多个物理 substep。帧率在 Root Layer 的 "Timecodes per second" 设置；物理步率在 Physics Scene 属性 "Simulation Steps per Second"（Create > Physics > Simulation Scene）。
- **刚体与碰撞**：Rigid Body API 启用动力学；Collider API 启用碰撞，形状近似：convex hull（默认）、凸分解、球近似、SDF mesh。`Contact Offset`（提前生成接触约束的距离，越大越早检测但更耗时）、`Rest Offset`（碰撞几何胀缩以对齐视觉网格）、CCD（场景与刚体都需启用，防高速穿透）。
- **接触与摩擦**：物理材料定义 restitution/friction，combine mode 有 average/min/multiply/max；Compliant Contacts 以弹簧-阻尼近似软体接触。
- **关节与 Articulation**：关节可配局部 frame、轴、限位、drive（位置/速度控制）；Articulation 对多关节结构性能与稳定性优于独立关节。
- OmniGraph 可用 "On Physics Step" 节点（PipelineStageOnDemand）与物理同步；Simulation Data Visualizer 可查看约束残差（residual reporting）评估收敛质量。

来源：physics/index.html、physics/simulation_fundamentals.html

### 2. PhysX 限制（要点）

- 自定义几何的接触数据不经 tensor API 的 GPU getter 暴露，建议改用 convex hull 或球/盒/胶囊等 GPU 原生形状；自定义几何回调与传送带特性会触发 CPU 路径拖慢 GPU 管线。
- 粒子与可变形体：无 contact report、不能与传送带交互、不支持静摩擦与 friction combine mode。
- 确定性：从仿真中途保存的"接触中"状态回放可能非确定，应从头重启。
- 关节：非单位质量变换的球形 articulation 关节可能行为异常；D6 Joint Drive 与 TGS solver 组合表现不一致；isosurface 渲染存在内存泄漏风险。

来源：physics/physics_resources.html

### 3. Physics Inspector

Tools > Physics > Physics Inspector 打开（配套 Physics Toolbar：Tools > Physics Toolbar）。可检查/调整关节配置与约束、关节限位、drive 参数、物体位姿，用于 authoring 阶段调参。**重要警告**：Inspector 会部分初始化 `omni.physx`，打开期间一般仿真可能行为异常；调完后关闭窗口即可恢复正常。

来源：physics/joint_inspector.html
