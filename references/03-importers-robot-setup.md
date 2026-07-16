# 资产导入与机器人搭建

## 支持的格式总览

- **USD** 是 Omniverse 场景标准格式，材质用 MDL；FBX/OBJ/glTF 可经 Asset Converter 转 USD。
- 机器人描述格式：**URDF 导入器**、**MJCF 导入器**（均默认启用、已开源）、**Onshape 导入器**（云端 CAD）、**CAD 转换器**；ShapeNet 导入器已弃用。
- 反向导出：**USD → URDF Exporter**（`isaacsim.asset.exporter.urdf`，File > Export to URDF）。参数：输出路径、Mesh Directory Path（默认与 URDF 同目录，mesh 存入 meshes 子目录）、Mesh Path Prefix（如 `file://`）、Root Prim Path、Visualize Collisions。限制：URDF 不支持运动学闭环（有闭环则转换失败）；关节缺少 parent/child（如未连接的末端执行器）会导致失败。
- 注意：Isaac Sim 5.1.0 已停止支持。

来源：importer_exporter/importers_exporters.html、formats.html、ext_omni_exporter_urdf.html

## URDF 导入器关键选项

入口 File > Import。选项：

- **模型放置**：在 Stage 中创建 / 作为 Reference 添加；可选 Set as Default Prim、Clear Stage。
- **Base 类型**：Moveable base（轮式机器人）/ Static base（机械臂，root_joint 固定）。
- **密度**：对无质量的 link 应用默认 density；设 0 由物理引擎自动计算。
- **关节驱动**：Drive Type 选 Acceleration（与质量无关）或 Force（弹簧-阻尼）；Target Type 为 None/Position/Velocity；增益可直接填 Stiffness/Damping 或用 Natural Frequency 自动计算。教程示例：stiffness=1047.19751、damping=52.35988。默认遵循 mimic 标签，可勾选 Ignore Mimic；支持 Ctrl/Shift 多选批量编辑关节。
- **碰撞**：Collision From Visuals（无碰撞数据时由视觉网格生成）、Collider Type 为 Convex Hull 或 Convex Decomposition、Self-Collision 开关、圆柱转 Capsule 选项。
- 导入结果符合 Isaac Sim 资产结构约定，mesh 已可 instanceable；特殊字符替换为下划线。
- 导入方式：GUI、Python（`ImportConfig` + `URDFParseAndImportFile`）、Linux 下 File > Import from ROS 2 URDF Node（原生支持 XACRO）。
- 经验：移动机器人轮子用 Velocity drive、转向关节用 Position drive；力矩控制机器人驱动设 None。

来源：ext_isaacsim_asset_importer_urdf.html、import_urdf.html

## MJCF 导入

选项与 URDF 类似：Moveable/Static base、默认 density（0=引擎默认）、Visualize collision geometry、Self-collision（默认关闭，推荐保持）。示例资产 `nv_humanoid.xml`；Python 用 `MJCFCreateImportConfig` + `import_config.set_fix_base(False)`。link/joint 名特殊字符替换为下划线，下划线开头会加前缀 'a'。导入后通过 DriveAPI 调整 stiffness/damping 区分位置/速度控制。

来源：ext_isaacsim_asset_importer_mjcf.html、import_mjcf.html

## Robot Wizard（Beta）

处于 Beta，推荐用于 CAD 导入的、link/joint 较少的简单机器人。流程：Add Robot（选类型、命名、父 link）→ Prepare Files（指定保存目录）→ Robot Hierarchy（按"如何运动"组织 link 与网格父子关系）→ Add Colliders（选碰撞近似）→ Add Joints and Drives → Save Robot（指定 articulation root，可加最小环境）。生成分层文件：`configurations/<name>_base.usd`（网格层级）、`<name>_physics.usd`（刚体/碰撞/关节/驱动 sublayer）、`<name>_robot.usd`（robot schema），主文件 `<name>.usd` 含变体。限制：UI 无法手工建 collider，额外物理需直接改 USD；翻页前不落盘修改。

来源：robot_setup/robot_wizard.html

## 编辑工具

- **Merge Mesh Utility**：合并刚体内多网格并保留材质，减少 prim 数。
- **Gain Tuner**（robot_setup/ext_isaacsim_robot_setup_gain_tuner.html）：基于 `Effort = Kp(pos_des−pos_cur) + Kd(vel_des−vel_cur)`。模式判定：Position drive 需 stiffness>0；Velocity drive 需 stiffness=0；二者为 0 则 None；Mimic 由他关节驱动。支持直接编辑增益或按 Natural Frequency + Damping Ratio（在 home 位形计算）。Gains Test 提供正弦（幅值/偏移/周期/相位）与阶跃（min/max）轨迹，按关节分组（sequence）测试并对比实际位置/速度。
- **Robot Assembler**（robot_setup/assemble_robots.html）：用物理 fixed joint 组合两个 USD 资产（机械臂+夹爪、机器人+移动底座）。流程：加载资产 → 选 base/attach robot → 指定 attach point（Robot Link 或 Reference Point）→ 用旋转按钮/gizmo 对位 → 仿真验证 → 保存（可生成带 variant set 的配置）。仅时间线播放时生效；静止组合无需此工具。Python：`isaacsim.robot_setup.assembler.RobotAssembler` 的 `begin_assembly()/assemble()/finish_assemble()`。
- 检查工具：Physics Inspector、Simulation Data Visualizer。

来源：robot_setup/editing_tools.html 及上列各页

## 13 篇教程递进脉络

（robot_setup_tutorials/index.html）初级（轮式）：1 Stage Setup → 2 组装简单机器人 → 3 Articulate 基础机器人 → 4 相机与传感器 → 5 移动机器人 rig；中级（机械臂）：6 Setup Manipulator → 7 Configure Manipulator → 8 生成机器人配置文件 → 9 Pick & Place；高级：10 闭环结构 rig → 11 关节增益调优 → 12 资产优化 → 13 腿式机器人 locomotion policy rig。

- **T3 articulation 绑定**（tutorial_gui_simple_robot.html）：对车身+双轮创建 Revolute Joint（轴设 Y，调 Local Rotation 对齐），加 Angular Drive（轮速控制：Damping=1e4、Target Velocity=200 rad/s），在根 prim 加 **Articulation Root** 提升效率，再经 Tools > Robotics > Omnigraph Controllers > Joint Velocity 建控制图。注意：articulation controller 用弧度，USD 属性用度。成品对照 `mock_robot_rigged` 资产。
- **T4 相机/传感器**（tutorial_gui_camera_sensors.html）：Create > Camera 后拖到机器人 body 下随动；建议挂在中间 prim 上便于归零复位；示例前视相机 T=(-6,0,2.2)、R=(0,-80,-90)；用 Camera Inspector 预览，双 viewport 验证；方法可推广至其他传感器。
- **T5 移动机器人 rig**（rig_mobile_robot.html）：以叉车为例——分析 DOF（4 被动滚轮 revolute、1 主动升降 prismatic、2 主动后轮/转向）→ 按"父 prim 动则子网格随动"整理层级并赋刚体 → 碰撞用 convex decomposition，轮子用圆柱 collider → 建关节与驱动：升降 prismatic（Damping=10000、Stiffness=100000）、后轮角驱动（Damping=10000、Stiffness=100）、转向（Damping=100、Stiffness=100000），转向限位 ±60° → 加 articulation root。防自碰撞。
- **T7 机械臂配置**（tutorial_configure_manipulator.html）：UR10e + Robotiq 2F-140——位置迭代提至 64、速度迭代 4、sleep threshold 0.00005、stabilization threshold 0.00001；夹爪指尖加刚体材料（静/动摩擦均 1.0）；finger_joint 最大力矩 200；Gain Tuner 推荐 Nat. Freq.=0.5、Damping Ratio=1.0。提高迭代增精度但降性能；增大 max force 可能需提高仿真步频防穿透。
- **T11 关节调参**（joint_tuning.html）：位置驱动——先 damping=0 只调 stiffness，收敛后回退一个数量级作基线，再加约低一个数量级的 damping；速度驱动相反——stiffness=0 只调 damping，带载再加约 10%。目标：超调 ≤1%。工业机器人可将 stiffness 加倍近似完美跟踪，并设最大关节速度限制；有重力补偿的机器人可禁用重力；按关节组分别调再整合。
- **T10 闭环结构**：排障页建议将闭环拆为多 articulation 加约束，比单一复杂链更稳定。
- **T12 资产优化**（optimizing_asset.html）：Jetbot 示例经 Mesh Merge 从 40 FPS 提至 64 FPS；对重复部件（左右轮）启用 **Instanceable** 共享内存；重组为 Visual/Physics 分 scope 的分层结构；减少灯光与半透明材质；碰撞体简化（轮子用圆柱代 mesh）；调 contact 数量平衡精度与开销。
- **T13 腿式机器人 policy rig**（tutorial_rig_legged_robot.html）：以 H1 人形机器人为例，为外部 policy（ROS 等）部署准备资产——按 policy 环境定义文件设初始关节角（弧度转角度，如 left_hip_pitch −0.28 rad ≈ −16°，经 Joint State API 写入）；配置 stiffness/damping、velocity/effort limit、armature、friction（增益同样需 rad→deg 换算）；用 SingleArticulation API 脚本校验参数一致。参考资产 `Isaac/Samples/Rigging/H1/h1_rigged.usd`；标准 Isaac Lab 部署可由 Policy Controller 类自动完成运行时 rig。

## 资产校验

扩展 `isaacsim.asset.validation`（默认启用），Window > Asset Validator。三类规则：**PhysicsRules**（关节/质量/碰撞/articulation 配置，如"非固定关节必须有 Drive API 或 Mimic API"）、**RobotRules**（命名规范、RobotAPI、关系定义、物理属性的 layer 组织）、**SimReadyAssetRules**（材质层级规范、禁止嵌套）。

来源：robot_setup/asset_validation.html

## 排障

- **机器人爆炸/不稳**：检查碰撞网格互相穿插（尤其关节枢轴处）、关节限位被违反、articulation 链关节顺序；调 timestep、solver iterations、stiffness/damping——过高振荡、过低到不了目标，用 Gain Tuner 优化。
- **控制器问题**：并联机构夹爪（如 Robotiq）可能有不动的 link 需手工修正；差速轮摩擦过低打滑、过高行为异常。
- **导入/导出坑**：URDF 转换可能把 collider 网格混入 visual、body/joint 按字母序重排；导出器对无界限位写入 infinite 值可能导致解析失败；同名材质会被当作同一资产（忽略参数差异）。
- **闭环结构**：拆成多个 articulation 加约束更稳。

来源：robot_setup/troubleshooting.html
