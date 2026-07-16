# 资产导入与机器人搭建（6.0.1）

## 一、格式与导入器（含 Newton schema 变化）

- **支持格式**：Asset Converter 处理 `.fbx`/`.obj`/`.gltf`；材质用 `.MDL`；机器人格式为 URDF、MJCF；另有 Onshape Importer、CAD Converter（外部扩展）。**ShapeNet Importer 已弃用**。
- **URDF Importer**（`isaacsim.asset.importer.urdf`，File > Import）：选项含 Collision From Visuals（Convex Hull/Decomposition/包围球/包围盒）、Allow Self-Collision、Robot Type（Manipulator/Wheeled/Humanoid 等）、Base Type（Source/Fixed/Mobile）、Merge Mesh、ROS package 路径映射。**6.0 变化**：根 link 同时写入 `UsdPhysics.ArticulationRootAPI` 与 `NewtonArticulationRootAPI`；自碰撞改用 `newton:selfCollisionEnabled`；mimic 关节走 `NewtonMimicAPI`；joint 属性同时映射 PhysX 与 MJCF schema，实现多物理引擎；输出遵循 Isaac Sim Asset Structure，mesh 可实例化。
- **MJCF Importer**（`isaacsim.asset.importer.mjcf`）：选项类似，另有 Import Scene（带入 MuJoCo 仿真设置）；actuator 的 gain/bias 参数映射到 drive；**限制**：同一 body 对之间多个单轴关节会尝试转 D6，轴冲突可能丢 DOF。6.0 Newton schema 变化与 URDF 相同。
- **URDF Exporter**（`isaacsim.asset.exporter.urdf`，File > Export to URDF）：可设 mesh 目录/URI 前缀、根 prim。**限制**：不支持运动学闭环、孤立关节。
- **教程**：import_urdf（UR10，GUI/Python API/ROS 2 节点三种方式，导入后需查看 collider 并调 gain）、export_urdf、import_mjcf、shapenet_importer。

来源: importer_exporter/importers_exporters.html、ext_isaacsim_asset_importer_urdf.html、ext_isaacsim_asset_importer_mjcf.html、ext_omni_exporter_urdf.html、formats.html、import_urdf.html

## 二、6.0 新工具

- **Robot Inspector Window（新）**（Window > Robot Inspector，`isaacsim.robot.schema.ui`）：查看运动学结构，三种层级模式（Flat/Tree/MuJoCo）；三种非破坏性 mask 操作——Deactivate（禁用）、Bypass（禁用并重连链）、Anchor（固定到世界）；更改存于会话临时层不落盘；视口关节连线可视化；提供 `HierarchyMode`/`MaskingState`/`MaskingOperations` Python API。
- **Robot Poser（新）**（Tools > Robotics > Robot Poser）：创建/编辑/应用命名姿态，交互式 IK 拖拽末端；停止仿真时直接 teleport 关节、运行时作为 drive target；IK 不可达时 link 红色高亮；姿态按 Robot Schema 存为 USD prim 随资产携带。
- **Self-Collision Detector（新）**（Tools > Robotics > Asset Editors）：静态扫描重叠碰撞对，表格中勾选 Filtered Pair 屏蔽，支持批量、搜索、视口高亮、含环境物体；适合处理刚导入的 URDF/MJCF 及灵巧手密集碰撞体。
- **Asset Transformer（6.0 新增，重点）**（Tools > Robotics > Asset Editors > Asset Transformer）：基于规则流水线批量把 USD 转成优化的仿真就绪结构，解决 schema 分层、几何/材质去重、层级重组、组合弧生成四类问题。规则顺序执行，配置可存 JSON profile，自带 "Isaac Sim Structure" 默认档（13 步）；输出 Transform Report（逐规则日志）。
  - **API**：`AssetTransformerManager.run(input_stage, profile, package_root)` 返回 `ExecutionReport`；`RuleProfile`/`RuleSpec`（name/type/destination/params/enabled）、`RuleRegistry`、自定义规则继承 `RuleInterface`；`RuleProfile.from_dict()` 读 JSON。
  - **规则四类**：Core（SchemaRoutingRule、PropertyRoutingRule、PrimRoutingRule、RemoveSchemaRule）、Performance（MaterialsRoutingRule、GeometriesRoutingRule 去重+实例化）、Structure（FlattenRule、VariantRoutingRule、InterfaceConnectionRule）、Isaac Sim（RobotSchemaRule、PhysicsJointPoseFixRule、MergeMeshRule、Urdf/MjcToPhysx 转换规则支撑多物理引擎）。

来源: robot_setup/robot_inspector.html、robot_poser.html、ext_isaacsim_robot_setup_collision_detector.html、asset_transformer.html、asset_transformer_api.html、asset_transformer_rules.html

## 三、编辑与检查工具

- **Gain Tuner**：四种测试（Snap-to-Limits、正弦、阶跃、Stress Test 的 Random Walk/Adversarial 模式），测量 vs 指令曲线图；支持按自然频率调参、关节分组、RNG 种子复现。公式：Effort = Kp·(位置误差) + Kd·(速度误差)。
- **Robot Assembler**：用物理仿真的固定关节把两资产合并（机械臂+夹爪等），指定 attach 点（Link/Site）、90° 旋转微调、"Assemble and Simulate" 验证；两种模式（改资产生成 variant / 只改当前 stage）；`RobotAssembler` API（`begin_assembly/assemble/finish_assemble`）。
- **Mesh Merge Tool**：合并同一刚体下多 mesh（已标注将由 Scene Optimizer 取代）。
- **Inspector 工具**：Joint Inspector、Physics Inspector（仿真中滑杆验证 DOF）、Simulation Data Visualizer。
- **弃用：Robot Wizard 及其教程已弃用**（robot_wizard.html）。

来源: robot_setup/ext_isaacsim_robot_setup_gain_tuner.html、assemble_robots.html、ext_isaacsim_util_merge_mesh.html、robot_setup/index.html

## 四、13 篇 Robot Setup 教程脉络与关键参数

脉络：T1 场景搭建 → T2 组装简单机器人 → **T3 关节化**（刚体+revolute 关节，轮子速度控制用高 damping=1e4、零 stiffness、目标速度 200 rad/s；Articulation Root 提升精度与大质量比容忍；注意控制器用弧度、USD 属性用角度）→ T4 相机与传感器 → **T5 移动机器人（叉车）**（升降 prismatic：Damping 1e4/Stiffness 1e5；转向 Damping 100/Stiffness 1e5；轮子用圆柱碰撞近似防颠簸；后轮速度 -200 前进）→ T6 组装机械臂 → **T7 配置机械臂（UR10e+Robotiq）**（physx.usda 子层；Solver 位置迭代 64、速度迭代 4；Sleep/Stabilization 阈值调低；指尖摩擦 1.0；夹爪 Max Force 200）→ T8 生成机器人配置文件 → T9 抓放示例 → T10 闭环结构 → **T11 关节增益调参（UR10）**（Gain Tuner：肩/肘 stiffness 5e4–5e5、腕 50–500、damping 0–50；必须设速度上限否则求解器不稳）→ **T12 资产优化（Jetbot）**（mesh 合并+Instanceable 实例化，40→64 FPS；灯光≤10、简化碰撞形状）→ **T13 腿式机器人（H1）**（按 policy 环境定义设 stiffness/damping/effort/velocity limit，rad↔deg 换算 S_deg=S_rad×π/180；armature 与关节摩擦在 Raw USD 属性；Newton 对反向关节可能报错需交换 body；参考资产 Isaac/Samples/Rigging/H1/h1_rigged.usd）。

来源: robot_setup_tutorials/tutorial_gui_simple_robot.html、rig_mobile_robot.html、tutorial_configure_manipulator.html、joint_tuning.html、optimizing_asset.html、tutorial_rig_legged_robot.html

## 五、OpenUSD & Tuning Best Practices 系列（6.0 新增，7 篇）

以 Inspire Hand 灵巧手为贯穿案例（`IsaacSim/Samples/Rigging/Inspire/`），4 个检查点渐进：

1. **Setup**：下载资产、打开起始场景，前置要求（USD 基础、刚体物理）。
2. **Asset Structure**：讲解 **Asset Structure 3.0**——geometries.usd / materials.usda / robot.usda / instances.usda / base.usda / physics.usda（引擎无关）/ physx.usda（引擎特定）；根文件用 Physics variant（none/physics/physx）切换后端，支持 MuJoCo/PhysX 多引擎共存。
3. **Inspect**：Joint Visualizer、Robot Inspector、Physics Debugger（质心与主惯性轴）、Collider 可视化；检查关节类型、质心位置、惯性合理性、碰撞策略。
4. **Collider Pairs**：开启自碰撞→用 Self-Collision Detector 找重叠对→Filtered Pair 屏蔽；过滤写在中性 physics.usda 层；过滤过度会穿模、不足会不稳。
5. **Joint Drive**：区分"限制"与"增益"；mimic 关节设非柔顺（damping ratio=0、自然频率=0）；最大力矩由握力×力臂×传动比推算（掌指 1.68 Nm），最大速度按厂商规格（260 deg/s）。
6. **Joint Gains**：Gain Tuner 阶跃/正弦测试；先调 stiffness（damping 从 0 起），damping 约低一个数量级；目标接近临界阻尼。
7. **Dexterous Hand 实践**：总结全流程，产出可用于 Isaac Lab 的生产级灵巧手资产。

来源: openusd_tuning_tutorials/tutorial_01_setup.html 至 tutorial_07_practice.html

## 六、资产校验与排障

- **Asset Validation**（Window > Asset Validator，`isaacsim.asset.validation`）：三类规则——IsaacSim.PhysicsRules（11 条：关节驱动、速度限制、质量、碰撞体、articulation）、IsaacSim.RobotRules（10 条：命名规范、RobotAPI/JointAPI/LinkAPI、schema 分层）、IsaacSim.SimReadyAssetRules（3 条：材质层级）；报告带自动修复建议。
- **Troubleshooting**：机器人"爆炸"多因关节处碰撞体重叠；导入常见问题（材质重名、body/joint 按字母序需重排、非零初始 target 导致第一帧乱动、max force 不合理）；控制器问题（Robotiq 并联机构 link 不动、差速轮摩擦不足打滑）；速查表：关节不动查限位/增益、抖动降增益加子步、穿地调 contact offset、性能差简化碰撞体。

来源: robot_setup/asset_validation.html、robot_setup/troubleshooting.html

**弃用项汇总**：Robot Wizard 及其教程、ShapeNet Importer、Mesh Merge Tool（将由 Scene Optimizer 取代）。
