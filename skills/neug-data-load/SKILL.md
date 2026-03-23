---
name: neug-data-load
description: "Load graph data into NeuG/GraphCube. Scan a CSV folder, auto-detect vertex and edge files, generate schema.json, and import data via GraphCube server. Use when the user wants to import data, load CSV files into a graph database, generate a schema, or set up a new NeuG dataset."
---

# NeuG 数据加载

从用户提供的 CSV 文件夹自动生成 schema.json 并导入 GraphCube。

## 流程总览

```
用户提供 CSV 文件夹路径
    │
    ▼
Step 1: Schema 生成 → 扫描 CSV → 分类节点/边 → 生成 schema.json
    │
    ▼
Step 2: 用户确认 → 展示 schema 摘要 → 等待用户确认/修改
    │
    ▼
Step 3: 数据导入 → 调用 /api/data/import → 等待完成
```

---

## Step 1: Schema 生成

详细指南见 [sub-skills/schema-gen.md](sub-skills/schema-gen.md)。

### 1.0 检查已有 Schema 描述文件

在扫描 CSV 之前，**先检查目录下是否已存在描述 schema 的文件**（如 `schema.json`、`schema.yaml`、`schema.txt`、`README.md`、`data_dictionary.*`、`metadata.*` 等）。

```bash
ls <folder>/schema.* <folder>/metadata.* <folder>/README* <folder>/data_dictionary* 2>/dev/null
```

**如果存在**:
1. 读取该文件内容，提取其中的节点/边定义、属性描述、数据类型等信息
2. 将这些信息作为**优先参考**，指导后续的分类、label 命名、属性类型标注和边端点推断
3. 如果已有文件已经是合法的 `schema.json` 格式，直接复用并跳到 Step 2 用户确认

**如果不存在**: 正常进入 1.1 扫描流程，完全从 CSV 推断。

### 1.1 扫描 CSV 文件

列出用户指定文件夹中的所有 `.csv` 文件，读取每个文件的表头（第一行）。

```bash
# 列出所有 CSV 文件
ls <folder>/*.csv

# 读取每个文件的表头
head -1 <folder>/<file>.csv
```

### 1.2 分类：节点 vs 边

根据表头列名判断文件类型：

| 判断条件 | 文件类型 | 依据 |
|----------|----------|------|
| 表头包含 `src` 和 `dst` 列 | **边文件** | 边文件固定使用 `src,dst` 表示起止节点 |
| 表头包含 `id` 列（且无 `src,dst`） | **节点文件** | 节点文件以 `id` 作为主键 |
| 都不满足 | **跳过** | 在报告中提示用户手动处理 |

### 1.3 推断边的 from_type / to_type

边文件的 label 和端点类型从文件名推断：

**命名模式**: 边文件名通常是 `<from>_<relationship>_<to>.csv`，其中 `<from>` 和 `<to>` 与节点文件名（即节点 label）对应。

**推断规则**:
1. 将文件名（去掉 `.csv`）作为边的 `label`
2. 从已识别的节点 label 列表中，找到文件名中匹配的 from_type 和 to_type
3. 匹配策略：从文件名的开头和结尾分别尝试匹配节点 label（大小写不敏感）

**示例**:
```
节点 labels: [Author, Publication, Organization]
边文件名: author_authored_publication.csv
→ label: "author_authored_publication"
→ from_type: "Author" (匹配文件名开头)
→ to_type: "Publication" (匹配文件名结尾)
```

**⚠️ 无法推断时**: 列出无法匹配的边文件，请用户手动指定 from_type 和 to_type。

### 1.4 推断属性类型

读取 CSV 文件的前 100 行数据，推断每个列的数据类型：

| 推断规则 | 类型 |
|----------|------|
| 所有值均为整数 | `INT64` |
| 所有值均为浮点数 | `INT64` 或 `DOUBLE`（根据精度判断） |
| 其他 | `STRING` |

### 1.5 生成 schema.json

将结果写入 `<folder>/schema.json`，格式如下：

```json
{
  "vertex_tables": [
    {
      "label": "Author",
      "file_path": "author.csv",
      "properties": [
        {"name": "id", "type": "INT64"},
        {"name": "fullName", "type": "STRING"}
      ]
    }
  ],
  "edge_tables": [
    {
      "label": "author_authored_publication",
      "from_type": "Author",
      "to_type": "Publication",
      "file_path": "author_authored_publication.csv",
      "properties": [
        {"name": "src", "type": "INT64"},
        {"name": "dst", "type": "INT64"}
      ]
    }
  ]
}
```

**⚠️ 关键格式规则**:
- `file_path` 只写文件名（不含目录路径），因为导入时会通过 `dataPath` 指定目录
- `label` 对于节点取文件名（首字母大写），对于边取完整文件名（不含 `.csv`）
- 节点的 `id` 字段类型固定为 `INT64`
- 边的 `src`/`dst` 字段类型固定为 `INT64`
- 生成的文件名**必须**是 `schema.json`

---

## Step 2: 用户确认

生成 schema.json 后，向用户展示摘要并等待确认：

**展示内容**:
1. 识别到的节点文件数量和列表（label + 属性数量）
2. 识别到的边文件数量和列表（label + from→to）
3. 无法识别的文件（如有）
4. schema.json 的保存路径

**等待用户**:
- 确认无误 → 进入 Step 3
- 需要修改 → 按用户要求修改 schema.json 后重新确认

---

## Step 3: 数据导入

详细指南见 [sub-skills/import-data.md](sub-skills/import-data.md)。

### 3.1 调用导入接口

```bash
curl -X POST http://127.0.0.1:8073/api/data/import \
  -H "Content-Type: application/json" \
  -d '{"dataPath": "<folder的绝对路径>"}'
```

**参数**: `dataPath` — CSV 文件夹的**绝对路径**（schema.json 所在目录）。

### 3.2 等待导入完成

**⚠️ 导入耗时不确定**: 根据数据量，可能几十秒到几十分钟不等。**必须耐心等待**，不要中断。

- 导入过程中**禁止**调用其他 GraphCube API（`/api/cube/schema`、`/api/cube/query`、`/api/data/open`）
- 设置充足的超时时间（建议 `block_until_ms` 至少 1800000，即 30 分钟；数据量大时可设更长）
- 如果命令进入后台，通过轮询终端输出监控进度

### 3.3 验证导入结果

导入接口返回：

```json
{"success": true, "message": "..."}   // 成功
{"success": false, "message": "..."}  // 失败，message 包含错误原因
```

- **成功**: 告知用户数据已导入，可以开始使用 GraphCube 进行查询分析
- **失败**: 展示错误信息，协助用户排查（常见问题：路径错误、schema.json 格式错误、CSV 数据不匹配 schema）

---

## 依赖

| 服务 | 地址 | 健康检查 |
|------|------|----------|
| GraphCube Server | `http://127.0.0.1:8073` | `curl http://127.0.0.1:8073/api/cube/schema` |

## Sub-Skills 参考

| 文件 | 内容 | 何时查阅 |
|------|------|----------|
| [sub-skills/schema-gen.md](sub-skills/schema-gen.md) | CSV 扫描和 schema.json 生成的详细逻辑 | Step 1 遇到复杂 CSV 结构时 |
| [sub-skills/import-data.md](sub-skills/import-data.md) | 数据导入 API 细节和错误处理 | Step 3 导入出错时 |
