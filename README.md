# Alpha-Sprint Skills

Alpha-Sprint 团队的 Claude Code Skill 集合。

## Skills

| Skill | 职责 | 状态 |
|-------|------|------|
| [lpm-orchestrator](./lpm-orchestrator/) | 团队调度大脑，接收模糊需求，通过 3W1H 协议对齐，输出结构化任务图谱 | ✅ 已上线 |

## 安装方法

将 `.skill` 文件放入 `~/.claude/skills/` 目录，或直接复制对应文件夹。

## 架构

每个 Skill 包含：
- `SKILL.md` — 核心指令与工作流
- `evals/` — 测试用例（可选）
- `*.skill` — 打包文件（可直接安装）
