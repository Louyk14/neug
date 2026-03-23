# 数据库理解 (Database Understanding)

> 本文件是 SKILL.md Step 1 的详细补充指南。核心规则已在 SKILL.md 中内联，此处提供更细致的方法。

在分析之前，先全面理解数据库的结构和语义，为后续实体映射提供坚实基础。

## 执行步骤

### 1. 获取 Schema

```bash
curl -s http://127.0.0.1:8073/api/cube/schema | jq
```

### 2. 逐 Label 解析节点语义

对 Schema 中的**每个节点类型 (Vertex Label)**，逐一分析：

1. **Label 名称的语义**: 这个 label 代表什么现实实体？（如 `Publication` = 学术论文，`Author` = 论文作者）
2. **在图中的角色**: 核心实体 / 辅助实体 / 关联实体
3. **所有属性逐一解析**:

对每个属性，记录以下信息：

| 属性名 | 数据类型 | 语义描述 | 属性分类 | 分析用途 |
|--------|----------|----------|----------|----------|
| id | int64 | 唯一标识符 | 标识属性 | 用于 COUNT 计数 |
| name | string | 名称 | 标识属性 | 用于 drill_down 获取具体名称 |
| year | int32 | 发表年份 | 时间属性 | 用于时序分析/时间约束 |
| NumCited | int32 | 被引用次数 | 度量属性 | 作为 dimension + drill_down |
| type | string | 类别 | 分类属性 | 用于分组/筛选 |
| score | float | 评分 | 数值属性 | 用于 constraint 或 measure |

**属性分类标准**:

| 分类 | 特征 | Schema 中的线索 | GraphCube 中的用法 |
|------|------|----------------|-------------------|
| 标识属性 | 唯一标识实体 | id, name, code, key | drill_down 获取具体值 |
| 度量属性 | 本身就是统计/聚合值 | Num*, Avg*, Total*, Sum*, Count*, Rate*, Pct*, Mean* | **作为 dimension + drill_down，不要作为 measure** |
| 分类属性 | 有限离散值 | type, category, status, level, class | 用于 constraint 筛选或 dimension 分组 |
| 数值属性 | 连续数值，非统计值 | age, salary, price, size | 用于 constraint 约束或 measure 聚合 |
| 时间属性 | 时间信息 | year, date, month, timestamp | 用于时序分析、constraint 时间范围 |
| 文本属性 | 自由文本 | description, abstract, title | 通常仅用于展示，难以直接查询 |

### 3. 逐 Label 解析边语义

对 Schema 中的**每个边类型 (Edge Label)**，逐一分析：

1. **边名称的语义**: 这条边代表什么关系？（如 `author_writtenBy_publication` = 作者撰写了论文）
2. **方向性含义**: source → target 的方向表示什么
3. **连接的节点类型**: 明确 source 和 target 的 vertex label
4. **边上的属性**: 如果有属性，逐一解析其语义（同节点属性分析方法）

**边语义解析表**:

| 边标签 | 语义描述 | source (含义) | target (含义) | 边属性 | 属性语义 |
|--------|----------|--------------|--------------|--------|----------|
| person_workAt_company | 人在公司工作 | person (人) | company (公司) | since_year | 入职年份 |
| author_writtenBy_pub | 作者撰写论文 | author (作者) | publication (论文) | — | — |

### 4. 图拓扑理解

分析整体图结构的连通方式：

- **核心连接路径**: 哪些节点通过什么边相连，列出主要路径
- **多跳路径**: 是否存在 A → B → C 的多跳路径可以回答更复杂的问题
- **Hub 节点**: 哪些节点类型连接了最多种类的边（通常是分析的核心）

```
示例拓扑:
Author ──writtenBy──▶ Publication ──publishedIn──▶ Journal
   │                      │
   └──affiliatedWith──▶ Organization ──locatedIn──▶ Country
```

### 5. 产出

将以上分析整理为 **Schema 语义概要**，必须包含：

1. **图结构总览**: 一句话描述数据库建模的领域和核心实体
2. **逐节点语义**: 每个 vertex label 的含义和所有属性的语义解析（含属性分类）
3. **逐边语义**: 每个 edge label 的含义、方向性、连接关系
4. **度量属性标注**: 明确标注哪些属性是已有统计值（这些在 GraphCube 中必须作为 dimension + drill_down）
5. **关联路径图**: 主要的节点连接路径

这份概要将作为 Step 2（问题理解）中实体映射的核心参考。属性语义理解得越深入，后续实体映射的质量越高。
