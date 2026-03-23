# Schema 生成详细指南

> 本文件是 SKILL.md Step 1 的补充指南，提供 CSV 扫描和 schema.json 生成的详细逻辑。

## 0. 检查已有 Schema 描述文件

在扫描 CSV 之前，先检查目录下是否已有描述 schema 的文件：

```bash
ls <folder>/schema.* <folder>/metadata.* <folder>/README* <folder>/data_dictionary* 2>/dev/null
```

**已有文件的利用方式**:

| 文件类型 | 利用方式 |
|----------|----------|
| 合法 `schema.json`（含 vertex_tables/edge_tables） | 直接复用，跳到用户确认 |
| 其他结构化描述（yaml/json/txt） | 提取 label 名、属性名和类型作为优先参考 |
| README / 数据字典 | 提取语义信息辅助 label 命名和类型推断 |

**优先级**: 已有文件中的信息 > 自动推断。具体地：
- 已有文件指定了 label 名 → 使用已有名称，不从文件名推断
- 已有文件指定了属性类型 → 使用已有类型，不从数据采样推断
- 已有文件指定了边的端点 → 使用已有端点，不从文件名匹配推断
- 已有文件未覆盖的部分 → 回退到自动推断

## 1. CSV 文件扫描

```bash
ls <folder>/*.csv
```

对每个 CSV 文件读取表头：

```bash
head -1 <folder>/<file>.csv
```

## 2. 节点 vs 边 分类逻辑

**核心判断**: 检查表头是否同时包含 `src` 和 `dst` 列。

```
表头包含 src 和 dst？
    ├── 是 → 边文件
    └── 否 → 表头包含 id？
              ├── 是 → 节点文件
              └── 否 → 未知，提示用户
```

**边文件的额外属性**: 边文件除了 `src,dst` 外可能有其他列（如 `weight`, `type`），这些列也需要记录为边的 properties。

## 3. 节点 Label 推断

节点 label 从文件名推断，**首字母大写**：

| 文件名 | label |
|--------|-------|
| `author.csv` | `Author` |
| `publication.csv` | `Publication` |
| `my_entity.csv` | `My_entity`（只首字母大写） |

规则：取文件名（去掉 `.csv`），将首字母转为大写。

## 4. 边端点推断（from_type / to_type）

边文件名通常遵循 `<from>_<relation>_<to>.csv` 模式。

### 匹配算法

```
输入: 
  - edge_filename: 边文件名（不含 .csv）
  - vertex_labels: 所有节点 label 列表

步骤:
1. 对每个 vertex_label，检查 edge_filename 是否以该 label 开头（大小写不敏感）
   → 匹配到的是 from_type
2. 对每个 vertex_label，检查 edge_filename 是否以该 label 结尾（大小写不敏感）
   → 匹配到的是 to_type
3. 如果 from_type 和 to_type 都匹配到 → 成功
4. 如果只匹配到一个或都没匹配到 → 标记为"需用户确认"
```

### 自引用边

from_type 和 to_type 可以是同一个 label，例如：
- `organization_IsParentOf_organization.csv` → from: Organization, to: Organization

### 歧义处理

如果多个 label 都能匹配文件名开头或结尾，选择最长匹配：
- labels: `[Project, ProjectGroup]`
- filename: `projectGroup_contains_project`
- from_type: `ProjectGroup`（更长匹配优先）

## 5. 属性类型推断

读取 CSV 前 100 行（或全部行如果不足 100 行），对每列进行类型推断：

### 推断规则

```
对每一列:
1. 跳过空值和 null
2. 尝试将所有非空值解析为整数
   → 全部成功 → INT64
3. 尝试将所有非空值解析为浮点数
   → 全部成功 → DOUBLE
4. 否则 → STRING
```

### 特殊情况

| 情况 | 处理 |
|------|------|
| 列全为空 | 标记为 `STRING` |
| 混合整数和浮点数 | 标记为 `DOUBLE` |
| 包含科学计数法 | 标记为 `DOUBLE` |
| 日期格式字符串 | 标记为 `STRING`（NeuG 不区分日期类型） |

### 固定类型

| 列名 | 固定类型 | 原因 |
|------|----------|------|
| `id` | `INT64` | 节点主键 |
| `src` | `INT64` | 边起点 ID |
| `dst` | `INT64` | 边终点 ID |

## 6. schema.json 生成规范

### 完整结构

```json
{
  "vertex_tables": [
    {
      "label": "<首字母大写的文件名>",
      "file_path": "<文件名.csv>",
      "properties": [
        {"name": "<列名>", "type": "<类型>"}
      ]
    }
  ],
  "edge_tables": [
    {
      "label": "<完整文件名，不含.csv>",
      "from_type": "<起点节点的label>",
      "to_type": "<终点节点的label>",
      "file_path": "<文件名.csv>",
      "properties": [
        {"name": "src", "type": "INT64"},
        {"name": "dst", "type": "INT64"}
      ]
    }
  ]
}
```

### 关键规则

1. `file_path` 只含文件名，不含目录路径
2. `properties` 中列的顺序与 CSV 表头顺序一致
3. `vertex_tables` 和 `edge_tables` 按 label 字母序排列
4. 输出文件名固定为 `schema.json`，保存在 CSV 文件夹中

## 7. 展示摘要格式

schema.json 生成后，向用户展示：

```
Schema 生成完毕，保存在 <path>/schema.json

节点 (N 个):
  - Author (9 个属性): id, fullName, name, surname, ...
  - Publication (12 个属性): id, label, type, year, ...

边 (M 个):
  - author_authored_publication: Author → Publication
  - funding_for_project: Funding → Project

⚠️ 无法自动识别的文件: (如有)
  - unknown_file.csv — 请指定类型和端点

请确认 schema 是否正确，确认后将开始导入数据。
```
