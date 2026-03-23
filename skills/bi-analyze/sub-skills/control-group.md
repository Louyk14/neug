# 对照分析 (Control Group Analysis)

> 本文件是 SKILL.md Step 4.5 的详细补充指南。核心规则（判断标准、对照类型、必须包含差异率）已在 SKILL.md 中内联，此处提供详细的对照设计方法和可视化代码。

通过设计对照组来加强结论的可靠性。核心思路：为每个子问题构造一个对照子问题，通过对比两者的结果得出更准确的结论。

## 1. 判断是否需要对照组

| 场景 | 是否需要 | 原因 |
|------|----------|------|
| 绝对值结论 | 需要 | 需要参照系才能判断好坏 |
| 差异声明 | 需要 | 需要对比才能说明差异 |
| 效果评估 | 需要 | 需要基准来评估效果 |
| 纯排名 | 不需要 | 排名本身已有对比 |
| 纯占比 | 不需要 | 占比本身是相对值 |
| 描述性统计 | 不需要 | 仅需客观描述 |

## 2. 对照组类型与设计

| 类型 | 原问题示例 | 对照问题构造 | 对比维度 |
|------|-----------|-------------|----------|
| 群体对照 | "Top10学校的平均成绩" | "所有学校的平均成绩" | 目标群体 vs 总体 |
| 时间对照 | "2024年招生人数" | "2023年招生人数" | 今年 vs 去年 |
| 条件对照 | "城市学校升学率" | "农村学校升学率" | 有条件 vs 无条件 |
| 基准对照 | "A学校师生比" | "行业平均师生比" | 个体 vs 标准 |
| 分组对照 | "大型学校成绩" | "小型学校成绩" | 组A vs 组B |

**设计原则**:
- 对照组与原始组使用**相同的指标**
- 只改变一个变量（被对比的维度）
- 对照问题应该能用同样的 GraphCube 查询方式获取

## 3. 执行流程

```
原始子问题结果
    │
    ▼
1. 构造对照问题 → 确定对照类型和变量
    │
    ▼
2. 构建对照查询 → 使用 graphcube skill
    │
    ▼
3. 执行查询 → 调用 GraphCube API
    │
    ▼
4. 提取结论 → 从对照结果中提取数值
    │
    ▼
5. 对比计算 → 绝对差异 + 相对差异率
    │
    ▼
6. 绘制对比图 → Python 可视化
    │
    ▼
更准确的结论
```

## 4. 对比计算

```
绝对差异 = 实验组值 - 对照组值
相对差异率 = (实验组值 - 对照组值) / 对照组值 × 100%
```

## 5. 对比可视化

推荐使用**分组柱状图**进行对比展示：

```python
import matplotlib.pyplot as plt
import numpy as np

categories = ['指标1', '指标2', '指标3']
experimental = [85, 92, 78]
control = [70, 80, 65]

x = np.arange(len(categories))
width = 0.35

fig, ax = plt.subplots(figsize=(10, 6))
bars1 = ax.bar(x - width/2, experimental, width, label='实验组')
bars2 = ax.bar(x + width/2, control, width, label='对照组')

ax.set_ylabel('数值')
ax.set_title('实验组 vs 对照组对比')
ax.set_xticks(x)
ax.set_xticklabels(categories)
ax.legend()
ax.bar_label(bars1, padding=3)
ax.bar_label(bars2, padding=3)

plt.tight_layout()
plt.savefig('images/subq_X_Y_comparison.png', dpi=150, bbox_inches='tight')
plt.close()
```

## 6. 报告格式

```markdown
#### 对照验证

**对照设计**:
- 原始问题: [描述]
- 对照问题: [描述]
- 对照类型: [群体/时间/条件/基准/分组]

**对比分析**:

| 指标 | 原始组 | 对照组 | 差异 | 差异率 |
|------|--------|--------|------|--------|
| 指标A | 85 | 70 | +15 | +21.4% |

![对照对比](images/subq_X_Y_comparison.png)

**验证结论**: [基于对比得出的更准确结论]
```
