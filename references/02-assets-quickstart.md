# 资产库与快速入门（6.0.1）

## 一、USD 资产体系

**总览**：资产通过 Window > Browsers 的 Content Browser 访问，分七大类：机器人、传感器（相机/深度、非视觉）、道具、环境、Featured、第三方 USD、NuRec 神经渲染。资产根路径由 `persistent.isaac.asset_root.default` 设置控制；机器人首次加载可能需数分钟，大型环境可达 10 分钟以上，可用 Omniverse Activity UI 扩展监控加载进度。
来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/assets/usd_assets_overview.html

**机器人清单**（`Isaac Sim/Robots/<厂商>/<型号>/*.usd`）：

- 轮式/AMR：iRobot Create3、Turtlebot3、NVIDIA（Jetbot、Carter v1/v2、Nova Carter、Leatherback、Robomaker）、Forklift B/C、Idealworks iwhub、Fraunhofer Evobot、Clearpath Jackal/Dingo、AgileX Limo
- 机械臂：Universal Robots（ur3/3e/5/5e/10/10e/16e/20/30）、Franka（Panda/FR3/Emika）、Yaskawa Motoman NEX 系列、Fanuc CRX10IAL 等、KUKA KR210_L150、Kinova Jaco2/Gen3、Kawasaki RS、Flexiv Rizon4、Techman TM12、UFactory xArm/uf850/lite6、Rethink Sawyer、Unitree Z1
- 灵巧手/夹爪：Shadow Hand、Allegro Hand、Robotiq 2F-85/2F-140/Hand-E、Schunk SVH/EGK/EGU、Psyonic Ability Hand、Unitree Dex3/Dex5、Yahboom Dofbot、RobotStudio SO 系列
- **6.0 新变化**：新增平行目录 `Isaac Sim/Robot_Multiphysics`（镜像 Robots 布局），资产仅含中性物理原语，供 **Newton 物理后端**加载，不含 PhysX 专属 schema（5.1 无此目录）；带验证图标的资产已确认行为与原版一致，未标记者仍在验证中。

来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/assets/usd_assets_robots.html

**传感器资产**（相机/深度 + 非视觉两类）：

- 相机：Leopard Imaging Hawk/Owl（`Sensors/LeopardImaging/Hawk/hawk_v1.1_nominal.usd` 等）、Sensing SG2/SG3/SG5/SG8 系列、SICK Inspector83x/InspectorP61x
- 深度：RealSense D455/D457/**D555（新增）**（`Sensors/Realsense/D555/rsd555.usd`）、Orbbec Gemini 2/335/335L、Femto Mega、Luxonis OAK4-D/OAK4-D Wide/OAK-D Pro PoE/ToF、SICK safeVisionary2
- 非视觉：RTX Lidar（NVIDIA 示例、HESAI XT32、Ouster OS0/OS1/OS2/VLS128、SICK LMS/LRS/MRS/multiScan/picoScan/nanoScan/microScan/TiM781、SLAMTEC RPLIDAR S2E、ZVISION ML-30s+/ML-Xs）；RTX Radar：TI IWRL6432AOP（57–63.5 GHz mmWave）；触觉：Tashan TS-F-A（11 维输出）；IMU：ST ASM330LHH、LSM6DSV。
- **6.0 变化**：SICK 等多型号改用 USD variant，从单一父资产提供多配置。创建入口 Create > Sensors > 类型 > 厂商。

来源: assets/usd_assets_camera_depth_sensors.html、assets/usd_assets_nonvisual_sensors.html

**道具**：People/Characters 角色（警察、医生、建筑工人等，可经 Looks 的 Albedo Color Tint 换装色）、April Tags 可配置基准标记材质。
来源: assets/usd_assets_props.html

**环境**：Flat/Black/Curved Grid（`default_environment.usd` 等）、Simple Room、四种仓库（`warehouse.usd`→`full_warehouse.usd`）、Hospital、Office、JetRacer Track、小仓库数字孪生（`small_warehouse_digital_twin.usd`）。
来源: assets/usd_assets_environments.html

**Featured**：Nova Carter——基于 Nova Orin 传感器/算力架构的 AMR 参考平台（Isaac AMR/Isaac ROS），首次加载需数分钟。
来源: assets/usd_assets_featured.html

**第三方资产**：Lightwheel SimReady 商店、Synthesis Asset Pack、imagine.io 道具、X-Humanoid ArtVIP、SpatialVerse（InteriorAgent/InteriorGS）、MolmoSpaces、XGrid 扫描转仿真、Extwin。
来源: assets/usd_assets_third_party.html

**NuRec（6.0 新增）**：以 3D Gaussian Splats/辐射场表示真实场景，RTX 原生与多边形几何混合渲染。Hugging Face 提供 Andoria、Wormhole、Cafe、Galileo 等 `.usdz` 场景；示例演示在 NuRec 场景中运行 Nova Carter 导航。注意：开启 DLSS 帧生成渲染粒子场可能出现伪影，需关闭。配套 **NuRec Rendering Utilities**（`isaacsim.replicator.nurec_utils`，实验性）：自动识别粒子场/体积/SPG-PPISP 路径并配置渲染；渲染脚本 `standalone_examples/nurec/nurec_render.py` 支持姿态模式（TUM 轨迹）与关键帧模式，输出图像+`manifest.json`；`test_nurec_render_vs_gt.py` 计算 PSNR/SSIM 并生成对比报告。
来源: assets/usd_assets_nurec.html、assets/nurec_utils.html

## 二、资产结构规范

推荐多文件分层：`base.usda`（层级）、`geometries.usd`（网格）、`instances.usda`（视觉/碰撞组合）、`materials.usda`（材质），物理分引擎隔离——`physics.usda`（通用）、`physx.usda`、`mujoco.usda`（6.0 强调多引擎防冲突）；`robot.usda` 应用 Isaac Robot Schema，独立维护运动学关系。夹爪、控制图、ROS 集成等功能作为轻量层经 payload/variant 叠加，避免复制核心资产；ASCII 组合利于版本控制。改几何进 `geometries.usd`，选碰撞体进 `instances.usda`，引擎调参进对应引擎文件。
来源: robot_setup/asset_structure.html

## 三、快速入门

- 索引：两个核心快速教程（Isaac Sim 基础、机器人基础）+ 13 部分机器人装配教程系列；强调"GUI 能做的都可以用 Python 做"。
- Isaac Sim 快速入门：首次启动约 5–10 分钟；新建场景→加地面→加 Distant Light→加视觉立方体→分别添加 Rigid Body 与 Collider（可只碰撞不受重力）→变换工具调整；GUI/扩展/独立 Python 三种方式并行讲解，扩展支持热重载。
- 机器人快速入门：以 Franka Panda（9 DOF）为例，Create > Robots 加载→Physics Inspector 查看/修改关节限位、刚度阻尼→Tools > Robotics > OmniGraph Controllers > Joint Position 生成关节位置控制器→仿真运行中在 Property 面板改指令→Action Graph 查看控制图。

来源: introduction/quickstart_index.html、quickstart_isaacsim.html、quickstart_isaacsim_robot.html

## 四、示例体系

- 交互式示例：Window > Examples > Robotics Examples，左侧按类别浏览；分类含 Sensors、Input Devices、Manipulation（UR10/Franka 路径规划、装箱）、Multi-Robot（Robo Party）、General（Hello World）、Policy（四足/人形/Franka RL 策略）、Cortex 与机器人导入、Tutorials。
- 独立示例：`<isaac_sim_root>/standalone_examples`，用 `./python.sh`（Linux）/`python.bat`（Windows）运行，如 `standalone_examples/api/isaacsim.simulation_app/hello_world.py`；完整清单在 Python API 文档的 standalone_examples_list。
- 教程总表按 9 大系列组织：Core API、机器人设置、导入/导出（URDF/USD/CAD）、Isaac Lab（RL）、ROS 2、合成数据生成、传感器（RTX/PhysX）、运动生成、OmniGraph。

来源: introduction/examples.html、menu_examples.html、standalone_examples_list.html、tutorial_list.html

## 五、参考架构与工作流

**参考架构**五个顺序任务组：几何创作（CAD→OpenUSD）→资产导入（URDF/MJCF/CAD Converter）→场景设置（物理、碰撞、关节、合成传感器）→数字孪生交互（GUI/Python/扩展/OmniGraph）→用例落地。支撑工作流：Replicator 合成数据生成、软件在环（导航/操纵/RL）、硬件在环（ROS 2 连接实机）、云端 CI/CD。核心原则：一切资产须先转为 OpenUSD。

**工作流对比**：GUI——可视化搭建世界、装配机器人、OmniGraph 编程；Extension——异步运行、热重载、适合交互式 GUI 与实时应用；Standalone Python——手动控制物理/渲染步进、支持 headless，适合大规模 RL 训练、批量场景生成、确定性 ROS 消息发布。Extension 与 Standalone 共用同一 API，差异在步进时序控制；常见做法是 GUI 搭场景存 USD，再由脚本程序化修改迭代。

来源: introduction/reference_architecture.html、workflows.html

## 相对 5.1 的主要变化汇总

1. 新增 Robot_Multiphysics 资产目录与 Newton 后端支持、资产验证图标机制；
2. 新增 NuRec 神经重建资产类别及 nurec_utils 渲染/评估工具；
3. 传感器新增 RealSense D555、Luxonis OAK4 系列、Tashan 触觉、TI mmWave 雷达、ST IMU 扩展，SICK 系列改用 USD variant 管理；
4. 资产结构规范强化多物理引擎分层（physx.usda/mujoco.usda 分离）。
