# 查询构建与执行 (Query Execute)

> 本文件是 SKILL.md Step 4.1-4.2 的详细补充指南。此处提供 query_config 构建细节和错误处理策略。

对每个叶子子问题，构建 GraphCube query_config 并执行查询。依赖 **graphcube skill** 进行查询。

## 执行流程

对每个叶子子问题 Q，按以下步骤执行：

### 1. 确认实体映射

从问题理解阶段获取 Q 的实体映射选择：
- 使用哪个节点标签 (vertex label)
- 使用哪个边标签 (edge label)
- 使用哪些属性作为 dimension / measure / constraint

### 2. 构建 query_config

使用 graphcube skill 的规范构建查询：

**vertices + edges**: 根据子问题涉及的实体和关系

**constraints**: 根据提取的约束条件

**dimension vs measure 判断**:
- 属性本身是统计值（Num*, Avg*, Total* 等）→ 作为 **dimension**，在 internal 中 drill_down
- 需要聚合计算 → 作为 **measure**（COUNT/SUM/MIN/MAX）

**internal**: 确保对所有需要展示具体值的 dimension 执行 drill_down

**view**: 配置排序 (sort_by) 和数量限制 (top_n)

### 3. 执行查询

```bash
curl -X POST http://127.0.0.1:8073/api/cube/query \
  -H "Content-Type: application/json" \
  -d '<query_config_json>'
```

### 4. 结果校验

检查返回结果：

| 情况 | 处理 |
|------|------|
| 正常返回数据 | 记录结果，进入分析 |
| 空结果 | 检查 constraint 是否过严，放宽后重试 |
| API 错误 | 检查 query_config 格式，修正后重试 |
| **查询返回 0 条结果** | **这是有效结果**，0 本身是有意义的数据点（如"2023 年无新项目"） |
| 结果不合理 | 检查实体映射，尝试备选映射重新查询 |
| **字段无匹配数据** | **必须尝试替代路径**（见下方） |

> **⚠️ "返回 0 条结果"≠"查询失败"**。如果查询配置正确但返回 0，说明该维度确实不存在数据，这本身就是重要发现。只有 API 报错或字段不存在时才是真正的查询失败。

### 5. 备选映射与替代路径处理

**⚠️ 禁止因某条路径不通就放弃对原始问题的回答。**

如果高置信度映射的查询结果不理想或无数据：

**第一步：尝试同实体的其他候选映射**
1. 回退到中/低置信度映射
2. 重新构建 query_config
3. 再次执行查询

**第二步：尝试多跳路径**

如果直接映射都失败，分析图拓扑寻找间接路径：

```
直接路径失败: Funding.jurisdiction 无 "RU" 数据
    │
    ▼
替代路径1: Author(country=RU) → Publication → funded_by → Funding
    （查询俄罗斯作者的出版物获得了哪些资金）
    │
    ▼
替代路径2: Organization(country=RU) → employs → Author → Publication → funded_by → Funding
    （查询俄罗斯机构的作者获得了哪些资金）
```

**第三步：记录所有尝试**

在报告中记录：
- 每个尝试过的路径及其结果
- 映射切换的原因
- 最终选择的路径及其合理性

## 关键约束（来自 graphcube skill）

- vertices 最多 3 个节点，edges 最多 3 条边
- `input`、`internal`、`view` 是平级的三个顶层 key
- `internal` 中 `value_path` 第一项必须是 `"*"`
- `sort_by` 和 `top_n` 必须同时存在或同时不存在
- 禁止直接重复使用之前的 cube，必须重新 source
- calculations 中的 op1/op2 必须在 internal 中做过 drill_down

## 产出

每个叶子子问题的查询结果，包含：
1. 使用的 query_config（关键参数说明）
2. 查询返回的原始数据
3. 如有映射切换，记录原因
