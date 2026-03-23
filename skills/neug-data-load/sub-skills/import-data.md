# 数据导入详细指南

> 本文件是 SKILL.md Step 3 的补充指南，提供导入 API 细节和错误处理方法。

## 1. 导入前检查

确认以下条件：
- GraphCube Server 正在运行（`curl http://127.0.0.1:8073/api/cube/schema` 应有响应）
- `schema.json` 已生成在 CSV 文件夹中且已获得用户确认
- CSV 文件夹路径是绝对路径

## 2. 调用导入接口

```bash
curl -X POST http://127.0.0.1:8073/api/data/import \
  -H "Content-Type: application/json" \
  -d '{"dataPath": "/absolute/path/to/csv/folder"}'
```

**唯一参数**: `dataPath` — CSV 文件夹的绝对路径，服务端会在此目录下读取 `schema.json` 和所有 CSV 文件。

## 3. 等待策略

导入耗时取决于数据量，无法预估：

| 数据规模 | 预期耗时 |
|----------|----------|
| < 1 万行 | 几十秒 |
| 1-100 万行 | 几分钟 |
| 100 万-1000 万行 | 十几分钟 |
| > 1000 万行 | 可能超过 30 分钟 |

**执行策略**:
1. 使用 `block_until_ms: 0` 立即将命令放入后台
2. 定期轮询终端输出，检查是否完成
3. 轮询间隔建议：前 2 分钟每 30 秒检查一次，之后每 60 秒检查一次
4. **禁止**在导入过程中调用任何其他 GraphCube API

## 4. 响应处理

### 成功

```json
{"success": true, "message": "Data imported successfully"}
```

告知用户：数据导入完成，可以使用 `graphcube` skill 进行查询分析。

### 失败

```json
{"success": false, "message": "Error: ..."}
```

### 常见错误及排查

| 错误信息 | 可能原因 | 排查方法 |
|----------|----------|----------|
| 路径不存在 | dataPath 错误 | 检查路径拼写和是否为绝对路径 |
| schema.json not found | schema 文件缺失或位置错误 | 确认 schema.json 在 dataPath 目录下 |
| CSV file not found | schema 中 file_path 与实际文件名不匹配 | 对比 schema.json 中的 file_path 和实际文件名 |
| Column mismatch | CSV 列与 schema properties 不一致 | 检查 CSV 表头和 schema 中 properties 的对应关系 |
| Type conversion error | 数据类型推断错误 | 将问题列的类型改为 STRING 重试 |

### 错误修复流程

```
导入失败
    │
    ▼
分析错误信息
    │
    ├── 路径/文件问题 → 修正路径或文件名 → 重新导入
    ├── Schema 格式问题 → 修改 schema.json → 请用户重新确认 → 重新导入
    └── 数据类型问题 → 修改 schema.json 中的 type → 请用户确认 → 重新导入
```
