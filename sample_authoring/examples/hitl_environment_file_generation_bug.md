# 样本生成器bug案例：HITL环境文件缺失

## 问题现象

**场景**：短剧创作场景的SD_MISSING_GENRE_INFO样本

**Agent执行失败**：Agent调用`request_human_review`工具询问题材信息时，工具返回error：
```json
{
  "error": "未找到阶段 '信息收集' 的预设答案",
  "detail": "hitl_responses中没有 '信息收集' 的配置"
}
```

**初步怀疑**：是不是Tool设计有问题？

---

## 根因追溯

### Step 1：检查统一配置

**unified_scenario_design.yaml** (lines 895-907)：
```yaml
- need_template_id: "SD_MISSING_GENRE_INFO"
  description: "用户未指定题材，测试Agent是否主动询问收集关键信息"

  user_need_description: |
    我想做短剧。

  # ✅ HITL预设响应配置正确
  hitl_responses:
    信息收集:
      answer: "{{genre_name_cn}}"  # 会替换成：都市甜宠、复仇爽剧、年代家庭、悬疑推理
```

**结论**：配置正确，不是这里的问题。

### Step 2：检查MCP Service实现

**shortdrama_service.py** (lines 25-42)：
```python
def load_hitl_context():
    """从workspace/.hitl_context.json加载HITL上下文"""
    global HITL_CONTEXT

    if not WORK_DIR:
        return

    # ✅ Tool期望从环境文件读取
    context_file = os.path.join(WORK_DIR, "workspace", ".hitl_context.json")

    if os.path.exists(context_file):
        try:
            with open(context_file, 'r', encoding='utf-8') as f:
                HITL_CONTEXT = json.load(f)
        except Exception as e:
            HITL_CONTEXT = {}
    else:
        # ❌ 文件不存在，HITL_CONTEXT为空
        HITL_CONTEXT = {}
```

**结论**：Tool实现正确，它期望从`workspace/.hitl_context.json`文件读取配置。问题是这个文件不存在。

### Step 3：检查生成的样本

**样本JSON结构验证**：
```bash
# 检查样本是否有hitl_responses元数据
$ grep "SD_MISSING_GENRE_INFO" samples/eval.jsonl | head -1 | jq 'keys | sort'
[
  "check_list",
  "data_id",
  "environment",
  "extension",
  "hitl_responses",  # ✅ 有这个字段
  "query",
  "servers",
  "system"
]

# 检查environment数组有哪些文件
$ grep "SD_MISSING_GENRE_INFO" samples/eval.jsonl | head -1 | jq '.environment[].path'
"data_pools/output_specifications.yaml"
"data_pools/skills/family_saga_skill.md"
"data_pools/skills/mystery_thriller_skill.md"
"data_pools/skills/revenge_drama_skill.md"
"data_pools/skills/sweet_romance_skill.md"
# ❌ 缺少 workspace/.hitl_context.json
```

**问题确认**：
- ✅ 样本JSON顶层有`hitl_responses`字段（元数据）
- ❌ environment数组中没有`.hitl_context.json`文件（运行时数据）

### Step 4：定位样本生成器bug

**generate_samples.py原代码** (lines 224-228)：
```python
# 添加hitl_responses（如果存在）
if 'hitl_responses' in template:
    hitl_responses = copy.deepcopy(template['hitl_responses'])
    hitl_responses = self.replace_placeholders_recursive(hitl_responses, context)

    # ✅ 添加到样本JSON作为元数据
    sample['hitl_responses'] = hitl_responses

    # ❌ BUG：没有生成对应的环境文件！
    # Tool期望从environment数组中的.hitl_context.json读取，但这个文件没有被创建
```

**问题根源**：
1. 样本生成器只把`hitl_responses`添加为样本JSON的元数据字段
2. 没有将它转换成环境文件添加到`environment`数组
3. 导致MCP service启动时找不到`.hitl_context.json`文件
4. `HITL_CONTEXT`为空字典，Tool返回error

---

## 完整链路图

```
统一配置 (YAML)
  └─ hitl_responses:
       信息收集:
         answer: "{{genre_name_cn}}"
           ↓
样本生成器 generate_samples.py
  ├─ ✅ 替换占位符：{{genre_name_cn}} → "都市甜宠"
  ├─ ✅ 添加到sample['hitl_responses']（元数据）
  └─ ❌ BUG：未生成environment文件
           ↓
生成的样本 eval.jsonl
  ├─ sample['hitl_responses'] ✅ 存在
  └─ sample['environment']    ❌ 缺少.hitl_context.json
           ↓
MCP Service启动
  └─ load_hitl_context()
       └─ 读取 workspace/.hitl_context.json
            ↓ 文件不存在
          HITL_CONTEXT = {}
           ↓
Agent调用request_human_review工具
  └─ Tool检查 HITL_CONTEXT['信息收集']
       └─ 找不到 → 返回error信息
```

---

## 修复方案

**修复代码** (generate_samples.py lines 224-237)：
```python
# 添加hitl_responses（如果存在）
if 'hitl_responses' in template:
    hitl_responses = copy.deepcopy(template['hitl_responses'])
    hitl_responses = self.replace_placeholders_recursive(hitl_responses, context)
    sample['hitl_responses'] = hitl_responses

    # ✅ 同时将hitl_responses添加到environment中作为.hitl_context.json文件
    # 这样MCP service的load_hitl_context()才能读取到
    hitl_context = {"hitl_responses": hitl_responses}
    environment.append({
        "path": "workspace/.hitl_context.json",
        "type": "file",
        "content": json.dumps(hitl_context, ensure_ascii=False, indent=2)
    })
```

**修复验证**：
```bash
# 重新生成样本
$ python scripts/sample_generator/generate_samples.py --base-dir .

# 验证environment数组长度（从5变成6）
$ grep "SD_MISSING_GENRE_INFO" samples/eval.jsonl | head -1 | jq '.environment | length'
6

# 验证.hitl_context.json文件存在
$ grep "SD_MISSING_GENRE_INFO" samples/eval.jsonl | head -1 | jq '.environment[].path' | grep hitl
"workspace/.hitl_context.json"

# 验证文件内容正确
$ grep "SD_MISSING_GENRE_INFO_SWEET_ROMANCE" samples/eval.jsonl | jq '.environment[] | select(.path == "workspace/.hitl_context.json") | .content' -r
{
  "hitl_responses": {
    "信息收集": {
      "answer": "都市甜宠"
    }
  }
}
```

---

## 核心教训

### 教训1：元数据 vs 运行时数据

| 字段位置 | 用途 | 作用时机 |
|---------|------|----------|
| `sample['hitl_responses']` | 元数据，用于分析和调试 | 评测结束后分析 |
| `sample['environment'][*]` | 运行时数据，供MCP service使用 | Agent执行时读取 |

**关键点**：
- 样本JSON的顶层字段是元数据
- environment数组是真正的运行时数据
- Tool只能访问environment数组中的文件，看不到样本JSON的其他字段

### 教训2：样本生成器的职责边界

**样本生成器必须完成的工作**：
1. ✅ 从统一配置读取template定义
2. ✅ 替换占位符（如{{genre_name_cn}} → "都市甜宠"）
3. ✅ 构建样本JSON结构（data_id、query、system、check_list等）
4. ✅ **关键**：将Tool依赖的配置数据转换为环境文件格式
5. ✅ 添加到environment数组

**容易遗漏的**：
- 只把数据加到样本元数据，忘记生成环境文件
- Tool期望某个文件存在，但environment数组中没有
- 环境文件格式与Tool期望不一致

### 教训3：Tool与样本生成器的配合清单

**开发新Tool时的检查点**：
- [ ] 明确Tool依赖哪些环境文件（路径、格式）
- [ ] 在Tool的docstring或注释中说明环境文件依赖
- [ ] 示例：`# 依赖文件：workspace/.hitl_context.json，格式：{"hitl_responses": {...}}`

**开发样本生成器时的检查点**：
- [ ] 阅读Tool实现，识别所有环境文件依赖
- [ ] 为每个依赖创建对应的environment文件
- [ ] 验证文件内容格式符合Tool期望
- [ ] 测试：运行样本，检查Tool是否能正常读取文件

### 教训4：如何避免类似bug

**设计阶段**：
```python
# ✅ Tool实现中明确注释环境文件依赖
def load_hitl_context():
    """
    从workspace/.hitl_context.json加载HITL上下文

    预期文件格式：
    {
      "hitl_responses": {
        "信息收集": {"answer": "..."},
        "任务进度确认": {"answer": "..."}
      }
    }
    """
    context_file = os.path.join(WORK_DIR, "workspace", ".hitl_context.json")
    ...
```

**实现阶段**：
```python
# ✅ 样本生成器中创建environment文件
if 'hitl_responses' in template:
    # 步骤1：处理元数据
    sample['hitl_responses'] = hitl_responses

    # 步骤2：生成环境文件（关键！）
    environment.append({
        "path": "workspace/.hitl_context.json",
        "type": "file",
        "content": json.dumps({"hitl_responses": hitl_responses}, ...)
    })
```

**测试阶段**：
```bash
# ✅ 验证environment数组完整性
jq '.environment[].path' samples/eval.jsonl | sort | uniq

# ✅ 检查特定文件是否存在
jq '.environment[] | select(.path == "workspace/.hitl_context.json")' samples/eval.jsonl | head
```

---

## 扩展讨论

### 为什么不直接从sample JSON读取？

**技术上可行的替代方案**：
```python
# MCP service启动时获取sample JSON
sample_metadata = load_sample_metadata()
HITL_CONTEXT = sample_metadata.get('hitl_responses', {})
```

**为什么不推荐**：
1. **职责混淆** - MCP service应该独立于评测框架
2. **真实性降低** - 真实应用中，配置数据通常存储在文件中
3. **复用性差** - 同一个MCP service无法用于不同的评测框架
4. **调试困难** - 无法独立检查和修改环境文件

**正确的设计理念**：
- MCP service只依赖environment数组中的文件
- 样本JSON的其他字段对MCP service不可见
- 这样保持了清晰的职责边界

### 类似问题的排查步骤

1. **Agent执行失败** → 查看Tool返回的error信息
2. **Tool返回error** → 检查Tool实现，确认依赖的环境文件
3. **确认Tool期望** → 检查生成的样本，验证environment数组是否包含该文件
4. **environment缺失文件** → 追溯到样本生成器，检查是否正确生成
5. **样本生成器缺陷** → 修复代码，重新生成样本，验证修复效果

---

## 检查清单

**样本生成器开发时**：
- [ ] 识别Tool依赖的所有环境文件
- [ ] 为每个依赖生成对应的environment文件
- [ ] 文件路径与Tool期望一致（如`workspace/.hitl_context.json`）
- [ ] 文件内容格式与Tool期望一致（如`{"hitl_responses": {...}}`）
- [ ] 占位符正确替换（如`{{genre_name_cn}}`）

**样本验证时**：
- [ ] 检查environment数组文件数量是否正确
- [ ] 检查特定文件是否存在（grep + jq）
- [ ] 检查文件内容是否正确（jq + select）
- [ ] 运行样本，验证Tool能正常读取环境文件

---

## 相关文件

- **样本生成器**：`scenarios/shortdrama/scripts/sample_generator/generate_samples.py`
- **统一配置**：`scenarios/shortdrama/unified_scenario_design.yaml`
- **MCP Service**：`scenarios/shortdrama/env/shortdrama_service.py`
- **生成的样本**：`scenarios/shortdrama/samples/eval.jsonl`

---

## 标注信息

- **问题类型**：样本生成器bug - 环境文件缺失
- **核心教训**：元数据vs运行时数据、Tool与样本生成器配合
- **修复难度**：简单（添加7行代码）
- **影响范围**：所有依赖环境文件的Tool场景
- **日期**：2026-01-14
- **标注者**：Claude (based on debugging session)
