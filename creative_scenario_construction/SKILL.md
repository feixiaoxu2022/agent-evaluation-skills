---
name: creative-scenario-construction
description: 构建创意类（小说/短剧/文案等）自动评测场景的完整方法论。基于小说写作(novel_writing_alchemist)场景的多轮设计迭代实战经验沉淀，覆盖从顶层设计到评测分析的全流程，以具体版本迭代中的真实案例展示关键设计决策的思考过程。
---

# 创意类场景自动评测构建指南

> 基于 novel_writing_alchemist（小说写作）场景的完整构建经验总结。
> 适用于：小说、短剧、文案、漫改、剧本等一切需要Agent进行创意内容生产的评测场景。

---

## 一、创意类场景 vs 业务规则类场景的本质差异

在开始之前，必须理解创意类场景与传统业务规则场景（如请假申请、广告投放）的根本不同，这决定了设计方法论的分化：

| 维度 | 业务规则类场景 | 创意类场景 |
|-----|-------------|-----------|
| **规则性质** | 精确、可枚举的业务规则 | 模糊的创作标准 + 少量硬性规则 |
| **验证方式** | final_state精确比对 | 分层验证：硬规则(rule-based) + 软质量(LLM judge) |
| **输出形式** | 结构化数据(JSON/状态更新) | 大量非结构化文本(章节/剧本/角色卡) |
| **Agent能力侧重** | Prompt遵循、Tool Use、信息融合 | 领域知识自主规划、任务规划与工具组合、长程一致性 |
| **设计核心** | 规则体系 + 实体池 + 员工画像叉乘 | 领域知识体系 + 创作流程 + 质量评价体系 |
| **SOP差异** | 规则先行，YAML中直接装载全部设计 | 自然语言规则先行，YAML做结构化映射 |

**关键经验**：创意类场景不适合把所有内容都塞进 unified_scenario_design.yaml 中用结构化方式表达。**应该先写自然语言版本的 BusinessRules.md 和领域知识文档，再在 YAML 中做结构化映射**。这是在构建小说场景过程中总结出的SOP分化原则——业务规则类场景适合在YAML中直接结构化定义规则，而创意类场景的"规则"本质上是创作方法论和质量标准，用自然语言表达远比结构化YAML更直观高效。

---

## 二、整体构建流程（6个阶段）

```
阶段1: 领域知识体系构建
    ↓
阶段2: 创作Agent流程设计 (BusinessRules.md + Skill文件体系)
    ↓
阶段3: 评价体系设计 (能力维度分类 + Checklist + LLM Judge Criteria)
    ↓
阶段4: 统一配置 & 样本合成 (unified_scenario_design.yaml + query_pools + samples)
    ↓
阶段5: 评测执行 & 多模型对比
    ↓
阶段6: 结果分析 & 迭代优化 → 反馈到阶段3重新迭代
```

**核心理念：评测驱动的迭代闭环**。不要试图一次设计完美的评价体系，先跑一轮评测，根据实际产出的问题针对性优化。小说场景的checklist从rev_001的39项迭代到rev_004的43项，每一项新增都来源于具体模型的真实失败案例。

---

## 阶段1: 领域知识体系构建

### 1.1 目标
为Agent提供完成创作任务所需的领域专业知识，这些知识以Skill文件的形式存放在 `data_pools/skills/` 中，Agent在创作过程中需要主动读取和遵循。

### 1.2 实施方案

**输出物**：`data_pools/skills/*.md` — 一组领域知识文档

**小说场景的实际Skill体系（8个文件）**：

| 文件 | 适用阶段 | 内容 |
|------|---------|------|
| RECIPE_KNOWLEDGE.md | 配方选择 | X轴36种剧情模式 × Y轴12种网文标签 × 3种反应强度 |
| CHARACTER_DESIGN_GUIDE.md | 写作准备 | 人物特质设计、动机、隐秘恐惧、配角体系 |
| CHARACTER_NAMING_GUIDE.md | 写作准备 | 按风格(现代/古风/玄幻/科幻)的命名规范 |
| OUTLINE_DESIGN_GUIDE.md | 写作准备 | 三层大纲：故事梗概 > 章节梗概 > 场景序列 |
| WRITING_TECHNIQUE_GUIDE.md | 章节创作 | 9种写作技巧：场景、对话、叙事、情感等 |
| ROMANCE_WRITING_GUIDE.md | 章节创作 | 感情线写作：含蓄 > 直白，行动 > 语言 |
| SHORT_STORY_GUIDE.md | 章节创作(短篇) | 超短篇(6500-10000字)专项指南 |
| CONSISTENCY_MANAGEMENT_GUIDE.md | 章节创作 | 设定一致性管理：道具/数字/人物特征追踪 |

### 1.3 Skill设计的核心原则

**原则1: 从真实质量短板中产生Skill，不凭空创造**

Skill文件不应该是凭空想象的"最佳实践手册"，而应该来源于Agent实际创作中暴露的质量问题。

> **实际案例**：`ROMANCE_WRITING_GUIDE.md` 的诞生源于用户对AI写作效果的真实不满——"只会说你爱我我爱你，很没意思"。而 `CONSISTENCY_MANAGEMENT_GUIDE.md` 则来自rev_003阶段发现多个模型在长文本写作中出现道具数量前后矛盾（如枕巾数量从40→50→1→4→2的混乱跳变）。

**原则2: Skill提供方法论框架，不规定执行细节**

过于详细的Skill会导致Agent产出千篇一律的内容，或过度询问用户。

> **实际案例**：测试创作时用户反馈"你问的问题太多了，交互成本太高"——Agent把Skill中每个环节的问题都问了一遍。Skill应该是"怎么想"的方法论，不是"问什么"的问题清单。

**原则3: "必须阅读" vs "参考"——指令强度直接影响Agent行为**

小说场景design_v1阶段做过A/B测试：V1版本的system prompt中用"参考"（弱指令），V2版本改为"必须先阅读"（强指令）。

> **发现**：弱指令词"参考"导致模型在token成本压力下跳过阅读补充文档。改为"必须先阅读"后阅读率显著提升。但这带来了一个设计张力——**写"必须"会让所有模型都通过，失去区分度；写"建议"才能测出模型是否有自主阅读的意识**。解决方案是分层：system prompt中对核心Skill写"必须"，对辅助Skill在用户模拟器的对话中"顺口提一嘴"，测试Agent是否能捕捉用户暗示。

**原则4: 区分"用户需求多样性"和"Agent创作Skill"**

分析人工构造数据时发现：有些prompt是在制造"用户会怎么表达需求"的多样性，有些是完整的创作SOP——这两类要严格区分：
- 前者 → 进入 query_pools / user_need_templates
- 后者 → 进入 data_pools/skills/

---

## 阶段2: 创作Agent流程设计

### 2.1 目标
设计Agent执行创作任务的标准流程，以 BusinessRules.md（Agent的system prompt）形式呈现。

### 2.2 BusinessRules.md 核心结构

```
1. 角色定位 — Agent是什么角色
2. 创作流程 — 分几个阶段完成，每阶段做什么
3. 知识库引用 — 每个阶段需要读取哪些Skill文件
4. 输出规范 — 每个阶段的输出文件格式和存放位置
5. 交互规范 — 何时需要与用户确认（HITL点）
6. 硬性规则 — 必须遵守的约束（如字数规则、集数规则等）
```

**小说场景的4阶段SOP**：
```
Phase 1: 理解需求（收集用户创作意图，分类灵感明确度）
Phase 2: 生成配方（X-Y轴配方匹配，HITL：用户选择配方）
Phase 3: 写作准备（人物设计 + 三层大纲，HITL：用户确认）
Phase 4: 章节创作（按大纲逐章写作，维护writing_log）
```

### 2.3 关键设计决策

**HITL（Human-in-the-Loop）点的设计**

创意类场景必须在关键节点让用户确认，否则跑偏了整个任务报废。但不能过度确认——"交互成本太高"的教训。小说场景最终收敛到2个HITL点：配方选择 + 写作准备确认。

**"建议" vs "必须"的措辞——服务于能力区分度**

> **实际案例**：BusinessRules中"大规模创作建议分批写入文件"用的是"建议"而非"必须"。Claude执行50集任务用了405轮（每轮只写1个文件），而其他模型使用了`write_files`批量工具。这是**有意设计**：如果写"必须"，所有模型都会遵循，失去了区分Tool Use能力的测试价值。

**线性流程 > 分支流程**

> **迭代教训**：最初设计了"情节驱动/人物驱动/混合式"三种创作模式让用户选择 → 发现太复杂（验证分支爆炸） → **最终回退到线性的 配方→人物→大纲→创作 流程**。线性流程更适合自动评测，因为验证点明确。复杂度应该靠Skill文件补充，而不是靠流程分支。

---

## 阶段3: 评价体系设计（最核心的阶段）

这是创意类场景与业务规则场景差异最大的地方，也是迭代次数最多的环节。

### 3.1 三层架构：维度 → 子类 → 检查项

小说场景经过v1.1到v2.7共10个版本的迭代，最终形成了三层能力分类体系：

```
Capability Dimension (维度)
  └── Subcategory (子类)
        └── Check Item (检查项)
```

**5大能力维度（v2.7）**：

| 维度 | 说明 | 检查类型 | 检查项数 |
|------|------|---------|---------|
| format_compliance | JSON语法、结构、字段命名、数据类型 | 纯规则检查 | 4 |
| business_rule_compliance | SOP流程、交付完整性、Skill读取、约束遵守 | 纯规则检查 | ~23 |
| memory_management | 创作日志创建与使用（写-读-写闭环） | 规则检查 | 2 |
| interaction_completeness | HITL确认节点调用 | 规则检查 | 0-2 |
| content_quality | 内容创作质量（两层评估） | LLM语义检查 | ~14 |

### 3.2 内容质量的两层评估体系

这是创意类评价的核心设计。底线问题必须一票否决，优秀特征综合评分：

**Basic Tier（及格线，通过/不通过）**：

包含15个子类，按检查对象组织为4组：

```
▌人物体系检查（5项，有递进关系）
  递进逻辑：设计完整性 → 规划覆盖 → 执行落地 → 表现准确 → 前后一致

  characters.json ──┬──→ outline.json ──→ chapters/*.md
  (Plan: 设计)      │    (Plan: 规划)     (Execute: 写作)

  [1] main_character_consistency:     主角不能中途丢失/被替换
  [2] character_presence_in_outline:  设计的角色必须在大纲中规划出场
  [3] character_presence_in_chapters: 规划的角色必须在正文中实际出现
  [4] character_design_adherence:     正文表现必须符合设计文档的人设 ★ v3新增
  [5] character_trait_consistency:    章节之间人物特征保持一致

▌故事整体性检查（3项）
  [6] theme_consistency:         主题一致性（防止主题漂移）
  [7] logical_contradiction:     逻辑硬伤（前后矛盾的事实错误）
  [8] late_stage_digression:     后期跑偏（章节后期内容偏离主线）★ rev_003→004拆分

▌情感与调性检查（2项）
  [9]  emotional_delivery_match: 情感交付匹配度（配方方向 vs 实际内容）
  [10] narrative_tone_match:     叙事调性匹配（文笔风格 vs 题材类型）★ rev_004新增

▌叙事质量检查（3项）
  [11] plot_progression:         情节推进（需有事件密度）★ v2.5新增
  [12] full_narrative_content:   完整叙事文本（正文 vs 大纲概述体）
  [13] language_purity:          语言纯净性（避免无意义多语言混用）
  [14] repeated_endings:         反复结局（避免多次出现"全文完"标记）★ rev_003新增
```

**Advanced Tier（优秀度，1-5分评分）**：

```
核心通用（priority: high）
  - appeal_strength_advanced:    吸引力强度（冲突、悬念、开篇抓人）
  - emotional_impact:            情感冲击力
  - emotional_progression:       情感层次递进 ★ v2.3新增
  - genre_fit:                   题材契合度（精致度，做得好不好）

扩展通用（priority: medium）
  - pacing_rationality_advanced: 剧情节奏合理性
  - hook_design:                 钩子设计
  - novelty_appeal:              立意新颖性
  - dialogue_quality:            台词质量

题材特定（priority: low）
  - sweetness_quality:           甜度质量（仅甜宠/言情类）
  - humor_effect:                搞笑效果（仅喜剧/轻松向）
```

**质量等级计算**：
- **不合格**: basic层有任何失败 → 0-60分
- **合格**: basic全过 + advanced通过率<70% → 60-70分
- **优秀**: basic全过 + advanced通过率≥70% → 70分以上

### 3.3 维度迭代的真实案例：每一项新增都来自具体的模型失败

这是创意类场景评价体系建设中最重要的方法论——**评测驱动的迭代闭环**。以下是taxonomy从v2.4到v2.7的真实迭代轨迹：

**v2.4 → 三层情感验证体系**

最初只有一个模糊的"情感质量"检查。v2.4将其拆分为三层：
```
用户说"我要甜文" → emotional_tendency_consistency (业务规则层：配方匹配用户意图)
配方是"甜爽向"  → emotional_delivery_match (基础质量层：内容匹配配方方向)
内容是甜爽的    → genre_fit (进阶质量层：做得精致不精致)
```
> **设计思考**：用户要甜文结果写成虐心的——这不是"质量不够好"，是"方向错了"，属于基础质量的一票否决项，不能放在advanced层扣分了事。

**v2.5 → 新增 plot_progression（情节推进）**

> **触发案例**：评测发现某些模型的章节中"两个人站在那里聊感情聊了3000字，没有任何事件发生"——技术上不算逻辑错误，情感方向也对，但就是没有故事在发生。现有检查项都抓不住这个问题。

**rev_003 → 新增 repeated_endings（反复结局）和 late_stage_digression（后期跑偏）**

> **触发案例**：kimi-k2.5和其他模型在长文本写作中出现"写作疲劳"症状——多次输出"全文完""全书完""后记"标记后又继续写，或者在后期章节中跑偏到作者感言、读者互动、番外预告等与正文无关的内容。

**rev_004 → 拆分 late_stage_digression + 新增 narrative_tone_match**

这个版本的两个变更都值得详细展开：

**拆分案例**：rev_003的late_stage_digression混合了两类检查：
1. workspace白名单文件检查（规则检查）——Agent在workspace中创建了白名单外的无关文件
2. chapters后期章节内容跑偏检查（LLM语义检查）——章节内容跑偏

> **问题**：阶段一（规则检查）fail后阶段二（语义检查）不执行，导致归因模糊——到底是"创建了多余文件"还是"内容跑偏了"？语义检查的能力也被浪费了。
>
> **解决**：拆分为两个独立检查项。workspace_file_compliance（business_rule_compliance维度）+ late_stage_digression（content_quality维度），各自独立pass/fail。

**新增案例**：narrative_tone_match的发现过程：

> **触发案例**：kimi-k2.5的`NW_CLEAR_MEDIUM_BRAINY_ACTION_001`样本——配方是烧脑规则怪谈，但从第5章起文笔急剧转向言情调——大量"泪水夺眶而出""两人的手握在一起""直到永远"。
>
> **为什么现有检查项抓不住？** emotional_delivery_match检查的是情感倾向**方向**（爽/虐/烧脑），可能判正确（因为故事确实是烧脑悬疑方向）；genre_fit是advanced层的精致度检查。但这里的问题是**底线级别**的调性错误——烧脑悬疑用言情笔法写，就像恐怖片用喜剧配乐。
>
> **与现有检查项的精确区分**：
> - emotional_delivery_match: 检查情感倾向**方向**（写没写对）
> - narrative_tone_match: 检查文笔**画风**（像不像这个题材）
> - genre_fit (advanced): 检查题材**精致度**（做得好不好）

这个例子完美展示了创意类评价体系的精炼过程：从一个模糊的"质量"概念，逐步分化为方向×调性×精致度三个正交维度。

### 3.4 Checklist设计

Checklist是检查项的具体集合，分为两大类：

**Rule-based检查项（可精确验证）**：
- 文件格式检查（JSON schema验证）
- 字数范围检查（动态注入±10%容差）
- 必需元素存在性检查（4个交付文件独立检查）
- 跨文件一致性检查（角色卡↔大纲↔章节）
- Skill文件读取验证（9个Skill的read_file调用检查）
- HITL交互完成性检查（配方选择 + 写作准备确认）
- Workspace白名单文件检查

**Semantic检查项（LLM Judge评分）**：
- 内容质量各子维度评分
- 需要外部criteria文件定义具体评分标准

**关于output_completeness的教训——100%通过率=零区分度**：

> **实际案例**：初始设计中只有一个"最终交付物完整性"检查，验证`workspace/chapters/`目录是否存在。结果所有模型100%通过——只要Agent执行了`create_directory("chapters")`就通过了，完全无法区分"创建了目录但没写内容"与"完整交付"。
>
> **修复方案**：拆分为4个独立的文件存在性检查（creative_intent.json、characters.json、outline.json、chapters/），通过率从100%降到了真实水平（60-80%）。
>
> **教训**：粗粒度检查≈无效检查。检查的粒度必须足够细，才能产生有意义的区分度。

### 3.5 LLM Judge Criteria设计

**这是最容易反复、最耗时间的环节。**

**核心原则**：
1. **参考人类标注数据，不要让AI自造例子** — "你不要自己构造例子，给你这个文档作为参考的最重要原因就是这里边有实际的例子"
2. **Criteria外部化为单独YAML文件** — 方便版本迭代，不耦合在checklist中
3. **明确检查边界** — 每个检查项必须明确"只检查什么"和"不检查什么"

**Criteria YAML结构要点**（以人物设计遵循度为例）：

```yaml
llm_judge_criteria: |
  **【重要】本检查项的范围约束**：
  - ✅ 只检查：角色"是什么样的人"——性格特质、核心动机、行为模式是否符合characters.json
  - ✅ 这是检查Plan阶段（characters.json）vs Execute阶段（chapters/*.md）的一致性
  - ❌ 不检查：章节之间的人物性格一致性（那是character_trait_consistency的范围）
  - ❌ 不检查：剧情揭示时机/叙事节奏问题——这些是"故事怎么讲"的问题

  **不合格（matched: false）：实际表现与设计严重不符**
  - 性格特质不符：characters.json定义"沉默寡言" → chapters中表现为话痨
  - 核心动机偏离：characters.json定义动机"复仇" → chapters中完全不提复仇

  **合格（matched: true）：实际表现符合设计**
  - 允许在设计基础上的合理发挥，只要不违背核心设定

  **示例**（来源：Claude Opus 4.5 成功案例）：
  ✅ 设计"沉默寡言、外冷内热" → 实际：话少，但暴雨中默默买红丝巾
  ❌ 设计"沉默寡言" → 实际各章节中话痨、热情外向
```

**Criteria设计的四个关键点**：
1. **范围约束前置**：开头必须明确"只检查什么"和"不检查什么"，防止检查项职责重叠
2. **正反例并举**：不合格和合格的判断标准都要给，最好用真实评测中的案例
3. **边界情况处理**：明确易混淆情况的归属（如"这是character_trait_consistency的范围"）
4. **输出格式统一**：basic层统一用`{"matched": true/false, "reason": "..."}`

### 3.6 情感验证的特殊设计——从硬编码到语义检查的演进

> **rev_001**：最初使用`entity_attribute_equals`规则检查，将tone（如"甜爽"）硬编码映射到reaction_strength（如↗）。
>
> **rev_002**：发现映射不准确——3/13样本的期望值有争议。原因：tone→reaction_strength的映射无法覆盖Y-axis tags对反应强度的影响。比如"甜宠"配上"虐心"Y轴标签，最终的情感方向不一定是↗。
>
> **解决**：移除硬编码的entity_attribute_equals检查，改由emotional_delivery_match的LLM语义检查覆盖。**教训：当"规则"包含主观判断成分时，不要硬编码为规则检查，应该用语义检查。**

---

## 阶段4: 检查项与场景输入的解耦（架构关键决策）

### 4.1 为什么解耦？

这是小说场景迭代过程中最重要的架构决策之一。

**问题背景**：最初checklist定义嵌入在`unified_scenario_design.yaml`中。每次修改检查项都需要重新生成样本，且存在多版本同步问题（design_v1和design_v2各维护一份checklist）。

**认识到两个正交维度**：
- **场景输入版本**（v1/v2）：管理query、system prompt、user_simulator_prompt、hitl_responses
- **检查方案版本**（rev_001→004）：管理检查项定义、LLM judge criteria

两者应该独立演进。

### 4.2 解耦后的目录结构

```
check_definitions/                 # 检查项定义（独立于场景输入版本）
├── common_check_list.yaml         # 37项通用检查（所有模板共用）
├── template_checks/               # 各模板特有检查项（1-4项/模板）
│   ├── NW_CLEAR_SHORT_SWEET.yaml
│   └── ...
├── judge_criteria/                # LLM评判标准
│   ├── content_quality_basic.yaml  # v1.6
│   └── emotional_delivery.yaml     # v1.0
└── check_revisions/               # 版本管理
    ├── REVISION_LOG.yaml          # 版本日志（记录每个revision的变更和评测批次映射）
    ├── rev_001/                   # 39 checks, 初始版
    ├── rev_002/                   # 38 checks, 移除错误的硬编码映射
    ├── rev_003/                   # 41 checks, 新增反复结局+后期跑偏
    └── rev_004/                   # 43 checks, 拆分+叙事调性 ← 当前活跃
```

### 4.3 解耦带来的工作流优化

**修改检查项时**：
1. 编辑`check_definitions/`下的文件
2. 运行导出：`python main.py --export-check-revision rev_NNN`
3. 对已有执行结果跑recheck（无需重新执行Agent）

**修改场景输入时**：
1. 编辑`unified_scenario_design.yaml`中的query/prompt
2. 重新生成样本：`python main.py --output samples/eval_dsv2.jsonl`
3. 样本生成器自动从`check_definitions/`合并检查项

这意味着**检查方案可以高频迭代而不需要重跑Agent执行**，极大降低了迭代成本。

---

## 阶段5: 统一配置 & 样本合成

### 5.1 YAML核心结构

```yaml
scenario_name: "novel_writing_alchemist"
domain: creative_content

mcp_service_config:
  service_name: "novel_writing_service"
  tools: [read_file, write_file, write_files, list_directory,
          create_directory, bash, request_human_review]

environment:
  workspace_files:              # 12个文件，分必需(5)和可选(7)
    - path: "skills/SKILL.md"   # Skill文件自动扫描data_pools/目录

user_need_templates:            # 10个模板，覆盖4个变异维度
  # 明确度: vague(1个) / clear(8个) / ip_adaptation(1个)
  # 篇幅: short(2个) / medium(8个)
  # 调性: sweet/angsty/suspense/brainy_action/adventure/heroine/sweet_drama/neutral
```

### 5.2 模板设计的变异维度

小说场景通过10个模板覆盖4个变异维度的组合：

| 维度 | 变异值 | 测试目标 |
|------|--------|---------|
| **灵感明确度** | vague / clear / ip_adaptation | Agent处理模糊vs清晰需求的能力 |
| **篇幅** | short (15k-40k字) / medium (80k-250k字) | 短文vs长文的规划和一致性管理 |
| **调性** | sweet / angsty / suspense / brainy_action / adventure / heroine / sweet_drama / neutral | 不同题材的创作质量 |
| **特殊需求** | IP改编 / 大女主 / 非恋爱主线 | 特定约束下的创作能力 |

> **设计决策**：design_v2砍掉了ultra_short和long/ultra_long篇幅，聚焦short和medium。原因：ultra_short测试不出长程一致性，ultra_long太耗资源且样本太少无法做统计。每个模板只生成1个样本，总共10个——这是**成本控制**的务实选择。

### 5.3 用户模拟器的分层设计

用户模拟器prompt采用三层优先级叠加：

```
query-specific prompt → template-level prompt → common behavior (always appended)
```

**common_user_simulator_behavior** 是所有模板共用的用户交互模式：
- 配方选择阶段：用户随意选一个配方 + "顺口提醒"Agent阅读CHARACTER_DESIGN_GUIDE等
- 写作准备确认阶段：用户确认 + "顺口提醒"Agent阅读WRITING_TECHNIQUE_GUIDE等

> **设计思考**：这里的"顺口提醒"是故意设计的。system prompt中没有强制要求读取某些Skill文件，而是通过用户模拟器的对话暗示。测试的是Agent是否能捕捉用户对话中的隐含需求并主动行动。

### 5.4 样本生成器的设计模式

样本生成器（`scripts/sample_generator/main.py`）实现了几个值得复用的设计模式：

**模式1: 环境文件自动扫描**

不维护手动文件列表，而是递归扫描`data_pools/`和`judge_criteria/`目录。新增Skill文件自动被包含在样本中。文本文件as-is，二进制文件base64编码。

**模式2: 构建时校验**

在生成样本时就校验checklist的合法性：
- dimension_id必须存在于capability taxonomy中
- subcategory_id必须属于指定的dimension
- content_quality检查必须指定quality_tier
- params按sample_format_spec.json验证（必需字段、类型、路径前缀、约束结构）

> **好处**：在生成阶段就捕获配置错误，而不是在评测运行时才发现。

**模式3: 动态字数注入**

当query中指定了字数（如"30000字"或"6500-10000字"），生成器解析数值并注入±10%容差范围到checklist的validation_rules中，避免硬编码字数期望值。

**模式4: 双输出模式**

```bash
# 模式1: 生成完整评测样本
python main.py --output samples/eval_dsv2.jsonl

# 模式2: 仅导出检查方案（用于recheck已有执行结果）
python main.py --export-check-revision check_revisions/rev_004
```

---

## 阶段6: 评测执行与结果分析

### 6.1 执行配置要点

创意类任务耗时极长，需要特殊的超时配置：
- 单个API调用超时：600s
- 最大轮数：200轮（长篇可能需要400+轮）

**Agent是否使用批量写入工具是一个测试点**：提供`write_files`工具但不强制使用，观察模型是否自主选择更高效的工具。

### 6.2 控制变量分析法——从评测数据中提取多维度洞察

解耦架构最大的价值不仅是降低迭代成本，更在于**构造了一个可控实验框架**。系统中存在三个独立变量：

```
变量1: 场景设计版本 (design_version) — dsv1(弱指令"参考") / dsv2(强指令"必须阅读")
变量2: 检查方案版本 (check_revision) — rev_001 / rev_002 / rev_003 / rev_004
变量3: 被测模型 (model)              — Claude / Gemini / Ernie / kimi / EB5 / ...
```

**每次只变动一个变量、固定其余两个，就能提取不同维度的洞察**：

#### 洞察角度1: 设计进步验证——固定model + checklist，变动design_version

**问题**：dsv1→dsv2的设计变更（更强的指令措辞、更明确的阶段检查点）是否真的提升了Agent表现？

**方法**：取同一模型（如Claude Opus 4.5）在dsv1和dsv2下的执行结果，用相同的checklist revision（如rev_003）评分，对比6个重叠样本ID上的通过率变化。

```
固定: model=claude-opus-4-5, checklist=rev_003
对比: dsv1 vs dsv2
观察: 同一样本(如NW_CLEAR_MEDIUM_SWEET_001)的通过率是否提升
```

> **小说场景实例**：dsv1→dsv2最核心的变更是将Skill读取指令从"参考"改为"必须先阅读"。如果设计确实有效，预期在skill_reading相关检查项（9项）上会看到显著的通过率提升，而在content_quality相关检查项上可能没有明显变化（因为内容质量更多取决于模型本身能力而非指令措辞）。

#### 洞察角度2: 模型能力画像——固定design_version + checklist，变动model

**问题**：在相同的任务设计和评价标准下，各模型的能力长短板是什么？

**方法**：取相同design+checklist下的所有模型结果，按维度横向对比。

```
固定: design=dsv1, checklist=rev_003
对比: Claude vs Gemini vs Ernie vs kimi vs EB5
观察: 各维度通过率的差异模式
```

> **小说场景实例**：5模型在dsv1/rev_003下的对比揭示了鲜明的能力画像：
> - **Claude Opus 4.5 (89.7%)**：全面领先，但字数控制(35.7%)和逻辑一致性(35.7%)仍是短板
> - **kimi-k2.5 (71.3%)**：后期跑偏(0%)和语言纯净性(50%)是独特短板
> - **Ernie 5.0 (76.3%)**：人物一致性(0%)和创作日志(42.9%)暴露长程记忆缺陷
> - **EB5 (58.9%)**：最低通过率，格式合规和业务规则遵循均弱
>
> 更深层的洞察：**所有模型都在字数控制(7-36%)和逻辑矛盾(7-36%)上表现差**——这不是某个模型的问题，而是当前LLM的共性短板（长程状态追踪能力不足）。

#### 洞察角度3: 检查方案有效性——固定model + design_version，变动checklist

**问题**：新增的检查项是否真的带来了更准确的评价？会不会只是增加了噪音？

**方法**：取同一模型+设计版本的执行结果，分别用rev_001和rev_003评分，对比新增检查项的区分度。

```
固定: model=claude-opus-4-5, design=dsv1
对比: rev_001(39项) vs rev_003(41项)
观察: 新增的2项(repeated_endings, late_stage_digression)是否在不同模型间产生差异
```

> **小说场景实例**：rev_001→rev_002移除了emotional_tendency_consistency（硬编码映射），原因是在3/13样本上期望值有争议。通过对比rev_001和rev_002的评分结果，可以验证：(a)被移除的检查项确实导致了false negative（错误地判定样本失败），(b)移除后不影响其他维度的评价准确性。这就是**检查方案自身质量的验证方法**。

#### 洞察角度4: 难度标定——固定model + checklist，变动template类型

**问题**：哪些模板/题材/篇幅对模型更难？难度曲线是否合理？

**方法**：对同一模型的不同模板结果分组对比（按篇幅、调性、明确度分组）。

```
固定: model=claude-opus-4-5, checklist=rev_003
分组: short vs medium / sweet vs angsty vs suspense / vague vs clear
观察: 各组的通过率差异 → 难度标定
```

> **应用**：如果发现所有模型在某个模板上都100%通过，说明该模板难度不足；如果所有模型都0%，说明难度过高或设计有问题。理想的难度分布应该让不同能力水平的模型产生**梯度化的通过率差异**。

#### 洞察角度5: 检查项区分力——固定design_version + checklist，跨model统计单项

**问题**：哪些检查项真正有区分度？哪些是"无效检查"？

**方法**：统计每个检查项在所有模型上的通过率分布。

```
固定: design=dsv1, checklist=rev_003
统计: 每个检查项在5个模型上的通过率
判断: 全100%→无效, 全0%→可能过难或设计问题, 方差大→高区分度
```

> **小说场景实例**：output_completeness在所有模型上100%通过——这直接导致了将其拆分为4个细粒度检查项（见3.4节）。反过来，late_stage_digression在不同模型间呈现0%-93%的巨大差异——说明这是一个高区分度检查项。

#### 洞察角度6: 指令-行为因果链——固定model + checklist，对比design中的特定变更

**问题**：某个具体的设计变更（如在user simulator中加入"顺口提醒Agent读ROMANCE_WRITING_GUIDE"）是否真的导致了Agent行为变化？

**方法**：精细对比dsv1和dsv2在某一个特定检查项上的差异，追溯到设计中具体改了什么。

```
固定: model=claude-opus-4-5, checklist=rev_003
对比: dsv1 vs dsv2
聚焦: ROMANCE_WRITING_GUIDE reading 这一个检查项的通过率变化
追溯: dsv2相对dsv1在user_simulator_prompt中增加了什么
```

> **价值**：这是最精细的因果分析——能回答"我改了这句话，模型行为到底变没变"。如果改了指令措辞但通过率没变化，说明指令变更无效，需要换策略。

#### 总结：控制变量分析矩阵

| 洞察目标 | 固定变量 | 变动变量 | 回答的问题 |
|---------|---------|---------|-----------|
| 设计进步验证 | model + checklist | design_version | 设计迭代是否真的有效？ |
| 模型能力画像 | design + checklist | model | 各模型长短板是什么？ |
| 检查方案有效性 | model + design | checklist_revision | 新检查项是否有价值？ |
| 难度标定 | model + checklist | template类型 | 样本难度分布是否合理？ |
| 检查项区分力 | design + checklist | 跨model统计 | 哪些检查项值得保留？ |
| 指令-行为因果链 | model + checklist | design中的特定变更 | 具体改动是否生效？ |

**前提条件**：这套分析法的基础是解耦架构——场景输入和检查方案的独立版本管理，使得"固定A变动B"成为可能。如果checklist嵌入在样本中不可分离，就无法做到"相同执行结果、不同检查标准"的对比。

### 6.3 归因分析方法论

**强制要求 One-by-One 分析**：
> 不能只给汇总表。每种归因类别都必须有至少一个详细的case分析作为支撑。

**四类归因（与主框架一致）**：
1. **Agent能力问题** — 模型确实做不到
2. **样本设计问题** — 配置错误、规则模糊、checklist不合理
3. **用户模拟器问题** — prompt设计问题或执行偏差
4. **系统问题** — 工具Bug、Checker逻辑错误

**创意类场景特有的深度分析**：

以`ernie_vs_claude_character_consistency_analysis.md`为例——Ernie 5.0在人物设定一致性上0%通过率vs Claude 62.5%。深度分析发现了4个核心能力差距维度：

| 能力维度 | Ernie 5.0 | Claude Opus 4.5 |
|---------|-----------|-----------------|
| 长程一致性管理 | 关键设定在章节间出现互斥版本 | 道具数量递增清晰(17对→23对) |
| 因果链管理 | 动机在"自救/控局/献祭"间随意切换 | 伏笔提前埋设并回收 |
| 规则约束意识 | "失语者"又能说话、"盲女"又能看见 | 世界观规则自洽 |
| 心理过渡处理 | 从"持刀恐惧"直接跳到"亲密表白" | 每步都有内心独白支撑 |

> **根因洞察**：Ernie 5.0更像"逐章生成"模式——生成每章时几乎不回顾前文，导致大量不一致。Claude具备明显的全局规划能力。这不仅是通过率差异，而是**长程推理能力**和**全局状态管理**的根本性差异。

### 6.4 迭代优先级

```
P0: 配置错误修复（如output_completeness 100%通过率问题）→ 重新生成样本
P1: 检查方案迭代（新增/调整检查项）→ 对已有结果recheck（无需重跑Agent）
P2: Skill文件补强（针对质量短板）→ 重新评测全部样本
P3: Query Pool扩展 → 增加新样本
```

> **关键区分**：P1利用了检查方案解耦机制——只需recheck，不需重跑Agent。这是解耦架构带来的直接收益。

---

## 三、Checker实现要点

### 3.1 架构

```
Checker
├── checker.py              — 入口，样本解析和流程控制
├── checker_execute.py      — 5种检查类型的执行实现
│   ├── entity_attribute_equals  — 实体属性精确匹配
│   ├── json_schema              — JSON格式校验
│   ├── tool_called_with_params  — 工具调用验证（部分匹配）
│   ├── semantic_check           — LLM语义判断
│   └── file_whitelist_check     — 文件白名单校验
└── checker_score.py        — 维度聚合和质量等级计算
```

### 3.2 关键实现细节

**LLM Judge Criteria引用**：
- Criteria文件存放在`judge_criteria/`目录，Checker通过`llm_judge_criteria_file`字段引用
- 支持版本化管理，方便A/B测试不同评分标准

**条件性检查**：
不是所有检查项对所有样本都适用。通过`skip_if_file_not_exists`和模板特有检查实现条件化：
- 感情线相关检查 → 仅对感情向需求的样本启用
- 短篇写作指南检查 → 仅对短篇样本启用
- writing_log检查 → 仅对>8000字的模板启用

---

## 四、关键设计原则总结

### 原则1: 顶层设计优先
所有修改先追溯到 unified_scenario_design.yaml / check_definitions，再级联到下游。永远不要跳过顶层直接改实现。

### 原则2: Rule-based优先于LLM Judge
能用精确规则验证的，绝不用LLM判断。LLM Judge只用于真正无法规则化的主观质量评价。

> **但要注意边界**：当"规则"包含主观判断成分时（如tone→reaction_strength映射），硬编码规则检查反而不如语义检查准确（rev_001→002的教训）。

### 原则3: 闭集合原则
Check type、能力维度、评分标准都必须是预定义的闭集合，不允许LLM自由发挥。

### 原则4: 分层验证
```
L1: 格式/结构（100%可自动化）
L2: 业务规则/一致性（95%可自动化）
L3: 内容质量Basic（LLM judge，通过/不通过）→ 一票否决
L4: 内容质量Advanced（LLM judge，1-5分）→ 综合评分
```

### 原则5: 评测驱动的迭代
先用最小可行集跑一轮评测，根据实际产出的问题针对性优化。小说场景的每个新检查项都来自具体的模型失败：
- plot_progression: "角色站着聊天3000字无事件"
- repeated_endings: "写了5次全文完"
- narrative_tone_match: "烧脑悬疑用言情笔法"

### 原则6: 测试区分度优于测试覆盖度
BusinessRules中的规则措辞选择（"建议" vs "必须"）要服务于区分模型能力的目标。

### 原则7: 正交解耦
场景输入和检查方案是正交维度，必须独立演进。这使得检查方案可以高频迭代而不需要重跑Agent执行。

### 原则8: 检查粒度足够细
粗粒度检查≈无效检查。一个100%通过率的检查项没有任何价值。

---

## 五、常见时间黑洞与应对策略

| 时间黑洞 | 应对策略 |
|---------|---------|
| 创作流程设计反复 | 确定线性流程就不再改，复杂度靠Skill文件补充 |
| LLM Judge Criteria调优 | 先用粗粒度criteria跑一轮，根据实际分数分布调整 |
| YAML与BusinessRules不一致 | 建立逐项对照checklist，样本生成器内置校验 |
| 配置错误导致重新评测 | 评测前强制做配置一致性检查（构建时校验） |
| 评价维度反复调整 | 基于真实评测结果新增维度，不凭空设计 |
| 多模型API兼容性问题 | 统一adapter层，提前测试每个provider的格式差异 |

---

## 六、快速启动Checklist（新创意场景）

当你开始构建一个新的创意类评测场景时，按以下顺序执行：

- [ ] **Step 1**: 明确Agent角色定位和创作流程（BusinessRules.md草稿）
- [ ] **Step 2**: 编写2-3个核心Skill文件（领域知识）
- [ ] **Step 3**: 让Agent跑3-5个手动case，观察实际产出
- [ ] **Step 4**: 基于实际产出设计能力维度分类（check_capability_taxonomy.yaml）
- [ ] **Step 5**: 设计Checklist（先rule-based，再semantic）
- [ ] **Step 6**: 编写LLM Judge Criteria（参考真实评测案例，不让AI自造）
- [ ] **Step 7**: 建立check_definitions/目录，实现检查方案与场景输入的解耦
- [ ] **Step 8**: 结构化到unified_scenario_design.yaml，实现样本生成器
- [ ] **Step 9**: 合成样本，确保构建时校验通过
- [ ] **Step 10**: 小批量评测（3-5个样本 × 2个模型），验证评价体系
- [ ] **Step 11**: 根据初测结果迭代检查方案（利用recheck机制，无需重跑Agent）
- [ ] **Step 12**: 全量评测 + 多模型对比分析

---

## 七、参考文件索引

| 文件 | 路径 | 说明 |
|-----|------|------|
| 小说场景统一配置 | `tmp_scenarios/novel_writing_alchemist/design_v2/unified_scenario_design.yaml` | 最完整的创意类场景YAML示例 |
| 小说场景BusinessRules | `tmp_scenarios/novel_writing_alchemist/design_v2/BusinessRules.md` | Agent system prompt范例 |
| 小说能力维度分类 | `tmp_scenarios/novel_writing_alchemist/check_capability_taxonomy.yaml` | v2.7三层维度体系（含完整版本历史） |
| 检查项定义（解耦） | `tmp_scenarios/novel_writing_alchemist/check_definitions/` | 通用+模板检查项、judge criteria、revision log |
| 版本迭代日志 | `tmp_scenarios/novel_writing_alchemist/check_definitions/check_revisions/REVISION_LOG.yaml` | rev_001→004的完整变更记录 |
| 样本生成器 | `tmp_scenarios/novel_writing_alchemist/design_v2/scripts/sample_generator/main.py` | 含解耦、校验、双模式输出 |
| 深度分析范例 | `tmp_scenarios/novel_writing_alchemist/analysis/ernie_vs_claude_character_consistency_analysis.md` | 模型能力对比的深度分析标杆 |
| output_completeness问题分析 | `tmp_scenarios/novel_writing_alchemist/analysis/output_completeness_issue_analysis.md` | 检查粒度不足导致100%通过率的经典案例 |
| v3修复总结 | `tmp_scenarios/novel_writing_alchemist/analysis/v3_major_fixes_summary.md` | 拆分output_completeness+新增memory_management的完整记录 |
| 项目开发标准 | `docs/PROJECT_DEVELOPMENT_STANDARDS_V2.md` | 通用开发SOP |
