# isaacsim-qa — Isaac Sim 5.1.0 问答 Skill

一个 [Claude Code](https://claude.com/claude-code) 个人 skill：回答 NVIDIA Isaac Sim 5.1.0 使用过程中的各类问题（安装、URDF 导入、机器人搭建、ROS 2、Isaac Lab、Replicator 合成数据、传感器、物理、性能等）。

## 工作方式

两级查询策略：

1. **本地优先**：先按主题速查表读取 `references/` 下的中文文档摘要（约 200 页官方文档的整理稿，保留了关键命令、API 名、参数值与原文出处 URL）；
2. **网络兜底**：本地摘要不足以解决时（缺具体步骤、API 细节、报错匹配），自动带版本号与当前年份进行网络搜索，优先官方文档、NVIDIA 开发者论坛与 isaac-sim GitHub。

答案末尾会标注来源（本地摘要 / 官方原页 / 论坛）。

## 安装

```bash
git clone https://github.com/<your-username>/isaacsim-qa-skill.git
mkdir -p ~/.claude/skills
cp -r isaacsim-qa-skill ~/.claude/skills/isaacsim-qa
```

安装后在 Claude Code 中提问 Isaac Sim 相关问题会自动触发，也可用 `/isaacsim-qa <问题>` 手动调用。

## 目录结构

```
isaacsim-qa/
├── SKILL.md          # skill 主文件（查询流程与主题索引）
└── references/       # Isaac Sim 5.1.0 官方文档中文摘要（11 个文件）
    ├── README.md     # 摘要总索引与关键速查
    ├── 01-overview-installation.md
    ├── ...
    └── 10-utilities-usd-reference.md
```

## 来源与许可

- skill 本身（SKILL.md、README、仓库结构）以 [MIT License](LICENSE) 发布。
- `references/` 内容整理自 [NVIDIA Isaac Sim 5.1.0 官方文档](https://docs.isaacsim.omniverse.nvidia.com/5.1.0/index.html)（整理于 2026-07），各条目均附原文链接；文档版权归 NVIDIA 所有，摘要仅为学习用途，**不在 MIT 许可范围内**（详见 LICENSE 附注）。
- 注意：官方已标注 Isaac Sim 5.1.0 停止支持（EOL），缺陷修复与新功能只进后续版本。
