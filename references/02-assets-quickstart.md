# 资产库与快速入门

## 一、官方 USD 资产体系

Isaac Sim 提供经策划的 USD 资产库，通过 Content Browser（Window > Browsers）访问，根路径由 `persistent.isaac.asset_root.default` 配置，支持本地磁盘或 Nucleus 服务器存储。资产需先下载到本地/Nucleus，首次加载较慢（机器人可能需数分钟），可启用 `omni.activity.ui` 扩展监控加载进度。共八大类：机器人、相机/深度传感器、非视觉传感器、道具、环境、特色资产、第三方 SimReady、神经体渲染。

来源: assets/usd_assets_overview.html

### 机器人资产（按类别，路径相对 `/Isaac/Robots/`）

- **轮式/AMR**: Nova Carter、Carter v1/v2、Jetbot、Leatherback（NVIDIA）、iRobot Create3、Turtlebot3 Burger、Clearpath Jackal/Dingo、Agilex Limo、Idealworks iw_hub、叉车 ForkliftB/C
- **机械臂**: Franka Panda/FR3、UR3/UR5/UR10、KUKA KR210、UFactory xArm、Kinova Gen3、Yaskawa Motoman NEX10、SO100
- **人形**: Unitree H1/G1、Agility Digit v4、IHMC Valkyrie、RobotEra STAR1
- **四足**: Unitree Go1/Go2/A1/B2、ANYbotics ANYmal、Boston Dynamics Spot（含 spot_with_arm 移动操作变体）
- **全向**: NVIDIA Kaya、Fraunhofer O3dyn
- **无人机**: Crazyflie、Quadcopter、NASA Ingenuity

来源: assets/usd_assets_robots.html

### 传感器资产

- **相机/深度**（`/Isaac/Sensors/`）: Leopard Imaging Hawk 立体相机、Owl 鱼眼相机；Sensing SG2/SG3/SG5/SG8 系列；SICK Inspector83x；Intel RealSense D455（`intel/RealSense/rsd455.usd`）；Orbbec Gemini 2/335/335L、Femto Mega；Stereolabs ZED X / ZED X Mini。来源: assets/usd_assets_camera_depth_sensors.html
- **非视觉**: RTX Lidar 配置（HESAI XT32、Ouster OS0/OS1/OS2、SICK microScan3/multiScan/picoScan/TiM781、SLAMTEC RPLIDAR S2E、ZVISION ML-30s+ 固态雷达等，分认证/非认证）；触觉传感器 Tashan TS-F-A（输出 11 维特征通道）。关键概念：传感器功能与物理网格解耦，可无几何体挂载，Gizmo 可在视口开关。来源: assets/usd_assets_nonvisual_sensors.html

### 道具与环境

- **道具**: People/Characters 人物角色（警察、医生、建筑工人等，带重定向骨骼，可经材质 Albedo Tint 换装色）；AprilTag MDL 材质（可配 Tag ID、马赛克贴图）。来源: assets/usd_assets_props.html
- **环境**: Simple Grid（`default_environment.usd`/`gridroom_black.usd`/`gridroom_curved.usd`）、Simple Room、Warehouse 四种（`warehouse.usd` 至 `full_warehouse.usd`）、Hospital、Office、JetRacer Track、小型仓库数字孪生（`small_warehouse_digital_twin.usd`），均可从 Create > Environments 菜单创建。来源: assets/usd_assets_environments.html

### 特色资产：Nova Carter

基于 NVIDIA Nova Orin 传感器计算架构的 AMR 参考平台（Isaac AMR / Isaac ROS 参考机型，可从 Segway Robotics 购买）。传感器套件：4×Hawk 立体相机 + 4×Owl 鱼眼 + IMU + 2×2D RPLidar + 1×XT-32 3D Lidar。资产：基础版 `/Isaac/Robots/NVIDIA/NovaCarter/nova_carter.usd`（无传感器，含 Base/Fully Merged/No_Internals/Skirt_only 配置、No_Physics/Physics_Base 物理、None/All_Sensors 传感器等 variant）；ROS 2 版 `/Isaac/Samples/ROS2/Robots/Nova_Carter_ROS.usd`（含传感器与 ROS 2 Action Graph）。

来源: assets/usd_assets_featured.html、assets/nova_carter_landing_page.html

### 第三方 SimReady 与 NuRec

- **SimReady**：针对仿真优化、物理/语义就绪的通用 USD 资产标准。第三方来源：Lightwheel SimReady 商店（开/闭源环境道具）、X-Humanoid ArtVIP（大规模关节化物体与场景）、Synthesis 资产包（合成数据）、SpatialVerse 室内数据集、MolmoSpaces；经商业平台或 HuggingFace/GitHub 获取。来源: assets/usd_assets_third_party.html
- **NuRec（神经重建）**：由真实图像训练的 3D Gaussian 神经体场景，以 USDZ 导入 Isaac Sim。要求 RTX（光线追踪）渲染、需配置物理（导航用同步模式）。示例数据集（HuggingFace）：Voyager Cafe、Galileo Lab、Wormhole、ZH Lounge；生成工具为开源 3DGRUT。来源: assets/usd_assets_nurec.html

## 二、机器人资产目录结构规范

推荐三阶段组织：

1. **Source（源）**：导入的原始资产保持不变以便无缝重导入，分基础结构、零件、材质文件；
2. **Transformation（转换）**：面向仿真优化——扁平化层级、合并网格、简化材质、将刚体整理为列表；
3. **Features（特性）**：物理、传感器、控制图、ROS 集成等作为独立轻量 USD 层。

最终 `asset.usd` 通过组合弧集成：**Sublayer**（核心结构、Robot Schema）、**Payload**（按需加载降内存）、**Reference**（物理挂到 defaultPrim）、**Variant**（免复制切换配置）。目录按 source 文件夹 + feature 文件夹 + 根级最终资产组织。

来源: robot_setup/asset_structure.html

## 三、快速入门要点

- **入口**（introduction/quickstart_index.html）：两个核心教程 + 教程参考表；新手建议完成 13 篇 Robot Setup 系列；支持 GUI 与 Python 灵活互换。
- **基础使用**（introduction/quickstart_isaacsim.html）：以 GUI / Extension 脚本 / Standalone Python 三种方式演示同一任务——导航界面、加地面与光源、添加物体、赋予 "Rigid Body with Colliders Preset"、Play 运行。关键概念：物理与碰撞属性可分别添加（可只碰撞不受重力）。
- **机器人入门**（introduction/quickstart_isaacsim_robot.html）：用 Franka Panda 7 自由度机械臂（Create > Robots），用 Physics Inspector 查看关节上下限/默认位、调刚度阻尼；三种控制：OmniGraph 关节位置控制器（GUI）、Script Editor 调 Articulation API（扩展）、显式 stepping 循环（standalone）。
- **教程体系**（introduction/tutorial_list.html）：十大系列——Core API、Robot Setup、导入/导出（URDF/CAD）、Isaac Lab（RL）、ROS 2、合成数据、传感器仿真、运动生成、OmniGraph、附加资源。

## 四、示例体系

- **交互式示例**：模拟器内 Window > Examples > Robotics Examples，含八类：Sensors（LIDAR/IMU/接触）、Input Devices（手柄/键盘）、Manipulation（Franka RMPFlow Follow Target、码垛、装箱）、Multi-Robot（Robo Party/Robo Factory）、General（Hello World）、Policy（Spot/人形/Franka 策略部署）、Cortex（UR10 码垛）、Import Robots（URDF 导入）。
- **Standalone 示例**：位于 `<isaac_sim_root>/standalone_examples`，命令行运行，如 `./python.sh standalone_examples/api/isaacsim.simulation_app/hello_world.py`；完整清单在 Python API 文档专页。

来源: introduction/examples.html、introduction/menu_examples.html、introduction/standalone_examples_list.html

## 五、参考架构与工作流对比

**参考架构五个任务组**：几何创作（CAD/SimReady，统一转 OpenUSD，默认单位米）→ 资产导入（URDF、CAD Converter、OBJ/FBX/glTF）→ 场景搭建（物理场景、碰撞、关节驱动、传感器）→ 数字孪生交互（GUI/Standalone/Extension/OmniGraph）→ 用例（Replicator 合成数据、SIL 软件在环、经 ROS 2 的 HIL 硬件在环、OSMO 云端 CI/CD）。与 Isaac Lab（强化学习）、ROS 2、Isaac ROS（加速感知）集成。

来源: introduction/reference_architecture.html

**工作流对比**（introduction/workflows.html）：

| 工作流 | 特点 | 适用 |
|---|---|---|
| GUI | 可视化直观工具 | 建场景、装配机器人、挂传感器、启动 ROS 桥 |
| Extension | 应用内异步非阻塞运行，热重载免重启，物理/渲染自动步进 | 代码片段试验、自定义 UI、实时响应应用 |
| Standalone Python | 手动控制物理/渲染步进时机，支持 headless | 大规模 RL 训练、系统化场景生成、同步随机化、ROS 消息速率控制 |

Extension 与 Standalone 共用同一套 API，核心差异在事件循环控制权；三者可组合：GUI 搭场景存 USD，再由 standalone 脚本程序化加载修改。（Jupyter 属 standalone 式交互变体，快速入门以脚本形式覆盖同一 API。）
