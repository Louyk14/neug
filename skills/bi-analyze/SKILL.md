---
name: bi-analyze
description: "Execute comprehensive BI analysis workflow: understand database schema, decompose user questions, query data via GraphCube, visualize results, run control group comparisons, perform deep iterative analysis, and generate a complete Markdown report with charts. Use this skill whenever the user asks for data analysis, BI reports, exploratory data analysis, trend analysis, distribution analysis, comparison analysis, or any analytical question about graph-structured data."
---

# BI Analyze Skill

你必须严格按照本文件的结构和流程执行分析，生成单一完整的 Markdown 分析报告。**请完整阅读本文件后再开始执行。**

---

## ⚠️ 核心原则（违反即不合格）

1. **追因分析必须基于查询验证**：禁止用"可能原因"替代实际查询
2. **每个子问题都必须调用 subagent 追因**：不能跳过任何一个
3. **Subagent 失败必须在主会话补救**：超时/失败不是跳过的理由
4. **⚠️ GraphCube 不支持并发查询**：所有查询必须串行执行，同一时间只能有一个查询在运行（包括主 agent 和 subagent）

---

## 一、报告结构（你的最终输出必须严格遵循此结构）

```markdown
# 分析报告

## 1. 执行摘要
[核心数据亮点表格]

## 2. 问题理解与实体对齐
### 2.1 实体对齐表（每个实体至少5个候选映射，含关联方式+回答角度）
### 2.2 映射分层（第一层/第二层）

## 3. 问题拆解 + 分析计划
### 3.1 拆解策略与子问题列表
### 3.2 分析计划表（每个子问题的时间范围、约束、对照组设计）

## 4. 详细分析

### 4.X [子问题标题]

#### 4.X.1 查询分析
[查询配置 + 核心发现(含数值) + 数据表 + 可视化图表]

#### 4.X.2 对照验证
[对照设计 + 对比表(原始组/对照组/差异/差异率) + 对比图 + 验证结论]

#### 4.X.3 追因分析（必须调用 subagent，见下方流程）
[基于4.X.1和4.X.2的发现，提出"为什么"问题，拆解为追因子问题]

##### 4.X.3.1 [追因子问题1]
[查询分析 + 可视化 + 对照验证 + 结论]
（如果递归深度<2，可继续追因: 4.X.3.1.3.1 ...）

##### 4.X.3.2 [追因子问题2]
[...]

#### 4.X.4 小结
**主分析结论**: [4.X.1的发现]
**对照结论**: [4.X.2的发现]
**追因结论**: [4.X.3的发现]
**因果链**: [主发现 ← 追因1 ← 追因2]

### 4.Final 顶层总结

## 5. 结论与建议（局限性仅含确实无法查询的方向）

## 附录
```

**以上结构是硬性要求。如果你的报告缺少 4.X.3（追因分析）章节，或追因分析只有"可能原因"而无查询验证，报告不合格。**

---

## 二、每个子问题的执行流程（7步，不可跳过任何一步）

对**每个**子问题 4.X，按以下顺序执行：

```
步骤1 → 构建 GraphCube 查询（使用 graphcube skill）
步骤2 → 执行查询，获取结果
步骤3 → 提取结论（必须引用具体数值，禁止"较高""较多"等模糊表述）
步骤4 → 绘制可视化
步骤5 → 对照验证（构造对照查询→执行→计算差异→绘制对比图）
步骤6 → 写入报告的 4.X.1 和 4.X.2 章节
步骤7 → ⚠️ 调用 subagent 递归追因（写入 4.X.3 章节）← 这一步禁止跳过
```

### 步骤7详解: 递归追因（重点！）

#### 7.0 ⚠️ 串行执行原则（GraphCube 并发限制）

**GraphCube 服务器不支持并发查询**，必须严格遵守以下规则：

1. **同一时间只能有一个查询在执行**：
   - 主 agent 执行查询时，不能启动 subagent
   - 一个 subagent 运行时，不能启动另一个 subagent
   - 必须等待当前查询完成后，才能执行下一个

2. **禁止并行启动多个 subagent**：
   ```python
   # ❌ 错误：同时启动多个 subagent
   sessions_spawn(runtime="subagent", task="任务1")
   sessions_spawn(runtime="subagent", task="任务2")  # 禁止！
   
   # ✅ 正确：串行执行
   sessions_spawn(runtime="subagent", task="任务1")
   # 等待完成后再启动下一个
   subagents(action="list")  # 确认状态为完成
   sessions_spawn(runtime="subagent", task="任务2")
   ```

3. **等待机制**：
   - 启动 subagent 后，必须等待其完成（或超时）
   - 使用 `subagents(action="list")` 检查状态
   - 只有当前 subagent 状态为 `completed` 或 `timeout` 或 `error` 时，才能进行下一步操作

4. **查询间隔**：
   - GraphCube 查询之间建议间隔 1-2 秒
   - 避免频繁查询导致服务器压力过大

#### 7.1 选择追因问题

完成步骤1-6后，从核心发现中选2-3个最值得追问的：
- 超预期变化
- 对照组差异
- 时间异常
- 违反直觉

#### 7.2 调用 subagent（串行执行）

**⚠️ 重要：每次只能启动一个 subagent，必须等待完成后才能启动下一个**

使用 **sessions_spawn** 工具调用 subagent，**必须设置 timeoutSeconds=1800（30分钟）**：

```python
# 步骤1: 确认当前没有其他 subagent 在运行
active_subagents = subagents(action="list")
if active_subagents.get("active") and len(active_subagents["active"]) > 0:
    # 有 subagent 正在运行，必须等待
    print("等待当前 subagent 完成...")
    # 使用 process 或 exec 等待

# 步骤2: 启动 subagent
sessions_spawn(
    runtime="subagent",
    mode="run",
    timeoutSeconds=1800,  # ⚠️ 必须设置30分钟
    task="""你是一个BI分析助手。请对以下发现进行追因分析。

【发现】: {finding}
【追因问题】: 为什么会出现"{finding}"？

【当前递归深度】: {depth}（最大2，当前{depth}）
【章节编号前缀】: {prefix}（你的子问题编号为 {prefix}.1, {prefix}.2, ...）
【报告输出路径】: {output_path}
【images 目录】: {images_dir}

⚠️ 重要限制：GraphCube 不支持并发！
- 你的所有查询必须串行执行
- 每个查询之间间隔 1-2 秒
- 禁止同时发起多个查询请求

请执行:
1. 获取 Schema: curl http://127.0.0.1:8073/api/cube/schema
2. 等待 2 秒
3. 将追因问题映射到 Schema，拆解为 2-3 个可查询的子问题
4. 对每个子问题（串行执行）:
   - 构建 GraphCube 查询
   - 等待 2 秒
   - 执行查询
   - 提取数值结论
   - 绘制图表
   - 对照验证
5. 如果当前深度 < 2，对重要发现继续追因（串行）
6. 将分析结果（含图表）保存到 {output_path}

⚠️ 重要：
- 禁止用"可能原因"替代查询验证
- 所有结论必须引用具体数值
- 图片保存到 {images_dir}
- 查询必须串行执行，禁止并发！"""
)

# 步骤3: 等待 subagent 完成（不要立即进行其他操作）
# subagent 会自动宣布完成，或者使用 subagents(action="list") 检查状态
```

**递归深度**: depth=0(顶层) → depth=1(追因) → depth=2(最深，不再递归)

#### 7.3 检查 subagent 结果

使用 **subagents(action="list")** 检查执行状态：

```python
subagents(action="list")
```

#### 7.4 Subagent 失败处理（关键！）

**如果 subagent 超时或失败，必须在主会话手动补救：**

1. 使用 `subagents(action="list")` 确认状态
2. 如果状态为 `timeout` 或 `error`：
   - **不要跳过追因分析**
   - **不要用"可能原因"填充**
   - 在主会话中手动执行查询验证
   - 生成图表并写入报告

```python
# 示例：subagent 失败后手动补救
# 1. 确认失败
subagents(action="list")  # 发现 status="timeout"

# 2. 手动执行追因查询
exec(command="curl -s -X POST http://127.0.0.1:8073/api/cube/query ...")

# 3. 生成图表
exec(command="python3 plot.py")

# 4. 更新报告
edit(path="report.md", oldText="...", newText="基于查询验证的结论...")
```

#### 7.5 完成追因

完成后读取 subagent 结果（或手动补救结果），嵌入到 4.X.3.1, 4.X.3.2, ... 章节。然后写 4.X.4 小结（必须包含因果链）。

---

## 三、分析流程

### Step 1: 数据库理解

```bash
curl -s http://127.0.0.1:8073/api/cube/schema | jq
```

逐一解析每个 label/属性的语义。详细指南见 [sub-skills/db-understand.md](sub-skills/db-understand.md)。

### Step 2: 问题理解与实体对齐

详细指南见 [sub-skills/question-understand.md](sub-skills/question-understand.md)。

1. 解析问题，识别类型（趋势/对比/分布/相关性）
2. **实体对齐**: 每个实体列出**至少5个、最多10个**候选映射

报告中必须输出:

| 问题实体 | 候选映射 | 置信度 | 字段类型 | 关联方式 | 回答角度 |
|----------|----------|--------|----------|----------|----------|
| (至少5行) | | | | | |

3. **映射分层**: 第一层(高置信度→初始拆解) / 第二层(中低置信度→留给追因)

### Step 3: 问题拆解 + 分析计划

1. 选择拆解策略（A=按映射 / B=按维度 / 组合），写明原因
2. 生成子问题列表
3. **生成分析计划表**:

| 子问题 | 分析目标 | 时间范围 | 关键约束 | 查询方式 | 对照组设计 |
|--------|----------|----------|----------|---------|-----------|
| (每个子问题一行) | | | | | |

### Step 4: 子问题处理

对每个子问题执行「二、每个子问题的执行流程」中的7步。

**⚠️ 关键检查点**：
- 每个子问题完成后，确认已调用 subagent
- 检查 subagent 状态，失败则手动补救
- 追因分析必须有查询数据支持，不能只有"可能原因"

### Step 5: 报告生成

所有子问题（含递归追因）完成后，分两步生成最终报告：

**5a. 组装报告**: 按「一、报告结构」将所有分析内容（含 subagent 追因结果）组装为完整报告。组装规则见 [sub-skills/report-gen.md](sub-skills/report-gen.md)。

**5b. ⚠️ 报告后处理（仅顶层 agent 执行，subagent 跳过）**:

由于 subagent 异步执行，组装后的报告可能存在占位符残留、内容错位、标题级别不对、文本粘连等问题。**你必须按 [sub-skills/report-cleanup.md](sub-skills/report-cleanup.md) 的流程逐一修复**，然后全文通读验证。

关键修复项：
1. 清除占位符（"详见 subagent"/"执行中"等）
2. 将错位在报告末尾的追因内容移动到正确的 4.X.3 位置
3. 将 subagent 输出的标题降级到正确嵌套层级
4. 修复文本粘连，更新小结中的追因结论

---

## 四、执行清单（完成分析后逐项确认）

### 必查项目（任一项不合格则报告不合格）

- [ ] Schema 已获取，每个 label/属性语义已解析
- [ ] 实体对齐表: 每个实体至少5个候选映射
- [ ] 映射已分层: 第一层 + 第二层
- [ ] 分析计划表已生成（含时间范围、对照组设计）
- [ ] **所有 GraphCube 查询都是串行执行的，没有并发查询**
- [ ] **每个子问题都有 4.X.1(查询分析) + 4.X.2(对照验证) + 4.X.3(追因分析) + 4.X.4(小结)**
- [ ] **每个子问题都串行调用了 subagent**（用 `subagents(action="list")` 验证前一个已完成）
- [ ] **Subagent 失败的，已在主会话手动补救**
- [ ] **追因分析有查询数据支持**，没有"可能原因"这种空洞表述
- [ ] 4.X.4 小结包含因果链
- [ ] 所有子问题和追因都有可视化图表
- [ ] 顶层总结综合回答了原始问题
- [ ] 局限性章节不包含已分析过的内容
- [ ] **⚠️ 报告后处理已完成（仅顶层）**: 无占位符残留、追因内容在正确的 4.X.3 位置、标题级别正确、无文本粘连

---

## 五、常见错误及避免方法

| 错误 | 表现 | 正确做法 |
|------|------|----------|
| 跳过追因分析 | 4.X.3 章节缺失或为空 | 每个子问题都必须调用 subagent |
| 用"可能原因"替代验证 | 追因分析只有推测无数据 | 必须执行查询，引用具体数值 |
| Subagent 失败后跳过 | 发现 timeout 后不补救 | 在主会话手动执行查询 |
| 只对一个子问题追因 | 其他子问题的 4.X.3 为空 | 每个子问题都要独立追因 |
| 追因深度不足 | 只有 4.X.3 无 4.X.3.1 等 | 至少拆解为 2 个子问题 |
| **并发查询（严重）** | 同时启动多个 subagent 或主 agent 查询 | **必须串行执行，等待前一个完成** |

### 并发查询的正确处理

```python
# ❌ 错误：并发查询会导致 GraphCube 返回错误或超时
sessions_spawn(runtime="subagent", task="任务1")
sessions_spawn(runtime="subagent", task="任务2")  # 禁止！

# ✅ 正确：串行执行
# 1. 启动第一个 subagent
sessions_spawn(runtime="subagent", task="任务1")

# 2. 等待完成（subagent 会自动宣布完成）
# 或者主动检查：
# while True:
#     result = subagents(action="list")
#     if not result.get("active"):
#         break
#     time.sleep(30)  # 等待30秒后再检查

# 3. 确认第一个完成后，再启动第二个
sessions_spawn(runtime="subagent", task="任务2")
```

---

## 六、依赖

| Skill | 用途 |
|-------|------|
| **graphcube** | 构建 query_config，执行 Graph OLAP 查询 |

| 服务 | 地址 | 健康检查 |
|------|------|----------|
| GraphCube Server | `http://127.0.0.1:8073` | `curl http://127.0.0.1:8073/api/cube/schema` |

## Sub-Skills 参考

| 文件 | 内容 |
|------|------|
| [sub-skills/db-understand.md](sub-skills/db-understand.md) | Schema 解析详细方法 |
| [sub-skills/question-understand.md](sub-skills/question-understand.md) | 实体映射和问题拆解策略 |
| [sub-skills/query-execute.md](sub-skills/query-execute.md) | query_config 构建和错误处理 |
| [sub-skills/result-analyze.md](sub-skills/result-analyze.md) | 可视化绘图规范 |
| [sub-skills/control-group.md](sub-skills/control-group.md) | 对照组设计方法 |
| [sub-skills/report-gen.md](sub-skills/report-gen.md) | 报告组装和递归嵌套格式（所有层级通用） |
| [sub-skills/report-cleanup.md](sub-skills/report-cleanup.md) | 报告后处理: 修复 subagent 异步导致的错位/占位符（仅顶层） |
| [templates/report-template.md](templates/report-template.md) | 完整报告 Markdown 模板 |