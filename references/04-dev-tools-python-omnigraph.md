# 开发工具、Python 脚本与 OmniGraph

基础 URL: `https://docs.isaacsim.omniverse.nvidia.com/5.1.0/`

## 一、开发工具链

面向需要开发 Python/C++ 扩展、调试脚本、创建 OmniGraph 节点的中级用户（development_tools/index.html）。

- **VS Code**：安装 marketplace 扩展 *Isaac Sim VS Code Edition*，并在 Extension Manager 中启用 `isaacsim.code_editor.vscode`，经 `Window > VS Code` 打开预配置工作区。支持向运行中的 Isaac Sim 远程执行 Python（选中代码后在扩展面板点 Run）、代码片段、扩展/项目模板生成。`.vscode` 内三个关键文件：`launch.json`（含 "Python: Attach" 附加调试与 "(Linux) isaac-sim" 启动调试）、`settings.json`（指定解释器 `${workspaceFolder}/kit/python/bin/python3` 及分析路径）、`tasks.json`（standalone 脚本环境自动化）。（development_tools/vscode.html）
- **Jupyter Notebook**：启用 `isaacsim.code_editor.jupyter` 扩展，`Window > Jupyter Notebook` 打开 JupyterLab。两种内核：*Omniverse (Python 3)* 在运行中的实例内执行；*Isaac Sim Python 3* 跑 standalone（仅 Linux）。standalone 须先实例化 `SimulationApp` 再 import 其他模块，用 `./jupyter_notebook.sh xxx.ipynb` 启动；框架自动应用 `nest_asyncio`。限制：阻塞代码会冻结 Isaac Sim、不支持 IPython magic 与 matplotlib 绘图、回调 print 输出到终端而非 notebook。（development_tools/jupyter_notebook.html）
- **Script Editor**：Kit 内置 Python 编辑器，`Window > Script Editor` 打开；可多标签页，各标签共享同一解释器环境（变量/库互通），用于直接操作 Stage 与快速试验。（development_tools/omniverse_script_editor.html）
- **Carb 设置**：Carbonite 设置控制窗口、ROS 版本等默认行为。四种修改方式：①Script Editor 临时修改 —— `settings = carb.settings.get_settings(); settings.set("/exts/.../foo", True)`；②命令行 `./isaac-sim.sh --/exts/.../foo=True`；③改扩展 config 的 `.toml`（永久）；④改 `apps/` 下应用 `.kit` 文件（永久）。（development_tools/carb_settings.html）

## 二、Python 脚本核心概念

- **两种执行模式**：standalone 脚本（命令行执行、自动化/批量仿真）与交互式脚本（Script Editor/控制台内探索 API）。（python_scripting/python_scripting_concepts.html）
- **Core API 体系**：对原始 USD 与物理引擎 API 的机器人向封装 —— 原生 USD 建一个带物理的立方体约需 30 行，Core API 一个 `DynamicCuboid` 即可。概念层级（剧场比喻）：Application（UI 层）→ Simulation（推进时间的引擎）→ Stage（USD 容器）→ **World**（仿真上下文、时间管理，单例）→ **Scene**（资产集合）。注意：5.0 起引入重写的 **Core Experimental API**，现行 Core API 将逐步弃用，建议尽早迁移。（python_scripting/core_api_overview.html）
- **SimulationApp 与时序**：`SimulationApp` 管理应用生命周期，**所有 Omniverse 级 import 必须在其实例化之后**（依赖扩展系统先加载）；无显示环境设 `{"headless": True}`。基本骨架：实例化 → import → 循环 `simulation_app.update()` → `simulation_app.close()`。扩展可通过 `.kit` 文件 `[dependencies]` 或 `enable_extension()` 启用。

## 三、Standalone Python 环境（python.sh）

`python.sh`（Windows 为 `python.bat`）完成三件事：定位 apps 目录下的 .kit 文件、source `setup_python_env.sh` 加载扩展接口、调用打包的 Python 解释器；配置 `ISAAC_PATH`、`PYTHONPATH`、`LD_LIBRARY_PATH`、`CARB_APP_PATH` 等环境变量。官方 standalone 示例覆盖时间步进、加载 USD Stage、URDF 导入、分辨率修改、资产转换与 livestream。（python_scripting/manual_standalone_python.html，环境详情另见 python_scripting/environment_setup.html）

## 四、常用代码片段主题

- **场景搭建**（Scene Setup 片段）：刚体创建（DynamicCuboid、地面）、批量视图类 `RigidPrim`/`RigidContactView`（接触力监测）、OBJ/STL/FBX 转 USD、物理场景与重力/求解器配置、碰撞网格（凸包/凸分解）、物理查询（overlap、raycast）、MDL 材质、变换矩阵与保存 Stage。
- **通用工具**（python_scripting/util_snippets.html）：异步任务与 timeline 控制（`omni.timeline.get_timeline_interface().play/pause()`）、viewport 相机参数（`omni.kit.viewport.utility.get_active_viewport()`）、三种点/几何渲染方案 —— `UsdGeom.Points`（大量点最省）、`UsdGeom.PointInstancer`（需物理交互）、`DebugDraw`（纯可视化最快，`_debug_draw.acquire_debug_draw_interface()`）。默认渲染有最多 3 帧 in-flight，零延迟需求可用 `omni.isaac.sim.zero_delay.python.kit`。
- **机器人仿真**（python_scripting/robots_simulation.html）：`Articulation` 与批量 `ArticulationView`；四种关节控制 —— 位置目标 `set_articulation_dof_position_targets`、速度控制（先把 stiffness 置零）、力矩 `set_articulation_dof_efforts`、单 DOF `find_articulation_dof`；查询关节状态。须在仿真运行中执行，建议在 Script Editor 测试。

## 五、Core API 教程系列递进脉络

系列共 8 课，面向初学者，主线是"用控制器控制轮式机器人与机械臂并记录数据"（core_api_tutorials/index.html）：

1. **Hello World**：继承 `BaseSample`（封装世界加载、热重载）；`World` 为单例管理物理步进，`Scene` 管理 Stage 资产。关键方法：`setup_scene()`（建场）、`setup_post_load()`（物理句柄传播后赋值）、`world.add_physics_callback()`、`DynamicCuboid`、`get_world_pose()`/`get_linear_velocity()`。扩展工作流为异步回调驱动；standalone 则 `world.reset()` 后手动 `world.step(render=True)` 循环。（tutorial_core_hello_world.html）
2. **Hello Robot**：`add_reference_to_stage()` 从 Nucleus 加载 Jetbot，包成 `Robot` 类；经 `get_articulation_controller()` 取关节控制器，用 `apply_action(ArticulationAction)` 在物理回调中每步发速度指令；更高层可用 `WheeledRobot.apply_wheel_actions()`。（tutorial_core_hello_robot.html）
3. **添加控制器**：自定义 `CoolController` 继承 `BaseController`，实现 `forward()` 返回 `ArticulationAction`（独轮车模型：线/角速度 → 左右轮速）；再替换为内置 `DifferentialController` 与更高层的 `WheelBasePoseController`（直接给目标位姿）。（tutorial_core_adding_controller.html）
4. **添加机械臂**：导入 Franka（`isaacsim.robot.manipulators.examples.franka`），gripper 设初始张开关节位；用 `PickPlaceController` 完成抓取-放置。三种递进写法：直接实现 → 自定义 `FrankaPlaying(BaseTask)` 任务类 → 复用扩展内置 `PickPlace` 任务。（tutorial_core_adding_manipulator.html）
5. **多机器人**：`RobotsPlaying(BaseTask)` 统一管理 Jetbot+Franka，方法四件套 `set_up_scene()`/`get_observations()`/`pre_step()`/`post_reset()`；用 `task_event` 状态变量（0/1/2）编排：Jetbot 推方块→倒退 200 步让位→Franka 执行 PickPlace；子任务观测组合进父任务观测。（tutorial_core_adding_multiple_robots.html）
6. **多任务**：任务构造函数接受 `offset` 参数实现空间平移，多实例并行；用 `find_unique_string_name()` 与 `is_prim_path_valid()` 保证命名/prim 路径唯一；`get_params()` 暴露任务参数；应用类维护任务/控制器/机器人列表，`physics_step()` 中逐任务施加动作，`world_cleanup()` 处理热重载清理。（tutorial_core_multiple_tasks.html）

（系列后续还有 Adding Props、Data Logging 两课。）

## 六、OmniGraph 可视化编程

- **概念**：Omniverse 的图计算/可视化编程框架，连接多系统函数；Isaac Sim 中支撑 Replicator、ROS 2 bridge、传感器数据访问、控制器与外设 I/O。编辑器入口 `Window > Graph Editors > Action Graph`。（omnigraph/index.html）
- **入门教程**：用 Action Graph 控制 Jetbot。核心节点：**On Playback Tick**（仅播放时每帧发执行事件）、**Differential Controller**（线/角速度 → 双轮指令）、**Articulation Controller**（向指定机器人关节施加位置/速度/力指令，需 Constant Token + Make Array 组关节名列表）。也可用菜单 `Tools > Robotics > Omnigraph Controllers` 一键生成差速控制图（可配轮距/轮半径）。（omnigraph/omnigraph_tutorial.html）
- **Python 建图**：`og.Controller.edit()` 一次性指定图路径、evaluator，并用 keys —— `CREATE_NODES`、`CONNECT`、`SET_VALUES` —— 批量建节点/连线/赋值；支持事后 get/set 属性、增删节点；可设 "On Demand" 管线阶段按需显式求值。参考示例 `omnigraph_triggers.py`。（omnigraph/omnigraph_scripting.html）
- **自定义 Python 节点**：两个文件 —— `.ogn`（JSON：元数据、输入/输出端口、Action Graph 用 `execIn` 执行端口、`"language": "python"`）+ `.py`（同名类实现 `compute(db)`，经 database 读写输入输出，成功返回 True）。可放入现有扩展节点目录或用模板生成器建独立扩展；参考现有节点位于 `exts/isaacsim.<ext>/isaacsim/<ext>/ogn/python/nodes/`。（omnigraph/omnigraph_custom_python_nodes.html）

## 七、GUI 界面构成

- **文档结构**：UI 参考、快捷键、App Selector、Create 菜单、Replicator 菜单、Preferences、选择模式；Viewport、Stage、Property、Script Editor 等作为外部扩展另行文档化。注：5.1.0 已不再维护，修复只进新版本。（gui/index.html）
- **主界面**：菜单栏（Create/Window/Tools/Utilities/Layout 等 Isaac Sim 专属菜单）、**Viewport**（3D 资产主视图）、主工具栏（选择模式 模型/prim、移动/旋转/缩放及全局/局部坐标、吸附、播放/停止）、Browsers（资产与示例库）、**Stage 窗口**（USD 场景层级）、**Property 面板**（选中 prim 的属性编辑）。支持拖拽标签重排、面板分离为独立 OS 窗口、隐形分隔条调整大小。（gui/reference_user_interface.html）
