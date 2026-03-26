# Agora

> Claude Code 多 AI 角色辩论插件——通过结构化多视角讨论生成高质量提案。

## 概述

Agora 召集多个 AI 角色从不同视角审查你的提案，通过多轮辩论逐步精炼，直到全员达成共识。

- 🎭 **Momus（摩摩斯）** — 魔鬼代言人：挑战假设、发现安全风险、压力测试极端场景
- 🔮 **Cassandra（卡珊德拉）** — 细节洞察：捕捉被忽视的边界条件、逻辑漏洞和细微缺陷
- 🦉 **Athena（雅典娜）** — 战略架构：审视全局设计、可扩展性和长期影响
- ⚒️ **Hephaestus（赫菲斯托斯）** — 执行可行性：评估可实现性、资源和时间表风险

> *"Agora 汇聚四个不可替代的视角——批评者的眼光、先知的预言、战略家的判断、匠人的双手。"*

讨论自动进行直到所有角色达成共识（最多 N 轮），全过程写入文件供审阅。

## 安装

### 从插件市场安装

在 Claude Code 中运行：

```bash
# 1. 添加市场源
/plugin marketplace add ClaudeWorksHub/claude-plugins

# 2. 安装插件
/plugin install agora@claudeworkshub

# 3. 激活
/reload-plugins
```

更新到最新版：

```bash
/plugin marketplace update claudeworkshub
/plugin install agora@claudeworkshub
/reload-plugins
```

### 本地开发

将 `agora/` 目录复制到 `~/.claude/plugins/`：

```bash
cp -r agora ~/.claude/plugins/
```

或在项目的 `.claude/settings.local.json` 中配置插件路径。

## 使用方法

### 基本用法

```
/agora:proposal 设计一个用户认证系统
```

### 指定角色

```
/agora:proposal --roles athena,momus 设计缓存系统
```

### 排除角色

```
/agora:proposal --exclude hephaestus 设计通知系统
```

### 限制轮数

```
/agora:proposal --max-rounds 5 设计通知系统
```

### 交互模式

```
/agora:proposal --interactive 设计缓存系统
```

每轮 agent 反馈后暂停，让你补充意见、约束条件或方向指引，然后再进入下一轮。

### 中断恢复

```
/agora:proposal --resume proposals/proposal-20260326-152000.md
```

支持从中断点继续讨论，自动检测已完成的轮次和角色状态。

### 组合使用

```
/agora:proposal --roles cassandra,athena --max-rounds 8 --interactive 设计 API 网关
```

## 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--roles` | 全部 4 个角色 | 逗号分隔的角色名 |
| `--exclude` | 无 | 排除指定角色（与 `--roles` 互斥） |
| `--max-rounds` | 10 | 最大讨论轮数（1-20） |
| `--interactive` | 关闭 | 每轮后暂停等待用户输入 |
| `--resume` | 无 | 从中断的讨论文件恢复 |

### 角色名称

| 名称 | 视角 |
|------|------|
| `momus` | 魔鬼代言人 |
| `cassandra` | 细节与边界条件 |
| `athena` | 战略与架构 |
| `hephaestus` | 执行与可行性 |

## 输出

每次运行在项目的 `proposals/` 目录下生成两个文件：

| 文件 | 内容 |
|------|------|
| `proposal-<timestamp>.md` | 完整讨论过程 |
| `proposal-<timestamp>-final.md` | 最终提案 |

## 工作原理

```
阶段 0：解析参数，用户确认
           ↓
阶段 1：生成初始提案
           ↓
阶段 2：独立评估（各角色并行探索代码、生成开发计划）
           ↓
阶段 3：多轮辩论（最多 N 轮）
        ┌─→ 各角色审查提案（并行调用）
        │        ↓
        │   [交互模式] 用户补充意见
        │        ↓
        │   Orchestrator 逐条回应异议
        │        ↓
        │   检查收敛条件
        └── 未收敛则继续
           ↓
阶段 4：质量门自检 → 输出最终提案
```

### 收敛规则

- **第 1-3 轮**：鼓励深度审查，每个角色从至少 3 个维度评估
- **第 4 轮起**：所有活跃角色连续 2 轮无异议即收敛（结构化规则，orchestrator 不可覆盖）
- **交叉验证**：收敛前从讨论文件中提取 Status 行，与解析结果比对
- **硬限制**：达到 `--max-rounds` 时以当前最佳提案结束，列出未解决问题

### 成本控制

- **复杂度分级**：Light（2 角色/5 轮）、Medium（3 角色/8 轮）、Heavy（4 角色/10 轮）自动选择
- **休眠机制**：连续 2 轮无异议且提案未修改的角色自动休眠，节省 token
- **上下文管理**：Agent 每轮新建（无累积），orchestrator 仅保留最近 2 轮讨论
- **成本追踪**：最终输出展示 agent 调用次数和休眠节省的轮次

## Git 处理

插件不修改 `.gitignore`。建议：

- 讨论文件可能很大——考虑将 `proposals/proposal-*.md` 加入 `.gitignore`
- 最终提案（`*-final.md`）值得作为设计决策文档提交
- 或全部保留以获得完整审计追踪

## 讨论语言

讨论跟随任务描述的语言——中文任务产生中文讨论，英文任务产生英文讨论。角色名使用希腊神话人物（Momus / Cassandra / Athena / Hephaestus）。

## 许可证

MIT
