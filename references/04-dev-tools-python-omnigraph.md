# 开发工具、Python 脚本与 OmniGraph（6.0.1）

基础 URL: `https://docs.isaacsim.omniverse.nvidia.com/6.0.1/`

## 一、开发工具链

来源: development_tools/index.html

### Isaac Sim MCP Server（6.0 新增，重点）

来源: development_tools/isaac_sim_mcp.html（详细内容在 GitHub `NVIDIA-Omniverse/kit-usd-agents/source/mcp/isaacsim_mcp`）

一个 Model Context Protocol 服务器，让 AI 编码助手通过**语义检索**获取 Isaac Sim 深度知识（扩展、代码示例、settings、开发者指引）。注意：它是**知识检索型** MCP，不直接操控仿真。

- **部署**：推荐 Docker——`./build-docker.sh` 构建（约 1.35 GB），`docker run --rm -p 9904:9904 --env-file ../.env isaacsim-mcp:latest`；本地开发用 `./setup-dev.sh` + `./run.sh`
- **前置条件**：Python 3.11–3.13（3.10 不兼容）、Poetry、Git LFS（缺失会在首次调用时静默报 `Extension data is not available`）、完整 clone（sparse checkout 不行）、NVIDIA API Key（`.env` 中 `NVIDIA_API_KEY=nvapi-...`）
- **传输**：仅 Streamable HTTP，端点 `POST http://localhost:9904/mcp`（无尾斜杠，否则 307）
- **客户端配置**：Claude Code 用 `claude mcp add isaac-sim-mcp -t http http://localhost:9904/mcp`；Cursor/Windsurf/VS Code 在配置中填 `"url": "http://localhost:9904/mcp"`
- **5 个工具**：`get_isaac_sim_instructions`、`search_isaac_sim_extensions`、`get_isaac_sim_extension_details`、`search_isaac_sim_code_examples`、`search_isaac_sim_settings`
- **嵌入后端**：云端（仅需 NVIDIA_API_KEY）或本地 NIM（需 GPU、`NGC_API_KEY`，设 `KIT_EMBEDDER_BACKEND=local` 等）

### Python Server 远程代码执行（6.0 新增）

来源: development_tools/python_server.html

扩展 `isaacsim.code_editor.python_server` 提供 TCP 服务器，允许 VS Code、**LLM 智能体**或自定义脚本向运行中的 Isaac Sim 提交 Python 代码。这是 6.0 与 MCP 配套的"执行通道"：MCP 查知识，Python Server 执行代码。

- 默认 `127.0.0.1:8226`（可经 carb settings 改）
- 协议：发送 UTF-8 源码后 **TCP 半关闭**（`write_eof()`）标记结束；返回单个 JSON：`status`/`output`/`result`/`traceback`/`ename`/`evalue`
- 命令行示例：`echo 'print("Hello")' | nc 127.0.0.1 8226`
- 支持顶层 `await`；**跨连接保持全局状态**；可选 UDP 广播 carb 日志（`carb_logs`）
- 安全：host 设为 `0.0.0.0` 时局域网内任何机器都能执行任意代码，仅限可信环境

### VS Code

来源: development_tools/vscode.html

市场扩展 **Isaac Sim VS Code Edition**：远程执行代码、浏览 Isaac Sim/Kit/USD 片段、生成扩展模板、内嵌文档。依赖两个 Kit 扩展：`isaacsim.code_editor.vscode`（菜单集成）+ `isaacsim.code_editor.python_server`（执行后端）。安装目录自带 `launch.json`（调试当前文件/attach/启动带调试的 Isaac Sim）、`settings.json`、`tasks.json`。

### Jupyter Notebook

来源: development_tools/jupyter_notebook.html

两种模式：① 交互式——扩展 `isaacsim.code_editor.jupyter` 在运行中的 Isaac Sim 内启动 JupyterLab（Window > Jupyter Notebook）；② standalone——`./jupyter_notebook.sh xxx.ipynb`（仅 Linux），自动注册 `Isaac Sim Python 3` 内核并应用 `nest_asyncio`。限制：阻塞代码会冻结应用、不支持 IPython magic 与 matplotlib、回调输出走终端。

### Script Editor 与 carb settings

来源: development_tools/omniverse_script_editor.html、development_tools/carb_settings.html

Script Editor（Window > Script Editor）为 Kit 内置多标签 Python 环境，各标签共享变量与导入。carb settings 四种修改方式：Script Editor 里 `carb.settings.get_settings().set(...)`（可配合 `extension_manager.set_extension_enabled_immediate()` 重启扩展生效）、启动参数 `./isaac-sim.sh --/exts/<ext>/<key>=value`、扩展 `.toml` 的 `[settings]`、应用 `.kit` 文件。

## 二、Python 核心概念与 API 现状

来源: python_scripting/python_scripting_concepts.html、core_api_overview.html

两种工作流：**standalone 脚本**（命令行、自动化/批处理/headless）与**交互式脚本**（Script Editor/Jupyter，探索 API、快速原型），能力等价，区别在执行上下文。

**API 现状（相对 5.1 的关键变化）**：Core API 是 USD/PhysX 之上的机器人向封装（Application/Simulation/Stage/World/Scene 概念，`DynamicCuboid` 等）。文档明确警告：5.0 引入的 **Core Experimental API** 是 Core API 的重写版（更健壮、灵活），**现有 Core API 将在后续版本弃用并移除**，官方强烈建议尽早迁移。6.0 中这一转移已落到实处——官方教程与代码片段已大面积改用 experimental API（见下）。相关的 Motion Generation 也已标注 Deprecated。

## 三、standalone 环境

来源: python_scripting/manual_standalone_python.html

`python.sh`/`python.bat` 会 source `setup_python_env.sh` 配置 `ISAAC_PATH`、`PYTHONPATH`、`LD_LIBRARY_PATH`、`CARB_APP_PATH` 后调用内置解释器。核心模式：

```python
from isaacsim import SimulationApp
simulation_app = SimulationApp({"headless": True})
# 所有 Omniverse 级 import 必须在实例化之后
simulation_app.update(); simulation_app.close()
```

扩展可通过 `.kit` 依赖或运行时 `enable_extension()` 启用；headless 下需去掉 matplotlib 等 GUI 调用。

## 四、代码片段主题

来源: python_scripting/environment_setup.html、util_snippets.html、robots_simulation.html

- **场景搭建**：刚体创建（`Cube`、`GroundPlane`、`RigidPrim`、`GeomPrim`）、批量 view 对象、接触过滤（`get_net_contact_forces()`、`get_contact_force_matrix()`）、质量（`set_masses()`/`set_densities()`）、包围盒 `compute_aabb()`、语义标注 `add_labels()`、资产转换 `AssetConverterContext`；物理场景（`PhysxScene`、`set_gravity()`）、`overlap_box()`/`overlap_sphere()`、`raycast_closest()`；材质（`OmniGlassMaterial`/`OmniPbrMaterial`）、位姿（`get/set_world_poses()`）
- **工具**：异步任务（`omni.kit.app.get_app().next_update_async()`）、相机参数、三种渲染路径（`UsdGeom.Points`/`UsdGeom.PointInstancer`/DebugDraw：`isaacsim.util.debug_draw`）、零延迟渲染帧配置
- **机器人**：使用 **experimental** 的 `isaacsim.core.experimental.prims.Articulation`——`get_dof_positions/velocities/efforts()`、`set_dof_position_targets()`/`set_dof_velocity_targets()`/`set_dof_efforts()`、`switch_dof_control_mode()`、`get_dof_indices()`、`num_dofs`/`dof_names`/`dof_paths`。这是相对 5.1（以 `isaacsim.core.api` 的 `SingleArticulation`/`ArticulationAction` 为主）的显著变化

## 五、Core API 教程脉络

来源: core_api_tutorials/index.html

顺序：Hello World → Hello Robot → Adding a Manipulator → Adding Multiple Robots → Multiple Robot Scenarios → Adding Props。**6.0 教程已重写为基于 experimental API 与回调驱动的模式**（5.1 时以 `World`+`BaseSample`+Task 体系为主）：

1. **Hello World**：分层构建物体（几何 `Cube` → 碰撞 `GeomPrim` → 刚体 `RigidPrim`），`SimulationManager.register_callback()` 注册物理回调；experimental API 返回批量 warp 数组，单对象取 `.numpy()[0]`；讲解 extension 热重载（Ctrl+S）与 standalone 双工作流
2. **Hello Robot**：Jetbot；`stage_utils.add_reference_to_stage()` 导入资产，`Articulation` 包装，回调内下发轮速
3. **Adding a Manipulator**：Franka Panda；`Franka` 类（继承 `Articulation`，含 IK 与夹爪控制）、`FrankaPickPlace.setup_scene()` + `forward()` 驱动七阶段抓放状态机
4. **Adding Multiple Robots**：Jetbot 推方块 + Franka 抓取，三阶段状态机，多机器人各自控制循环共存于同一物理仿真
5. **Multiple Robot Scenarios**：`RobotScenario` 类封装 Jetbot+Franka+Cube 及位置偏移，for 循环批量实例化并行场景，独立状态机 + 统一 reset

## 六、OmniGraph

来源: omnigraph/omnigraph_tutorial.html 等

- **教程**：JetBot 差速控制 Action Graph——On Playback Tick → Differential Controller（轮距 0.1125、轮径 0.03）→ Articulation Controller，Constant Token + Make Array 指定关节名；进阶键盘控制；也可用菜单快捷方式一键生成
- **Python 脚本化**（omnigraph_scripting.html）：`og.Controller.edit()` 配 `og.Controller.Keys` 的 `CREATE_NODES`/`SET_VALUES`/`CONNECT`；`pipeline_stage=GRAPH_PIPELINE_STAGE_ONDEMAND` 后用 `graph_handle.evaluate()` 手动求值；示例节点 `omni.graph.action.OnTick`、`omni.graph.nodes.ConstantString` 等
- **自定义 Python 节点**（omnigraph_custom_python_nodes.html）：`.ogn`（JSON 定义输入/输出）+ `.py`（静态 `compute(db)`，成功返 True）；Action Graph 需 `execIn`，Push Graph 不需要；可参考 `exts/isaacsim.<ext>/…/ogn/python/nodes/` 现有节点
- **自定义 IPC 节点（6.0 新增文档）**（omnigraph_custom_ipc_nodes.html）："薄桥"设计——节点只做序列化与传输，数据读写留给上下游节点。参考实现 `isaacsim.examples.ipc` 用 BSD TCP socket，提供 `SimpleSendSimulationClockCpp/Py`（发仿真时钟）与 `SimpleReceiveExternalStepCpp/Py`（收外部步进令牌，可实现外部 step-gating）；架构传输无关，可换 ZeroMQ/gRPC/共享内存（Python 经 `extension.toml` 的 `[python.pipapi]`，C++ 经 packman 依赖）。关键模式：继承 `BaseResetNode`（C++ 实现 `reset()`、Python 实现 `custom_reset()`，timeline 停止时关闭句柄），per-instance 状态经 `db.per_instance_state`/`db.perInstanceState<>()`；`compute()` 内只做**非阻塞** I/O 后置 `db.outputs.execOut = og.ExecutionAttributeState.ENABLED`；慢路径用工作线程 + 队列；大消息（相机帧）走流式或句柄传递。开发流程：`./repo.sh template new` 选 "Isaac Sim OmniGraph Node Extension" → 加依赖 → 写 `.ogn` → 实现 compute → `./build.sh` 重编译并重启（C++ 不支持热替换）。可与 `IsaacReadSimulationTime`、`IsaacArticulationState/Controller`、physics/RTX 传感器节点对接

## 七、GUI

来源: gui/index.html、gui/reference_user_interface.html

GUI 章节含：用户界面参考、快捷键、Create 菜单、Replicator 菜单、Preferences、选择模式；Viewport/Stage/Property/Extension Manager/Layers/Console 等由 Omniverse 上游扩展提供。主界面：Viewport（主视图）、Stage（USD 层级）、Property 面板、菜单栏（Create/Window/Tools/Utilities/Layout）、主工具栏（移动/旋转/缩放、播放控制，图标下小三角右键展开更多选项）、Browsers；窗口支持拖拽停靠、撕出为独立 OS 窗口。UI 布局本身相对 5.1 无重大变化。

## 相对 5.1 的变化小结

1. **新增 Isaac Sim MCP Server**（知识检索型，HTTP :9904/mcp，5 个语义检索工具）
2. **新增 Python Server 远程执行文档**（TCP :8226，面向 LLM 智能体，持久会话状态）
3. **Core API 明确进入弃用通道**，教程与片段全面转向 Core Experimental API（warp 批量数组、`SimulationManager` 回调、`isaacsim.core.experimental.prims.Articulation`）；Motion Generation 标记 Deprecated
4. **新增 Building Custom IPC OmniGraph Nodes 指南**（BaseResetNode 模式、非阻塞 compute、`isaacsim.examples.ipc` 参考实现）
