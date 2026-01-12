# Agent Evaluation Skills

[![GitHub](https://img.shields.io/badge/github-agent--evaluation--skills-blue)](https://github.com/feixiaoxu2022/agent-evaluation-skills)
[![License](https://img.shields.io/badge/license-MIT-green)]()

## 简介

Agent Evaluation Skills是为LLM Agent评测场景设计的**标准化知识模块体系**，提供从场景设计到失败分析的全流程方法论支持。

**核心特点**：
- **项目解耦**：skills内容与具体项目目录结构无关，可移植到任何项目
- **模块化设计**：每个skill专注一个领域，可独立使用或组合使用
- **Agent友好**：结构化内容，便于LLM解析和学习
- **完整方法论**：覆盖设计、实现、评测、分析全生命周期（7个标准skill模块）

## 配套评测框架

这套 Skills 配合 [MCP-Benchmark](https://github.com/feixiaoxu2022/mcp-benchmark) 评测框架使用：

- **Skills**（本仓库）：提供方法论、模板和最佳实践
- **MCP-Benchmark**：提供 Executor、Evaluator 等运行时组件

## 评测目标：9个核心能力

Skills体系的最终目标是支持构建能够全方位评测LLM Agent在实际落地中所必备能力的自动化评测系统：

| 能力维度 | 定义 |
|---------|------|
| **1. 多模态理解** | 处理和融合文本、图像等多模态信息 |
| **2. 复杂上下文理解** | 跨轮整合上下文并保持一致性 |
| **3. Prompt遵循** | 理解并遵循用户指令与业务规则 |
| **4. Tool Use** | 准确选择工具并构造参数 |
| **5. 任务规划与工具组合** | 任务分解与工具编排 |
| **6. 多轮对话管理与用户引导** | 结构化收集信息与清晰反馈 |
| **7. 反思与动态调整** | 基于执行结果诊断并调整策略 |
| **8. 多源信息深度融合与洞察** | 处理多源数据的一致性与权衡 |
| **9. 结合领域知识的自主规划** | 利用领域知识做出专业决策 |

## Skills索引

### <img src="https://img.shields.io/badge/-%E8%AE%BE%E8%AE%A1-blue?style=flat-square&logo=figma&logoColor=white" height="20"/> 设计阶段

| Skill | 用途 | 核心内容 |
|-------|------|---------|
| [scenario_design_sop](scenario_design_sop/) | 场景设计SOP | 五种设计方法、YAML结构、需求模板设计、能力覆盖映射 |
| [business_rules_authoring](business_rules_authoring/) | 业务规则编写 | 结构化模板、可验证性原则、规则设计模式 |

### <img src="https://img.shields.io/badge/-%E5%AE%9E%E7%8E%B0-green?style=flat-square&logo=codacy&logoColor=white" height="20"/> 实现阶段

| Skill | 用途 | 核心内容 |
|-------|------|---------|
| [tool_implementation](tool_implementation/) | MCP工具实现 | 设计原则、代码模板、参数规范、错误处理 |
| [checker_implementation](checker_implementation/) | Checker实现 | 验证策略、类型选择、rule-based优先原则 |
| [sample_authoring](sample_authoring/) | 样本合成 | 格式规范、质量标准、生成器模板、JSONL格式 |

### <img src="https://img.shields.io/badge/-%E8%AF%84%E6%B5%8B-orange?style=flat-square&logo=testinglibrary&logoColor=white" height="20"/> 评测阶段

| Skill | 用途 | 核心内容 |
|-------|------|---------|
| [evaluation_execution](evaluation_execution/) | 评测执行 | benchkit使用、命令规范、调试技巧、3次失败规则 |

### <img src="https://img.shields.io/badge/-%E5%88%86%E6%9E%90-purple?style=flat-square&logo=googleanalytics&logoColor=white" height="20"/> 分析阶段

| Skill | 用途 | 核心内容 |
|-------|------|---------|
| [failure_analysis](failure_analysis/) | 失败归因分析 | 四类归因、8步流程、三层验证、能力维度映射 |

## Skills工作流

```
┌─────────────────────────────────────────────────────────────┐
│                         设计阶段                              │
├─────────────────────────────────────────────────────────────┤
│  scenario_design_sop     ──→  unified_scenario_design.yaml  │
│  business_rules_authoring ──→  BusinessRules.md             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓ 设计完成
┌─────────────────────────────────────────────────────────────┐
│                        实现阶段                               │
├─────────────────────────────────────────────────────────────┤
│  tool_implementation     ──→  tools/*.py                    │
│  checker_implementation  ──→  checkers/*.py                 │
│  sample_authoring        ──→  samples/eval.jsonl            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓ 样本生成完成
┌─────────────────────────────────────────────────────────────┐
│                        评测阶段                               │
├─────────────────────────────────────────────────────────────┤
│  evaluation_execution    ──→  evaluation_outputs/*.json     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ↓ 发现失败案例
┌─────────────────────────────────────────────────────────────┐
│                        分析阶段                               │
├─────────────────────────────────────────────────────────────┤
│  failure_analysis        ──→  analysis/*_analysis_*.json    │
│                          ──→  改进建议和问题定位              │
└─────────────────────────────────────────────────────────────┘
```

## 如何使用

### 方式1：集成到Agent系统（需自行实现工具）

Skills设计为Agent友好的知识模块，建议通过工具调用方式访问。你需要在自己的项目中实现skill访问工具：

**实现示例**：
```python
# 示例：实现skill访问工具
def get_skill(skill_name: str) -> str:
    """读取指定skill的内容"""
    skill_path = f"skills/{skill_name}/SKILL.md"
    return read_file(skill_path)

# Agent使用
scenario_design_guide = get_skill("scenario_design_sop")
sample_authoring_guide = get_skill("sample_authoring")
```

**集成要点**：
- 每个skill的SKILL.md包含frontmatter元数据（name、description）
- 在Agent system prompt中提供skills索引
- Agent可根据任务需要动态获取相应skill内容

### 方式2：直接阅读skill文档

每个skill目录包含：
- **SKILL.md**：快速入门和核心内容
- **references/**：详细参考文档
- **templates/**：代码模板和配置模板
- **examples/**：优秀案例

### 方式3：移植到其他项目

1. 复制整个skills目录到目标项目
2. 在Agent system prompt中提供skills索引和路径约定
3. 实现文件读取或工具调用接口让Agent访问skills（参考方式1）

**目录约定注入示例**：
```
你可以通过读取以下skills获取领域知识：
- scenario_design_sop: skills/scenario_design_sop/SKILL.md
- sample_authoring: skills/sample_authoring/SKILL.md
...
```

## 贡献指南

### 添加新skill

1. 创建目录：`skills/your_skill_name/`
2. 创建SKILL.md：遵循现有格式（name、description、Overview、Quick Start）
3. 添加references/和examples/（如需要）
4. 更新本README的索引表
5. 在skills.json中注册（如有）

### 优化现有skill

1. 确保修改不破坏项目解耦原则
2. 更新相关examples/
3. 如涉及格式规范，同步更新templates/

## 许可证

本skills体系遵循项目根目录的许可证。

## 联系方式

- **Issues**：在GitHub仓库提交issue
- **讨论**：在项目discussions区讨论设计思路

---

**版本**：v1.0
**最后更新**：2026-01-12
**维护者**：Universal Scenario Framework Team
