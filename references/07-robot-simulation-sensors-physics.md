# 机器人仿真、传感器与物理（6.0.1）

## 控制体系与控制器

**Articulation Controller（关节控制）**：底层关节命令机制，支持 position/velocity/effort 三种模式（角度用弧度），同一关节只能用一种控制方式。核心类：`SingleArticulation`（初始化时隐式创建控制器）、`ArticulationController`（需用 ArticulationView 手动 `initialize()`）、`ArticulationAction`（打包 positions/velocities/efforts + joint_indices），经 `apply_action()` 下发；另有同名 OmniGraph 节点（targetPrim、jointNames、position/velocity/effortCommand）。注意：该 API 已标弃用，新开发应使用 `isaacsim.core.experimental.prims` 的 `Articulation`。
来源: robot_simulation/articulation_controller.html

**移动机器人控制器**：`DifferentialController`（双轮差速，参数 wheel_radius、wheel_base、maxLinear/Angular/WheelSpeed）、`HolonomicController`（麦轮全向，wheel_positions/orientations、mecanum_angles，配 `HolonomicRobotUsdSetup` 辅助类）、`AckermannController`（阿克曼转向，wheel_base、track_width、前后轮半径、invert_steering），均有 OmniGraph 节点。
来源: robot_simulation/mobile_robot_controllers.html

**Surface Gripper（吸盘夹爪）**：基于 D6 joint 在接触点建约束；USD Surface Gripper Schema + Attachment Point API（Clearance Offset、Forward Axis）。关键属性：Max Grip Distance、Coaxial/Shear Force Limit、Retry Interval。代码用 `robot_schema.CreateSurfaceGripper()`，控制用 `close_gripper()/open_gripper()/set_gripper_action(-1~1)` 或 OmniGraph 节点；6.0 有绑定导入迁移指南。
来源: robot_simulation/ext_isaacsim_robot_surface_gripper.html

**Grasp Editor**：为特定夹爪/物体对手工创作并仿真验证抓取，五步流程（选夹爪与物体→定参考系→配置主动自由度→仿真闭合→导出），输出 `isaac_grasp` 格式 YAML（相对位姿、抓取/预抓取关节位置、置信度），支持导入验证、外力测试、碰撞屏蔽，含 Python API。
来源: robot_simulation/grasp_editor.html

**RL Policy 示例**：H1、Spot、Go2（平地行走）与 Franka（开抽屉）策略，均有 **PhysX/Newton 双版本**并按当前物理引擎自动选择；菜单 Robotics Examples > POLICY 加载，附 standalone 脚本。
来源: robot_simulation/ext_isaacsim_robot_policy_example.html

**仿真技巧**：物理与渲染步长对齐（`SimulationManager.setup_simulation(dt)` + `RenderingManager.set_dt()`）；轮式机器人调轮/地面摩擦（≤1.0）；夹爪抓不住则增摩擦、查质量（MassAPI 设质量/质心/惯量）、提 stiffness/damping；碰撞体只加在交互部件上，优先简单形状，convex decomposition 仅用于指尖等。
来源: robot_simulation/robot_simulation_tips.html

## 旧运动栈弃用状态

Lula 全家（RMPflow、RRT、Kinematics Solver、Trajectory Generator、XRDF Editor）**已弃用**，官方指向新的 Robot Motion (Experimental) API；另一替代是 GPU 加速的 cuRobo（批量无碰 IK、运动规划、反应式避障）。文档提供迁移指南入口。
来源: manipulators/motion_generation_overview.html

## 实验性运动栈（6.0 新增）

**Motion Generation (Experimental)**：统一实时运动生成框架。场景交互：`SceneQuery`（按条件发现场景物体）、`ObstacleStrategy`（障碍几何近似，prim 级/形状级/默认三层优先级）、`WorldInterface`（把 warp 数组障碍数据适配到规划库）、`WorldBinding`（基于 USDRT 变更追踪自动同步世界模型）。控制：`RobotState`（joint/site/link/root 空间统一状态）、`BaseController`（`reset()`/`forward(state, setpoint)→desired state`）、组合类 `ParallelController`/`SequentialController`/`ControllerContainer`（运行时切换）。轨迹：`Trajectory` 接口（duration + `get_target_state(t)`）、`Path.to_minimal_time_joint_trajectory()`（限速限加速的梯形速度剖面）、`TrajectoryFollower` 执行任意 Trajectory——规划与执行解耦、"unopinionated" 设计。
来源: motion_generation/index.html、scene_interaction.html、mobile_robot_control_example.html、trajectory_planning.html

**cuMotion Integration**：GPU 加速操作臂运动生成（Lula 后继，但 API 差异大），薄封装直接暴露原生 Python API。四大算法：RMPflow（反应式）、Graph-Based Planner（RRT 类采样规划）、Trajectory Generator（时间最优）、Trajectory Optimizer（全局优化）。机器人配置需 URDF + XRDF（碰撞球、自碰撞、工具坐标系，区分 active/fixed 关节），`load_cumotion_supported_robot()`（Franka/UR10）或 `load_cumotion_robot()` 返回 `CumotionRobot`。`CumotionWorldInterface` 实现 WorldInterface：cuboid 原生支持，圆锥/圆柱用 OBB，mesh 可选 OBB 或 TRIANGULATED_MESH；`synchronize_transforms()/synchronize()` 同步。`RmpFlowController` 实现 BaseController（rmp_flow.yaml 配增益/限速/安全裕度）；`GraphBasedMotionPlanner.plan_to_cspace/pose/translation_target()` 返回 Path；`TrajectoryGenerator.generate_trajectory_from_cspace_waypoints()`；`TrajectoryOptimizer.plan_to_goal()` 接受 `CSpaceTarget`/`TaskSpaceTarget`。附 `isaac_sim_to_cumotion_pose()` 坐标转换。
来源: cumotion/index.html 及 tutorial_robot_configuration/world_interface/rmpflow/graph_planner/trajectory_generator/trajectory_optimizer.html

**PINK Integration**：基于 Pinocchio + QP 求解器的微分 IK，任务加权为目标、安全约束为不等式。`PinkRobot`（URDF 解析，`load_pink_supported_robot()`/`load_pink_robot()`，`build_collision_model=True` 启用自碰撞）、`PinkIKController`（实现 BaseController，输出关节速度积分为 RobotState；参数：position/orientation/posture cost、dt 需匹配控制频率、QP 求解器 osqp/clarabel）。多任务：`FrameTask`、`PostureTask`、`DampingTask`，可加 RelativeFrameTask/ComTask/JointCouplingTask 与 `SelfCollisionBarrier`（需 SRDF）；cost 单位规范化（cost/m、cost/rad），可运行时调整。**仅 Linux x86_64 提供 wheel**。
来源: pink/index.html 及 tutorial_robot_configuration/ik_controller/multi_task.html

## Newton Actuators（6.0 新增）

一次定义执行器行为，在 Isaac Sim 与 Isaac Lab 中运行一致，配置随 USD 资产携带；**不要求 Newton 物理后端**，PhysX 亦可用。管线三段：Delay（可选，延迟建模）→ Controller（必选：PD、PID、神经网络 MLP/LSTM）→ Clamping（可选多个：`ClampingMaxEffort` 对称饱和、`ClampingDCMotor` 转矩-转速包络、`ClampingPositionBased` 位置查找表）。

- **Python**（`isaacsim.core.experimental.actuators`）：`ActuatorConfig` 打包控制器/钳位/延迟，`ArticulationActuators.from_actuators(robot_path, actuators=[(config, joint), ...])` 附加并自动注册物理回调；命令走 `set_dof_position_targets()` / `set_dof_feedforward_effort_targets()`；`set_dof_armatures()` 加虚拟转子惯量。
- **USD**：articulation root 下 `Actuators` scope 内的 `NewtonActuator` prim，`newton:targets` 指向关节，应用 `NewtonPDControlAPI`/`NewtonPIDControlAPI`/`NewtonMaxEffortClampingAPI` 等 schema；`add_actuator()` 辅助创作；仅凭 root 路径构造 `ArticulationActuators` 即自动发现解析。
- **OmniGraph**：实验性 "Articulation Actuators" 节点（robotPath、feedforwardCommand、autoStepPrePhysics、stepDt）配合 Articulation Controller 节点，要求资产已有 USD 执行器。
- **Tips**：高频振荡两大成因——armature 不足（经验公式 I_armature = I_rotor·N²）与物理步长过粗；高增益控制器建议把物理频率从 60 Hz 提到 120–200 Hz。

来源: newton_actuators_tutorials/index.html 及 newton_actuators_python/usd/omnigraph/tips.html

## 传感器

**相机（experimental API）**：`isaacsim.sensors.camera` 弃用，迁移到 `isaacsim.sensors.experimental.rtx`，统一"创作/运行时"分层：`RtxCamera`（OmniSensorAPI，光学参数）+ `CameraSensor`（管理 render product、annotator，`get_data()` 返 warp 数组）；支持 tick_rate 独立控制、OpenCV 针孔/鱼眼畸变、`TiledCameraSensor` 批渲染。**Depth**：`SingleViewDepthCameraSensor` 单视图后处理模拟双目深度（非 ToF/结构光），支持 RealSense D455 资产、DepthSensorDistance 等 annotator。**Structured Light（新）**：`StructuredLightCamera` 用多个带 ShapingAPI 的 RectLight 投影序列图案，参数 projector_timestamps/cycle_period/intensity，按仿真时间切换图案。
来源: sensors/isaacsim_sensors_camera.html、_camera_depth.html、_camera_structured_light.html

**RTX 传感器**：`isaacsim.sensors.experimental.rtx` 取代弃用的 `isaacsim.sensors.rtx`；prim 类型 `OmniLidar`/`OmniRadar`/`OmniAcoustic`，输出统一走 `GenericModelOutput` AOV。**Lidar**：`Lidar.create()`（config 如 HESAI_XT32、Ouster OS、SICK picoScan；tick_rate、aux_output_level、accumulate_outputs），`LidarSensor` + "generic-model-output" annotator。**Radar**：`OmniSensorGenericRadarWpmDmatAPI`；**必须启用 Motion BVH** 才能正确建模多普勒（默认关闭）。**Acoustic（新）**：GPU 超声波传播仿真，输出发射-接收对的幅值信号（非点云）；`OmniSensorGenericAcousticWpmAPI`，概念：sensor mounts、receiver groups，参数 centerFrequency（如 40 kHz）；6.0 GA 中忽略 tickRate 恒自动触发。**Annotators**：推荐 `IsaacExtractRTXSensorPointCloud`，旧 IsaacCreateRTXLidarScanBuffer 等弃用；aux 级别 BASIC（emitterID/channelID/tickID）、EXTRA（+materialID/objectID）、FULL（+速度/法线），用 `parse_generic_model_output_data()` 解析。**Multi-tick rendering（新）**：每个传感器独立 `omni:sensor:tickRate` 与仿真帧率解耦，降低 GPU 负载；限制：Lidar tickRate 须等于 scanRateBaseHz，Radar 忽略 tickRate，Radar/Lidar/Motion BVH 混用启动可能崩溃（5 帧预热规避）。**自定义 profile（新）**：直接改 sensor prim 的 USD 属性（扫描方式、FOV、firing pattern、强度模型），建议从厂商配置模板改起；该章仍在完善中。
来源: sensors/isaacsim_sensors_rtx.html、_rtx_lidar/radar/acoustic/annotators.html、_multitick_rendering.html、_rtx_custom.html

**物理传感器**：新扩展 `isaacsim.sensors.experimental.physics`（旧 `isaacsim.sensors.physics` 弃用），含 Contact、IMU、Effort、Articulation Joint、**Joint State（新）**、**Physics Raycast（新）** 六种，按物理频率输出真值。JointStateSensor 挂在 articulation root，`get_sensor_reading()` 一次返回全 DOF 名称/位置/速度/力矩/类型。Raycast：`Raycast.create()` 支持逐射线 origins/directions/timeOffsets（可做固态/旋转/光幕模式），输出 depths/hit positions/normals，配 `IsaacReadRaycastSensor`、`DebugDrawRayCast` OmniGraph 节点。
来源: sensors/isaacsim_sensors_physics.html、_physics_joint_state.html、_physics_raycast.html

**PhysX 传感器现状**：`isaacsim.sensors.physx`（generic、lidar、lightbeam、proximity）**全部弃用**，统一由 `RaycastSensor` 替代。
来源: sensors/isaacsim_sensors_physx.html

## 物理

**基础**：USD Physics/PhysX Schema 定义属性；物理步长（Physics Scene "Simulation Steps per Second"，默认 60）与渲染帧率分离（如 120 Hz 物理 / 60 fps 渲染 = 每帧 2 步）；核心组件 Rigid Body、Collider、Joint、Articulation（机器人首选）；快速物体开 CCD 防穿透。
来源: physics/simulation_fundamentals.html

**Physics Data Flow & Engine Integration（新）**：三条数据通路——USD（创作/持久化）、Fabric（GPU 运行时存储，每帧写 `omni:fabric:worldMatrix` 供渲染）、Physics Tensors（`ArticulationView`/`RigidBodyView` 批量读写，支持 NumPy/PyTorch/Warp）；顶层 Unified API（`isaacsim.core.experimental`）自动路由到当前引擎。新引擎接入需：解析 USD、每帧写 Fabric、实现标准 Tensor API，并经 Simulation/Stage Update Function 适配器注册；`isaacsim.physics.newton` 为参考实现。
来源: physics/new_physics_engine.html

**Newton Physics Backend（新，实验性）**：基于 NVIDIA Warp、集成 MuJoCo Warp 的开源 GPU 物理引擎（Disney Research/DeepMind/NVIDIA 共同维护）。求解器：MuJoCo（积分器 euler/rk4/implicit/implicitfast，文档最全）、XPBD、Featherstone、SemiImplicit。启用：`isaac-sim.newton.sh/.bat` 或运行时 `SimulationManager.switch_physics_engine("newton")`；配置类 `PhysicsScene` + `NewtonMjcScene`。资产硬性要求：joint body0 为父、无闭链、质量/惯量非零、碰撞体非零、所有关节在 articulation root 内；许多工作流尚不支持。
来源: physics/newton_physics.html

**PhysX 限制**：GPU convex hull 限 64 顶点/面；粒子与可变形体不支持接触报告/传送带/静摩擦（particle cloth 将弃用）；中途存档恢复不保证确定性（需从头重放）；球关节非单位质心变换可能异常、TGS + velocity iterations 下 D6 drive 行为异常、D6 闭环易不稳、PGS/TGS 摩擦系数不一致；腱（tendon）保真度不足，建议用 mimic joint 替代。
来源: physics/physics_resources.html

**Physics Inspector**：Tools > Physics > Physics Inspector，检查物理/关节属性；注意它会部分初始化 omni.physx 干扰正常仿真，关闭窗口即恢复。
来源: physics/joint_inspector.html
