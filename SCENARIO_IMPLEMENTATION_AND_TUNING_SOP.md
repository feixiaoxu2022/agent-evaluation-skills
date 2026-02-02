# 场景实现与调优SOP

## 文档定位

从设计完成到评测验证的标准化操作流程。

---

## 🎯 核心原则：两阶段流程与问题修复顺序

### 两阶段划分

```
【阶段1：顶层设计】
unified_scenario_design.yaml（唯一数据源）
BusinessRules.md
data_pools/（skills、output_specifications）
         ↓
【阶段2：合成系统与评测】
工具实现 → Checker实现 → 样本生成器 → 生成样本 → 评测 → 分析
```

### ⚠️ 问题修复的唯一正确顺序

```
发现问题（评测失败/归因为样本设计问题）
    ↓
【第一步】追溯到阶段1：检查顶层设计是否有缺陷
    ↓
阶段1设计是否有问题？
    ↓
┌────是────┐         ┌────否────┐
│          │         │          │
修改阶段1   │         │   阶段2合成
文档        │         │   没按设计干活
│          │         │          │
unified_   │         │   修改阶段2
scenario_  │         │   产物：
design.yaml│         │   - 样本生成器
BusinessRu │         │   - 工具实现
les.md     │         │   - Checker实现
data_pools/│         │          │
│          │         │          │
└──────────┴─────────┴──────────┘
           ↓
重新执行阶段2相关步骤
（Step 4→5→6）
```

### 🚫 绝对禁止的错误做法

❌ **直接修改生成的样本文件**（samples/*.json）
❌ **跳过阶段1检查，直接改阶段2产物**
❌ **阶段1和阶段2同时修改但不同步**
❌ **修改后不重新生成样本就评测**

### ✅ 正确的问题定位思路

```
样本设计问题 →
  ├→ BusinessRules表述不清？        → 阶段1问题
  ├→ unified_scenario_design配置错误？→ 阶段1问题
  ├→ Skill手册指导不完善？           → 阶段1问题
  ├→ 样本生成器占位符替换错误？      → 阶段2问题
  ├→ check_list未按设计生成？        → 阶段2问题
  └→ 工具schema与设计不一致？        → 阶段2问题
```

---

## 前置条件

**阶段1设计阶段完成**：
- `unified_scenario_design.yaml`
- `BusinessRules.md`
- `data_pools/`

**目标输出**：`samples/eval.jsonl`（通过评测验证）

---

## 目录结构

```
场景根目录/
├── unified_scenario_design.yaml    # 统一配置（唯一数据源）
├── BusinessRules.md                 # 业务规则
├── data_pools/                      # 领域知识和规范
│   ├── skills/*.md
│   └── output_specifications.yaml
├── env/                             # MCP工具实现
│   └── {service_name}.py
├── checkers/                        # 检查器实现
│   └── checker.py
├── scripts/                         # 样本生成器
│   └── sample_generator/
│       └── generate_samples.py
├── samples/                         # 生成的样本
│   └── eval.jsonl
└── evaluation_outputs/              # 评测结果
    └── {timestamp}/
        ├── execution/               # Agent执行过程
        └── check/                   # Checker结果
```

---

## 实现流程

### Step 1: 工具实现
📚 参考：`.claude/skills/tool_implementation/`

**输入**：`unified_scenario_design.yaml` → `mcp_service_config.tools_description`
**输出**：`env/{service_name}.py`

### Step 2: Checker实现
📚 参考：`.claude/skills/checker_implementation/`

**输入**：`unified_scenario_design.yaml` → `user_need_templates[*].check_list`
**输出**：`checkers/checker.py`

### Step 3: 数据池验证
**输入**：`unified_scenario_design.yaml` → `environment.files`
**验证**：`data_pools/`目录完整性

### Step 4: 样本生成器实现
📚 参考：`.claude/skills/sample_authoring/`

**输入**：`unified_scenario_design.yaml`（完整配置）
**输出**：`scripts/sample_generator/generate_samples.py`

**关键逻辑**：
- 占位符替换（environment内容、check_list）
- MCP工具名添加前缀（`service__tool`）

### Step 5: 生成样本
```bash
cd /path/to/scenario/
python scripts/sample_generator/generate_samples.py
```
**输出**：`samples/eval.jsonl`

### Step 6: 运行评测
```bash
python /path/to/benchkit/run_evaluation.py \
  --scenario . \
  --model sonnet-4.5 \
  --samples samples/eval.jsonl \
  --output evaluation_outputs/
```

**评测框架路径**：
```
/Users/feixiaoxu01/Documents/agents/agent_auto_evaluation/universal_scenario_framework/mcp-benchmark/release/framework/benchkit
```

**模型配置路径**：
```
/Users/feixiaoxu01/Documents/agents/agent_auto_evaluation/universal_scenario_framework/auto_synthesis_system/benchkit/model_config.json
```

**输出结构**：
```
evaluation_outputs/{timestamp}/
  ├── execution/           # Agent对话历史和工具调用
  │   └── {SAMPLE_ID}.json
  └── check/              # Checker验证结果
      ├── cases/
      │   └── check_{SAMPLE_ID}.json
      └── summary.json    # 总体统计
```

### Step 7: 查看结果
```bash
# 总体统计
cat evaluation_outputs/latest/check/summary.json | jq

# 列出失败案例
jq -r 'select(.overall_result=="Failure") | .data_id' evaluation_outputs/latest/check/cases/*.json

# 查看单个案例
cat evaluation_outputs/latest/execution/{SAMPLE_ID}.json | jq '.conversation_history'
cat evaluation_outputs/latest/check/cases/check_{SAMPLE_ID}.json | jq '.check_details'
```

### Step 8: 失败分析
📚 参考：`.claude/skills/failure_analysis/`

**四类归因**：
1. Agent能力问题
2. 样本设计问题
3. 用户模拟器问题
4. 系统问题

**分析输出**：创建分析文档到`.claude/skills/failure_analysis/examples/`

### Step 9: 问题修复

| 根因类型 | 返回步骤 |
|---------|---------|
| 工具schema问题 | Step 1 |
| Checker逻辑问题 | Step 2 |
| 数据池问题 | Step 3 |
| 样本生成器问题 | Step 4 |
| 配置设计问题 | 修改unified_scenario_design.yaml → 重新执行后续步骤 |

修复后：**Step 5 → Step 6 → Step 7 → 验证**

---

## 关键配置说明

### unified_scenario_design.yaml 核心字段

```yaml
# MCP服务配置
mcp_service_config:
  service_name: "xxx_service"              # 服务名（用于MCP前缀）
  tools_description:                       # 工具描述（代码生成依据）
    tool_name: "描述..."

# 环境文件配置
environment:
  files:
    - path: "data_pools/xxx.yaml"
      description: "..."

# 实体定义（样本变量维度）
entities:
  entity_name:
    attributes:
      key: value_mapping                   # 用于占位符{{key}}替换

# 需求模板（生成样本的模板）
user_need_templates:
  - need_template_id: "TEMPLATE_ID"
    user_need_description: "query with {{placeholder}}"
    check_list: [...]                      # Checker验证项
    hitl_responses: {...}                  # HITL预设答案
```

---

## 质量门禁

- [ ] 通过率≥90%
- [ ] P0检查项100%通过
- [ ] 无系统级错误
- [ ] 配置一致性（工具schema ↔ tools_description，check_list ↔ 工具schema）
- [ ] 记录≥3个典型失败案例

---

## 常见问题

| 问题 | 可能原因 | 诊断方法 |
|-----|---------|---------|
| Agent行为正确但check失败 | 样本设计问题 | 检查check_list约束是否与工具schema一致 |
| 所有样本都失败在同一项 | 系统问题 | 检查工具/Checker实现 |
| 部分entity的样本失败 | 占位符替换问题 | 检查environment和check_list的占位符 |
| 工具调用验证总是失败 | MCP前缀问题 | 检查tool_name是否包含`service__`前缀 |

---

## 核心依赖Skills

- `tool_implementation/` - 工具设计和实现
- `checker_implementation/` - 检查器设计和实现
- `sample_authoring/` - 样本生成器实现
- `failure_analysis/` - 失败案例归因分析
