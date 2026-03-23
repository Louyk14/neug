---
name: graphcube
description: "Query and analyze graph data using GraphCube OLAP operations. Build cubes, perform drill-down and roll-up operations for graph-based data analysis."
metadata: {"clawdbot":{"emoji":"📊","requires":{"bins":["curl"]}}}
---

# 📊 GraphCube Skill

Query and analyze graph-structured data using GraphCube's OLAP operations. Build cubes from graph patterns, then apply analytical operators like drill-down and roll-up.

## Server Setup

GraphCube server should be running at `http://127.0.0.1:8073` (default).

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/cube/schema` | GET | Get database schema info |
| `/api/cube/query` | POST | Execute query with cube operations |
| `/api/data/open` | POST | Open an existing database |

## Quick Start

### 1. Open an Existing Database 
```bash
curl -X POST http://127.0.0.1:8073/api/data/open \
  -H "Content-Type: application/json" \
  -d '{"databasePath": "/path/to/your/database"}'
```

**参数说明:**
- `databasePath`: 已有数据库目录的绝对路径，需要用户提供

**重要**
- Open Dataset的时候需要构造graph index，当图比较大的时候可能用时会超过10分钟，但是graph index是必要的，所以务必耐心等待，等待时不要进行任何别的操作，等到open完成后再继续处理
- Open Dataset过程中禁止调用其他API Endpoints，包括`/api/cube/schema`、`/api/cube/query`、`/api/data/open`


### 2. Check Schema

```bash
curl -s http://127.0.0.1:8073/api/cube/schema | jq
```

**用途:** 在构建 Cube 前必须先检查 schema，确保所有点的属性都存在。

### 3. Execute Query

```bash
curl -X POST http://127.0.0.1:8073/api/cube/query \
  -H "Content-Type: application/json" \
  -d @query.json
```

---

## Query Config 结构详解

`query_config` 是一个包含三个顶层 key 的 JSON 对象：

```json
{
  "input": { ... },      // 第一层级：数据源定义 (必需)
  "internal": [ ... ],   // 第一层级：中间算子列表 (必需，可为空 [])
  "view": { ... }        // 第一层级：视图定义 (必需)
}
```

**⚠️ 关键：`input`、`internal`、`view` 是平级的三个顶层 key，不是嵌套关系！**

---

## 1. Input 数据源定义

### Type: source (创建新 Cube)

```json
{
  "input": {
    "type": "source",
    "params": {
      "cube_name": "MyCube",
      "sample_size": -1,
      "vertices": [...],
      "edges": [...],
      "constraints": [...],
      "dimension": [...],
      "measure": [...]
    }
  }
}
```

#### 参数详解

##### cube_name
- **类型:** string
- **说明:** 新 Cube 的名称，必须全局唯一
- **示例:** `"PopulationByPlace"`

##### sample_size
- **类型:** integer
- **说明:** 采样大小，`-1` 表示使用全部数据
- **示例:** `-1` (全量) 或 `1000` (采样1000条)

##### vertices (节点列表)
- **类型:** array of objects
- **说明:** 定义图模式中的节点
- **⚠️ 限制:** 最多包含 **3 个节点**

```json
"vertices": [
  {"alias": "v0", "label": "person"},
  {"alias": "v1", "label": "place"},
  {"alias": "v2", "label": "company"}
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `alias` | string | 节点别名，用于后续引用 (如 v0, v1, v2) |
| `label` | string | 节点标签，需匹配 schema 中的 vertex label |

##### edges (边列表)
- **类型:** array of objects
- **说明:** 定义图模式中的边
- **⚠️ 限制:** 最多包含 **3 条边**

```json
"edges": [
  {
    "alias": "e0",
    "label": "person_isLocatedIn_place",
    "source": "v0",
    "target": "v1"
  },
  {
    "alias": "e1",
    "label": "person_workAt_company",
    "source": "v0",
    "target": "v2"
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `alias` | string | 边别名，用于后续引用 (如 e0, e1) |
| `label` | string | 边标签，需匹配 schema 中的 edge label |
| `source` | string | 起始节点别名，**必须**是 vertices 中已声明的 alias |
| `target` | string | 目标节点别名，**必须**是 vertices 中已声明的 alias |

**⚠️ 重要:** `source` 和 `target` 必须引用 `vertices` 数组中已定义的 `alias`！

##### constraints (过滤条件列表)
- **类型:** array of objects
- **说明:** 定义初始过滤条件，可为空数组 `[]`

```json
"constraints": [
  {
    "alias": "v0",
    "type": "vertex",
    "constraint": [
      {
        "property": "age",
        "operator": ">",
        "value": "20",
        "value_type": "int32"
      },
      {
        "property": "score",
        "operator": ">=",
        "value": "100 - 'v0.penalty'",
        "value_type": "int32"
      },
      {
        "property": "gender",
        "operator": "=",
        "value": "male",
        "value_type": "string"
      },
      {
        "property": "email",
        "operator": "is not null",
        "value": "",
        "value_type": "string"
      }
    ]
  },
  {
    "alias": "e0",
    "type": "edge",
    "constraint": [
      {
        "property": "weight",
        "operator": ">=",
        "value": "0.5",
        "value_type": "double"
      }
    ]
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `alias` | string | 约束对象的别名，**必须**对应 vertices 或 edges 中的 alias |
| `type` | string | 约束对象类型，仅支持: `"vertex"` 或 `"edge"` |
| `constraint` | array | 具体约束条件列表 |

**constraint 内部字段详解:**

| 字段 | 类型 | 说明 | 可选值 |
|------|------|------|--------|
| `property` | string | 属性名称，**必须**是属性名，禁止使用常量或表达式 | 需匹配 schema 中的属性 |
| `operator` | string | 比较运算符 | `>`, `>=`, `<`, `<=`, `=`, `is null`, `is not null` |
| `value` | string | 比较值 (见下方详细说明) | |
| `value_type` | string | 值的数据类型 | `string`, `int32`, `int64`, `float`, `double` |

**value 字段格式说明:**

1. **常量值** (直接写，不加引号):
   - 数值: `"20"`, `"3.14"`, `"100"`
   - 字符串: `"male"`, `"Beijing"`

2. **引用同一 alias 的其他属性** (用单引号包裹 `'alias.property'`):
   - `"'v0.age1' / 'v0.age2'"` - 属性之间的运算
   - `"20 - 'v0.penalty'"` - 常量与属性的运算
   - **⚠️ 禁止:** 使用其他 alias 的属性

3. **is null / is not null 时**:
   - `value` 必须为 `""`
   - `value_type` 必须为 `"string"`

**⚠️ 重要规则:**
- `property` 必须是纯属性名，禁止常量或表达式
- `value` 中引用属性时必须使用 `'alias.property'` 格式 (单引号包裹)
- `value` 中的常量不需要引号包裹
- **禁止**在 `value` 中使用其他 alias 的属性
- 一些约束可能隐含在属性名里，如 `NumOfHighSchool` 已隐含学校类型为 high school

**约束表达式变换:**

如果用户问题中的约束需要变换才能写入 constraints，可进行简单数学变换:
- `a + b > c` 等价于 `a > c - b`
- `a - b > c` 等价于 `a > c + b`
- `a * b > c` 等价于 `a > c / b`
- `a / b > c` 等价于 `a > c * b`

##### dimension (维度定义)
- **类型:** array of objects
- **说明:** 定义分析维度，作为 Drill Down 的目标

```json
"dimension": [
  {
    "alias": "v1",
    "name": "city",
    "type": "vertex",
    "data_type": "string",
    "hierarchy": ["city"]
  },
  {
    "alias": "v0",
    "name": "NumTstTakr",
    "type": "vertex",
    "data_type": "int32",
    "hierarchy": ["NumTstTakr"]
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `alias` | string | 维度所属节点/边的别名 |
| `name` | string | 维度名称 (通常与属性名相同) |
| `type` | string | 类型: `"vertex"` 或 `"edge"` |
| `data_type` | string | 数据类型: `string`, `int32`, `int64`, `float`, `double` |
| `hierarchy` | array | 层级路径，如 `["country", "city"]` 或 `["name"]` |

**⚠️ 重要:** 如果某个属性本身已表示**统计值或度量结果** (如 `NumTstTakr` 表示"考生人数"，`AvgScore` 表示"平均分")，应将其作为**维度**加入，而不是 measure！然后在 `internal` 中对该属性执行 `drill_down` 以获取具体值。

##### measure (度量定义)
- **类型:** array of objects
- **说明:** 定义聚合度量

```json
"measure": [
  {
    "alias": "v0",
    "property_name": "id",
    "property_type": "vertex",
    "method": "COUNT"
  },
  {
    "alias": "v0",
    "property_name": "salary",
    "property_type": "vertex",
    "method": "SUM"
  }
]
```

| 字段 | 类型 | 说明 | 可选值 |
|------|------|------|--------|
| `alias` | string | 度量所属节点/边的别名 | |
| `property_name` | string | 属性名称 | |
| `property_type` | string | 属性类型 | `"vertex"` 或 `"edge"` |
| `method` | string | 聚合方法 | `COUNT`, `SUM`, `MIN`, `MAX` |

**聚合方法说明:**
- `COUNT`: 计数
- `SUM`: 求和 (需要数值类型属性)
- `MIN`: 最小值
- `MAX`: 最大值

**⚠️ 注意:** 如果 `property_name` 已经表示计数/聚集/度量值了，一般不会再对其计算 COUNT，因为结果一定是 1。

## 2. Internal 中间算子列表

按顺序执行的算子列表，支持 `drill_down` 和 `roll_up`。

**⚠️ 注意:** `internal` 是 query_config 的顶层 key，不是 input 的子字段！

### Drill Down (下钻)

将聚合值按维度分解。**这是最常用的算子！**

```json
{
  "type": "drill_down",
  "params": {
    "cube_name": "MyCube",
    "target_cube_name": "MyCube_DR_v1_city",
    "alias": "v1",
    "property_name": "city",
    "value_path": ["*"],
    "value_type": ["string"]
  }
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `cube_name` | string | 上一步输出的 Cube 名称 |
| `target_cube_name` | string | 本步输出的 Cube 名称 |
| `alias` | string | 操作对象的别名 |
| `property_name` | string | 操作属性名 |
| `value_path` | array | 路径数组，**第一项必须是 `"*"`** |
| `value_type` | array | 值类型数组，**第一项必须是 `"string"`**，必须与 value_path 一一对应 |

**⚠️ 重要:** `value_path` 的第一项必须是 `"*"`，对应 `value_type` 的第一项必须是 `"string"`！

### Roll Up (上卷)

聚合到层级中更高的级别：

```json
{
  "type": "roll_up",
  "params": {
    "cube_name": "MyCube_DR_v1_city",
    "target_cube_name": "MyCube_RO_v1_country",
    "alias": "v1",
    "property_name": "country",
    "value_path": ["*"],
    "value_type": ["string"]
  }
}
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `cube_name` | string | 上一步输出的 Cube 名称 |
| `target_cube_name` | string | 本步输出的 Cube 名称 |
| `alias` | string | 操作对象的别名 |
| `property_name` | string | 更高层级的属性名 |
| `value_path` | array | 路径数组，**第一项必须是 `"*"`** |
| `value_type` | array | 值类型数组，**第一项必须是 `"string"`** |

### target_cube_name 命名规范

| 算子 | 简写 | 命名模板 |
|------|------|----------|
| Drill Down | DR | `{cube}_DR_{alias}_{property}` |
| Roll Up | RO | `{cube}_RO_{alias}_{property}` |
| Join | JOIN | `{left_cube}_JOIN_{right_cube}` |

---

## 3. View 视图定义

**⚠️ 注意:** `view` 是 query_config 的顶层 key，不是 input 的子字段！

```json
{
  "view": {
    "cube_name": "MyCube_DR_v1_city",
    "calculations": [
      {
        "operator": "/",
        "op1_alias": "v1",
        "op1_property": "visit_num",
        "op2_alias": "v1",
        "op2_property": "visit_all_num",
        "col_name": "v1.visit_num / v1.visit_all_num"
      }
    ],
    "sort_by": [
      {"type": "dim", "method": "", "alias": "v1", "property_name": "city", "order": "asc"},
      {"type": "measure", "method": "COUNT", "alias": "v0", "property_name": "id", "order": "desc"},
      {"type": "calculation", "method": "", "alias": "", "property_name": "v1.visit_num / v1.visit_all_num", "order": "desc"}
    ],
    "top_n": 10
  }
}
```

### 参数详解

| 参数 | 类型 | 说明 |
|------|------|------|
| `cube_name` | string | 要查看的最终 Cube 名称 |
| `calculations` | array | 额外计算列表 (可选) |
| `sort_by` | array | 排序条件列表 (可选，但如果存在则不能为空) |
| `top_n` | integer | 取前 n 条数据，`-1` 表示取所有数据 |

### calculations 字段说明

用于对维度进行额外计算 (支持 `+`, `-`, `*`, `/`)：

```json
"calculations": [
  {
    "operator": "/",
    "op1_alias": "v1",
    "op1_property": "visit_num",
    "op2_alias": "v1",
    "op2_property": "visit_all_num",
    "col_name": "v1.visit_num / v1.visit_all_num"
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `operator` | string | 运算符: `+`, `-`, `*`, `/` |
| `op1_alias` | string | 第一操作数的 alias |
| `op1_property` | string | 第一操作数的属性名 |
| `op2_alias` | string | 第二操作数的 alias |
| `op2_property` | string | 第二操作数的属性名 |
| `col_name` | string | 计算结果的列名 |

**⚠️ 重要规则:**
- `calculations` 中的每个计算，其 `op1_alias.op1_property` 和 `op2_alias.op2_property` 必须在 `internal` 中分别做过 `drill_down` 操作
- **主动分析比例:** 当查询结果同时包含"部分数量"和"总数量"时 (如成功次数和总次数)，应主动使用 calculations 计算比例

### sort_by 字段说明

| 字段 | 类型 | 说明 | 可选值 |
|------|------|------|--------|
| `type` | string | 排序类型 | `"dim"` (维度), `"measure"` (度量), `"calculation"` (计算结果) |
| `method` | string | 聚合方法 | type="dim"或"calculation"时为空`""`；type="measure"时为`COUNT`/`SUM`/`MIN`/`MAX` |
| `alias` | string | 排序对象的别名 | type="calculation"时为空`""` |
| `property_name` | string | 排序属性名/计算列名 | |
| `order` | string | 排序方向 | `"asc"` (升序) 或 `"desc"` (降序) |

### sort_by 和 top_n 约束规则

- ✅ `sort_by` 和 `top_n` 必须**同时存在或同时不存在**
- ✅ 如果 `top_n` 存在 (即使是 -1)，`sort_by` **不能**为空列表 `[]`
- ✅ 可以不写 `sort_by` 和 `top_n` 表示不排序
- ❌ **禁止**使用 `"sort_by": []` (空列表)

### sort_by 依赖规则

- ✅ `type = "dim"` 时，`alias` + `property_name` 必须在 `internal` 中做过 `drill_down`
- ✅ `type = "measure"` 时，`alias` + `property_name` + `method` 必须在 `input` 的 measure 中存在
- ✅ `type = "calculation"` 时，`property_name` 必须是 `calculations` 中定义的 `col_name`

---

## 完整示例

**问题:** 找出访问比例最高的前 10 个地区 (人口 > 20 岁)

```bash
curl -X POST http://127.0.0.1:8073/api/cube/query \
  -H "Content-Type: application/json" \
  -d '{
  "input": {
    "type": "source",
    "params": {
      "cube_name": "PopulationByPlace",
      "sample_size": -1,
      "vertices": [
        {"alias": "v0", "label": "person"},
        {"alias": "v1", "label": "place"}
      ],
      "edges": [
        {"alias": "e0", "label": "person_isLocatedIn_place", "source": "v0", "target": "v1"}
      ],
      "constraints": [
        {
          "alias": "v0",
          "type": "vertex",
          "constraint": [
            {"property": "age", "operator": ">", "value": "20", "value_type": "int32"}
          ]
        }
      ],
      "dimension": [
        {
          "alias": "v1",
          "name": "name",
          "type": "vertex",
          "data_type": "string",
          "hierarchy": ["name"]
        },
        {
          "alias": "v1",
          "name": "visit_num",
          "type": "vertex",
          "data_type": "int32",
          "hierarchy": ["visit_num"]
        },
        {
          "alias": "v1",
          "name": "visit_all_num",
          "type": "vertex",
          "data_type": "int32",
          "hierarchy": ["visit_all_num"]
        }
      ],
      "measure": [
        {
          "alias": "v0",
          "property_name": "id",
          "property_type": "vertex",
          "method": "COUNT"
        }
      ]
    }
  },
  "internal": [
    {
      "type": "drill_down",
      "params": {
        "cube_name": "PopulationByPlace",
        "target_cube_name": "PopulationByPlace_DR_v1_name",
        "alias": "v1",
        "property_name": "name",
        "value_path": ["*"],
        "value_type": ["string"]
      }
    },
    {
      "type": "drill_down",
      "params": {
        "cube_name": "PopulationByPlace_DR_v1_name",
        "target_cube_name": "PopulationByPlace_DR_v1_visit_num",
        "alias": "v1",
        "property_name": "visit_num",
        "value_path": ["*"],
        "value_type": ["string"]
      }
    },
    {
      "type": "drill_down",
      "params": {
        "cube_name": "PopulationByPlace_DR_v1_visit_num",
        "target_cube_name": "PopulationByPlace_DR_v1_visit_all_num",
        "alias": "v1",
        "property_name": "visit_all_num",
        "value_path": ["*"],
        "value_type": ["string"]
      }
    }
  ],
  "view": {
    "cube_name": "PopulationByPlace_DR_v1_visit_all_num",
    "calculations": [
      {
        "operator": "/",
        "op1_alias": "v1",
        "op1_property": "visit_num",
        "op2_alias": "v1",
        "op2_property": "visit_all_num",
        "col_name": "v1.visit_num / v1.visit_all_num"
      }
    ],
    "sort_by": [
      {"type": "calculation", "method": "", "alias": "", "property_name": "v1.visit_num / v1.visit_all_num", "order": "desc"}
    ],
    "top_n": 10
  }
}'
```

---

## 决策逻辑指南

### 冷启动 (Cold Start)
如果当前 GraphCube Tree 为空，必须从创建新 Cube (`type="source"`) 开始。

### 元素映射
仔细分析用户问题，映射到具体的图元素：
- 实体 → `vertices`, `edges`
- 筛选 → `constraints`
- 维度 → `dimension` (作为 Drill Down 的目标)
- 指标 → `dimension` (如果属性本身表示统计信息) 或 `measure` (聚合方式: COUNT, SUM 等)

### 实体映射规则
- 问题理解中，每个 entity 可能有若干 schema_mapping，一般按置信度从高到低依次尝试
- 对于每个 entity，一次只能选择一个 schema_mapping

### Drill Down 必选原则
⚠️ **注意:** 仅创建 Cube (`input` 阶段) 只会得到全局聚合值 (Total Measure)。

**必须 Drill Down 的场景:**
1. 问题包含"按...分类"、"每个...的..."或涉及具体维度分布
2. 想了解某个属性的具体值
3. 想了解每个个体在某个属性上的具体值时，必须同时 drill_down 个体标识符 (如 id) 和对应属性

**示例:** 问"各地区用户数"
1. Create Cube (User + Region)
2. Drill Down (Region)

不 Drill Down 只能得到用户总数！

### 属性作为统计值的处理
如果存在一个属性本身已直接表示用户问题所关心的**统计值或度量结果** (如 `NumTstTakr` 表示"考生人数")：
1. 将其作为**维度** (dimension) 加入 query_config 的 `input`
2. 在 `internal` 中对该属性执行 `drill_down` 以获取具体值
3. **不要**对其使用 COUNT (结果一定是 1)

---

## 执行流程

1. 结合 Schema/问题理解/Cube Tree/历史对话，明确需要分析的数据视角
2. 根据问题理解将关键实体映射到 schema 中的属性
3. 规划 `query_config`
4. **必须调用 query 工具** (至少一次)
5. 阅读工具返回的真实数据，输出 2-3 条结论，每条结论都要引用返回值里的具体数字并解释业务含义

---

## 严格约束清单

### 结构约束
- ✅ `internal` 和 `view` 是 query_config 的顶层 key (不是 input 的子字段)
- ✅ `internal`、`view` 字段不得省略，即使为空也要提供 (internal 可为空列表 `[]`)

### vertices/edges 约束
- ✅ `vertices` 最多包含 **3 个节点**
- ✅ `edges` 最多包含 **3 条边**
- ✅ `edges` 中的 `source`/`target` 必须匹配 `vertices` 中的 `alias`

### constraints 约束
- ✅ `constraints.alias` 必须在 `vertices` 或 `edges` 中存在
- ✅ `constraints.property` 必须是属性名，**禁止**使用常量或表达式
- ✅ `constraints.value` 中引用属性时必须使用 `'alias.property'` 格式
- ✅ `constraints.value` 中**禁止**使用其他 alias 的属性
- ✅ `constraints.operator` 支持: `>`, `>=`, `<`, `<=`, `=`, `is null`, `is not null`
- ✅ `is null`/`is not null` 时，`value` 必须为 `""`，`value_type` 必须为 `"string"`

### internal 约束
- ✅ `internal` 中 `value_path` 的第一项必须是 `"*"`
- ✅ `internal` 中 `value_type` 的第一项必须是 `"string"`

### view 约束
- ✅ `sort_by` 和 `top_n` 必须同时存在或同时不存在
- ✅ 如果 `top_n` 存在 (即使是 -1)，`sort_by` **不能**为空列表
- ✅ `sort_by` 中 `type = "dim"` 时，alias + property_name 必须在 internal 中做过 drill_down
- ✅ `sort_by` 中 `type = "measure"` 时，alias + property_name + method 必须在 input 的 measure 中存在
- ✅ `sort_by` 中 `type = "calculation"` 时，property_name 必须是 calculations 中定义的 col_name

### calculations 约束
- ✅ calculations 中每个计算的 op1 和 op2 都必须在 internal 中做过 drill_down
- ✅ 主动分析比例：当结果包含"部分数量"和"总数量"时，应使用 calculations 计算比例

### 执行约束
- ❌ **禁止**直接重复使用之前生成的 cube，必须重新 source 构造新的 cube
- ❌ **禁止**并行调用工具，必须串行调用
- ✅ 必须调用 query 工具 (至少一次)

---

## 输出格式规范

1. **[实体映射]**: 描述本次 query 中使用的实体映射结果，为每个实体选择一个 schema_mapping 并说明原因
2. **[分析前置结果]**: 如有前置问题的回答，分析是否能帮助回答当前问题
3. **[规划]**: 概述本次 query 准备创建的 cube，操作的算子/视图
4. **[query 工具调用]**: 描述 query_config 及调用结果
5. **[结果解读]**: 引用工具返回的字段/指标，列出关键洞察，说明业务意义。如基于子集分析，需说明子集范围和完整数据情况
