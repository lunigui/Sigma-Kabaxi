# Sigma-Kabaxi

结合 **Sigma 一对一辅导技能** 与 **llm-wiki-agent 自维护知识库** 的学习系统：学习资料喂给知识库，知识库自动生成"教案式"概念页，Sigma 基于教案进行掌握式教学（Bloom 2-Sigma），教学中的新发现再回写知识库——**用得越多，知识库教得越好**。

## 项目结构

```
kabaxi/
├── .agents/
│   └── skills/sigma/        # Sigma 辅导技能（已改造集成 wiki）
└── llm-wiki-agent-clean/    # llm-wiki-agent 知识库（已扩展概念 schema）
    ├── raw/                 # 存放学习资料（PDF、MD 等 20+ 格式）
    ├── wiki/                # 自动维护的知识库
    │   ├── index.md         # 目录（概念分 Teaching / Simple 两区）
    │   ├── overview.md      # 综述 + Learning Paths（依赖排序的学习路径）
    │   ├── concepts/        # 概念页（simple 与 teaching 两种）
    │   ├── sources/         # 每份资料一个摘要页
    │   └── entities/        # 人物、公司等实体页
    └── tools/               # 独立 Python 脚本（graph / lint / health）
```

## 核心设计：双概念类型

llm-wiki-agent 的概念页通过 `concept_kind` 前置字段分为两种：

| 类型 | 职责 | 结构 |
|---|---|---|
| `simple` | 维持知识图谱连通的轻量节点 | 定义 + 关联 + 来源 |
| `teaching` | Sigma 的"教案" | 定义、动机、关键要点、前置依赖、常见误解+反例、诊断题、掌握检查题（标注 rubric 维度）、练习任务 |

- **自动判断**：ingest 时按信号分类——来源篇幅、依赖链、可教素材（例子/对比/常见错误）、是否学习目标；拿不准一律先建 simple。
- **升级流程**：simple 概念被 2+ 来源深入讨论、进入学习路径、或用户显式要求时，自动升级为 teaching 并记录 `promote` 日志。
- **Learning Paths**：`wiki/overview.md` 维护按 `prerequisites` 拓扑排序的学习路径，Sigma 以此为骨架生成学习路线图。

## 使用流程

```bash
# 1. 学习资料放入知识库
cp 你的资料.pdf llm-wiki-agent-clean/raw/

# 2. 在 llm-wiki-agent-clean 目录对代理说：
#    "ingest raw/你的资料.pdf"
#    → 自动生成概念库（教学概念含完整教案）

# 3. 在项目目录运行：
#    /sigma 主题名
#    → Sigma 读取 wiki/index.md + overview.md 生成路线图

# 4. 逐概念教学：开教前自动加载教案页，
#    用页内预置的题目、误解反例、练习任务进行掌握式教学
```

## 双向闭环

```
资料 ──ingest──▶ wiki 教案 ──▶ /sigma 教学 ──▶ 新误解回写 ──▶ 更好的教案
```

- **正向**：wiki 向 Sigma 提供教案（诊断题、误解反例、掌握检查题、练习任务）
- **反向**：Sigma 把教学中发现的新误解（含实际奏效的反例）回写进教案页，log 记录 `sigma-feedback` 条目

## 致谢

- [Sigma 技能](https://skills.sh/sanyuan0704/sanyuan-skills/sigma) 来自 [sanyuan0704/sanyuan-skills](https://github.com/sanyuan0704/sanyuan-skills)，本项目修改其 `SKILL.md` 以集成 wiki 教案读取与回写。
- 知识库基于 [SamurAIGPT/llm-wiki-agent](https://github.com/SamurAIGPT/llm-wiki-agent)，本项目扩展了其概念 schema（双类型、升级流程、Learning Paths）。
