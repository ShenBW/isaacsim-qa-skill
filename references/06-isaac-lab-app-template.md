# Isaac Lab 与应用模板（6.0.1）

## Isaac Lab 定位

Isaac Lab 是 Isaac Sim 的**官方机器人学习框架**，提供强化学习、模仿学习的 API 与示例，采用模块化、配置驱动的环境定义体系。它取代已弃用的 IsaacGymEnvs、OmniIsaacGymEnvs 与 Orbit（均有迁移指南）。特性：预置学习环境、多 RL 库支持、外设示教接入、面向 sim-to-real 的自定义执行器模型。仓库 https://github.com/isaac-sim/IsaacLab。相对 5.1 定位无实质变化。
来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/isaac_lab_tutorials/index.html

## 策略部署流程

在 Isaac Lab 训练并导出策略，再在 Isaac Sim 中经控制器类部署。示例：Unitree H1（`Isaac-Velocity-Flat-H1-v0`，Robotics Examples > POLICY > Humanoid）、Spot（`Isaac-Velocity-Flat-Spot-v0`，POLICY > Quadruped）。

- 训练：`./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task Isaac-Velocity-Flat-H1-v0 --headless`；用 `play.py` 导出到 `exported/`，配置在 `logs/rsl_rl/<task>/<time>/params/`（`env.yaml` 含 dt、初始关节、刚度/阻尼、观测/动作缩放；`agent.yaml` 为网络结构）。
- 控制器类关键方法：`load_policy()`、`initialize()`（力矩模式、增益、最大力/速度）、`_compute_observation()`（H1 为 69 维张量）、`_compute_action()`、`forward()`（按 decimation 执行）。
- 力矩机器人可用执行器网络（如 ANYmal 的 `anydrive_3_lstm_jit.pt`，`compute_torques()` 将位置输出转力矩）。
- 排错要点：用 `dof_names` 核对关节顺序；核对默认关节位置（H1 踝关节错误会"太空步"）；`prim.get_dof_gains()` / `get_dof_max_efforts()` 验证增益；物理步长须与 `dt` 匹配（0.005s=200Hz，60Hz 会崩溃）；用 `set_dof_position_targets()` 而非 `set_joint_position()`；观测/动作按 YAML scale 缩放。

来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/isaac_lab_tutorials/tutorial_policy_deployment.html

## 经 ROS 2 跑 RL 策略

以 H1 为例：Isaac Sim 负责物理与传感器，外部 ROS 2 节点跑策略。三个 ActionGraph：ROS_Imu 发布 `/imu`（IMU 路径 `/h1/pelvis/Imu_Sensor`，取消 Read Gravity）；ROS_Joint_States 发布 `/joint_states`、订阅 `/joint_command` 并接 Articulation Controller；ROS_Clock 发布 `/clock`（Read Simulation Time 开 Reset on Stop）。物理 200Hz。资产：`Samples/Rigging/H1/h1_rigged.usd`、`Samples/ROS2/Robots/h1_ROS.usd`、`Samples/ROS2/Scenario/h1_ros_locomotion_policy_tutorial.usd`。命令：

```bash
ros2 launch h1_fullbody_controller h1_fullbody_controller.launch.py
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

注意：策略节点须**先于仿真启动**否则机器人瘫倒；线/角速度超 0.75 越限；策略不支持后退。

来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/ros2_tutorials/tutorial_ros2_rl_controller.html

## Cloner

`isaacsim.core.cloner` 提供 `Cloner`（手动给位姿）与 `GridCloner(spacing=3)`（自动网格布局）；流程 `generate_paths("/World/Cube", 4)` → `clone(source_prim_path=..., prim_paths=..., positions=...)`。用 `isaacsim.core.experimental.prims.XformPrim("/World/Cube_.*")` 的 `get/set_world_poses()` 做向量化访问（6.0 使用 experimental API 是与 5.1 的写法差异点）。`replicate_physics=True`（配 `base_env_path`/`root_path`）直接在 PhysX 复制以加速，但不支持运行时改材质/摩擦等形状属性。`copy_from_source=False`（默认）用 USD Inherits，源改动会传播；True 为独立副本。

来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/isaac_lab_tutorials/tutorial_cloner.html

## Instanceable 资产

利用 USD Scenegraph Instancing 让多份机器人克隆共享网格数据，降低大规模 RL 场景内存。要求每个 Mesh 有父级 Xform 承接 instanceable 标记。途径：URDF/MJCF 导入器勾选 "Create Instanceable Asset" 并指定 "Instanceable USD Path"；或用工具函数 `create_parent_xforms(asset_usd_path, source_prim_path, save_as_path)` 与 `convert_asset_instanceable(asset_usd_path, source_prim_path, save_as_path, create_xforms=True)`。限制：不能修改实例化 prim 后代属性；转换会移除 Mesh 上的 USD Relationship，需改挂到父 Xform。

来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/isaac_lab_tutorials/tutorial_instanceable_assets.html

## 排障

- 安装：ModuleNotFoundError → 确认 Python 3.9+、在 Isaac Lab 仓库 `pip install -e .`、核对 CUDA 版本。
- 性能：减少环境数/场景复杂度、减小 batch、降低传感器数量与分辨率。
- 环境配置：核对 URDF/USD 路径、任务配置格式、reward/observation 定义。
- 部署：确认观测空间与训练一致、仿真参数对齐训练设置、策略文件格式受支持。

来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/isaac_lab_tutorials/troubleshooting.html

## Kit App Template

**相对 5.1 的重大变化**：独立的 Application Template（`isaacsim-app-template`）在 **5.1.0 为最终版本**；自 6.0.0 起其能力已并入开源 Isaac Sim 仓库（https://github.com/isaac-sim/IsaacSim），不再单独维护。原用途是基于扩展注册表构建轻量自定义应用、提供镜像官方二进制配置的模板。历史实现见 https://github.com/isaac-sim/isaacsim-app-template；现有用户应迁移到开源 IsaacSim 仓库。

来源: https://docs.isaacsim.omniverse.nvidia.com/6.0.1/app_template/index.html
