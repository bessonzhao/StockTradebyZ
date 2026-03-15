# AI 复评模块

## 1. 模块概述

AI 复评模块是 AgentTrader 项目的核心组成部分，负责使用大语言模型（LLM）对量化初选产生的候选股票进行图表分析和评分。该模块结合了计算机视觉和自然语言处理技术，能够模拟专业交易员的分析思路，对股票的波段爆发潜力进行评估。

### 1.1 核心功能

- **图表分析**：自动识别股票 K 线图的趋势、位置、量价关系和历史异动
- **评分系统**：基于多维度指标对股票进行综合评分
- **信号生成**：生成主升启动、跌后反弹或出货风险等信号
- **结果汇总**：生成最终的股票推荐列表

### 1.2 技术栈

- **大语言模型**：Google Gemini API
- **图像处理**：内置图片处理功能
- **配置管理**：YAML
- **输出格式**：JSON

## 2. 架构设计

### 2.1 类结构

```
┌─────────────────────┐
│   BaseReviewer      │
├─────────────────────┤
│ + load_prompt()     │
│ + load_candidates() │
│ + find_chart_images() │
│ + extract_json()    │
│ + review_stock()    │  # 抽象方法，子类实现
│ + generate_suggestion() │
│ + run()             │
└─────────────────────┘
          ▲
          │
          │ 继承
          │
┌─────────────────────┐
│  GeminiReviewer     │
├─────────────────────┤
│ + __init__()        │
│ + image_to_part()   │
│ + review_stock()    │  # 实现父类抽象方法
└─────────────────────┘
```

### 2.2 核心组件

| 组件 | 作用 |
|------|------|
| BaseReviewer | 提供 LLM 图表分析的基础架构，包括配置加载、候选股票读取、图表查找、结果汇总等功能 |
| GeminiReviewer | 继承自 BaseReviewer，实现了使用 Google Gemini API 进行图表分析的具体逻辑 |
| prompt.md | 定义了 AI 复评的提示词模板，包括分析框架、评分标准和输出格式 |

### 2.3 工作流程

1. **加载配置**：读取 gemini_review.yaml 配置文件
2. **读取候选股票**：从 candidates_latest.json 读取候选股票列表
3. **查找 K 线图**：根据股票代码和日期查找对应的 K 线图
4. **生成提示词**：加载并使用 prompt.md 作为系统提示词
5. **调用 LLM**：使用 Gemini API 对每只股票的 K 线图进行分析
6. **解析结果**：从 LLM 响应中提取 JSON 格式的评分结果
7. **生成推荐**：根据评分结果生成最终的股票推荐列表
8. **输出结果**：将单股评分和汇总推荐保存到指定目录

## 3. 详细工作流程

### 3.1 配置加载

AI 复评模块使用 YAML 配置文件，默认路径为 `config/gemini_review.yaml`。配置文件包含以下主要参数：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| model | Gemini 模型名称 | models/gemini-1.5-flash |
| request_delay | 调用间隔（秒） | 2 |
| skip_existing | 是否跳过已存在的结果 | true |
| suggest_min_score | 推荐分数门槛 | 8.0 |
| kline_dir | K 线图目录 | data/kline |
| candidates_path | 候选股票路径 | data/candidates/candidates_latest.json |
| out_dir | 输出目录 | data/review |

### 3.2 候选股票读取

模块从 `data/candidates/candidates_latest.json` 文件中读取候选股票列表，获取股票代码、选股日期等信息。

### 3.3 图表查找

根据股票代码和选股日期，在 `data/kline/日期` 目录下查找对应的 K 线图文件，支持 JPG 和 PNG 格式。

### 3.4 提示词生成

使用 `agent/prompt.md` 作为系统提示词，指导 Gemini 进行图表分析。提示词包含以下核心内容：

1. **任务边界**：明确分析范围和禁止行为
2. **分析顺序**：趋势结构、价格位置结构、量价行为、前期建仓异动
3. **权重分配**：各维度评分的权重
4. **信号类型**：主升启动、跌后反弹、出货风险
5. **判定规则**：评分标准和结果判定
6. **强制推理步骤**：必须完成的推理过程
7. **输出格式**：严格的 JSON 格式

### 3.5 LLM 调用

1. **图片处理**：将 K 线图转换为 Gemini 可接受的 Part 对象
2. **生成请求**：构建包含图片和文字提示的请求
3. **调用 API**：使用 Gemini API 生成内容
4. **解析响应**：从响应中提取 JSON 格式的评分结果

### 3.6 结果解析

从 Gemini 响应中提取 JSON 格式的评分结果，包括：

- 各维度的评分和推理
- 总评分
- 信号类型
- 判定结果（PASS/WATCH/FAIL）
- 交易员点评

### 3.7 结果汇总

1. **筛选推荐股票**：根据评分门槛筛选出推荐股票
2. **排序**：按总评分降序排列
3. **生成汇总文件**：包含推荐股票列表和未达门槛的股票列表

### 3.8 结果输出

- **单股评分文件**：`data/review/日期/代码.json`
- **汇总推荐文件**：`data/review/日期/suggestion.json`

## 4. 分析框架

### 4.1 分析维度

| 维度 | 权重 | 评分标准 |
|------|------|----------|
| 趋势结构（Trend Structure） | 0.20 | 评估均线结构和趋势强度 |
| 价格位置结构（Price Position） | 0.20 | 评估当前价格在趋势中的位置 |
| 量价行为（Volume Behavior） | 0.30 | 评估量价配合情况 |
| 前期建仓异动（Previous Abnormal Move） | 0.30 | 评估主力建仓痕迹 |

### 4.2 评分标准

每个维度的评分范围为 1-5 分，具体标准如下：

#### 4.2.1 趋势结构

| 分数 | 标准 |
|------|------|
| 5 | 均线刚进入多头结构，短期均线上穿长期均线 |
| 4 | 均线处于健康上升趋势，多头排列已经形成 |
| 3 | 均线结构偏多，但不够理想，价格频繁跌破均线 |
| 2 | 趋势偏弱或较为混乱，均线频繁交叉 |
| 1 | 明显弱势，空头排列，均线向下 |

#### 4.2.2 价格位置结构

| 分数 | 标准 |
|------|------|
| 5 | 处于中低位刚突破，上方仍有明显空间 |
| 4 | 处于中位突破区，正在向前高推进 |
| 3 | 处于接近前高区，临近前期压力位 |
| 2 | 处于接近历史高位或长期压力区，上方空间有限 |
| 1 | 处于明显高位或过热区，已经大幅上涨 |

#### 4.2.3 量价行为

| 分数 | 标准 |
|------|------|
| 5 | 上涨阶段明显放量，回调明显缩量，没有放量大阴线 |
| 4 | 整体量价关系健康，上涨有放量，回调基本缩量 |
| 3 | 量价中性，上涨与回调的量能差异不大 |
| 2 | 量价偏弱，上涨没有明显放量，回调不缩量 |
| 1 | 量价明显恶化，出现放量大阴线，存在出货迹象 |

#### 4.2.4 前期建仓异动

| 分数 | 标准 |
|------|------|
| 5 | 存在非常明确的主力建仓痕迹，出现异常放量的中大阳线 |
| 4 | 存在明显放量阳线，但突破结构不够明显 |
| 3 | 存在一定放量上涨，但放量不够突出 |
| 2 | 只有普通上涨或轻微放量 |
| 1 | 已经完成主升浪，异动涨幅过大，存在出货迹象 |

### 4.3 信号类型

| 信号类型 | 含义 |
|----------|------|
| trend_start | 主升启动，适合买入 |
| rebound | 跌后反弹，谨慎参与 |
| distribution_risk | 出货风险，谨慎持有或卖出 |

### 4.4 判定规则

| 判定结果 | 条件 |
|----------|------|
| PASS | total_score ≥ 4.0 |
| WATCH | 3.2 ≤ total_score < 4.0 |
| FAIL | total_score < 3.2 |

**特殊规则**：如果量价行为评分为 1，则无论总评分如何，判定结果均为 FAIL。

## 5. 输出格式

### 5.1 单股评分文件

```json
{
  "trend_reasoning": "均线刚进入多头结构，短期均线上穿长期均线，趋势向上",
  "position_reasoning": "处于中低位刚突破，上方仍有明显空间",
  "volume_reasoning": "上涨阶段明显放量，回调明显缩量，量价配合良好",
  "abnormal_move_reasoning": "存在明显的主力建仓痕迹，出现异常放量的中大阳线",
  "signal_reasoning": "符合主升启动的特征",
  "scores": {
    "trend_structure": 5,
    "price_position": 5,
    "volume_behavior": 5,
    "previous_abnormal_move": 5
  },
  "total_score": 5.0,
  "signal_type": "trend_start",
  "verdict": "PASS",
  "comment": "周线趋势向上，量价配合良好，存在明显主力建仓痕迹，上方仍有明显空间，主升启动信号明显"
}
```

### 5.2 汇总推荐文件

```json
{
  "date": "2026-03-15",
  "min_score_threshold": 8.0,
  "total_reviewed": 10,
  "recommendations": [
    {
      "rank": 1,
      "code": "000001.SZ",
      "verdict": "PASS",
      "total_score": 9.5,
      "signal_type": "trend_start",
      "comment": "周线趋势向上，量价配合良好，存在明显主力建仓痕迹，上方仍有明显空间，主升启动信号明显"
    },
    {
      "rank": 2,
      "code": "000002.SZ",
      "verdict": "PASS",
      "total_score": 9.0,
      "signal_type": "trend_start",
      "comment": "周线趋势向上，量价配合良好，刚突破平台，主升启动信号明显"
    }
  ],
  "excluded": ["000003.SZ", "000004.SZ"]
}
```

## 6. 使用方法

### 6.1 环境变量设置

AI 复评模块需要设置 Google Gemini API 密钥：

#### Windows PowerShell

```powershell
[Environment]::SetEnvironmentVariable("GEMINI_API_KEY", "你的GeminiApiKey", "User")
```

#### Linux/Mac

```bash
export GEMINI_API_KEY=你的GeminiApiKey
```

### 6.2 运行命令

#### 基本用法

```bash
python agent/gemini_review.py
```

#### 指定配置文件

```bash
python agent/gemini_review.py --config config/gemini_review.yaml
```

#### 配置参数说明

| 参数 | 说明 |
|------|------|
| --config | 配置文件路径 |

## 7. 配置文件详解

### 7.1 gemini_review.yaml

```yaml
# Gemini 模型配置
model: "models/gemini-1.5-flash"  # 模型名称
request_delay: 2               # 调用间隔（秒），防止限流
skip_existing: true            # 是否跳过已存在的结果，支持断点续跑
suggest_min_score: 8.0         # 推荐分数门槛

# 路径配置
kline_dir: "data/kline"        # K 线图目录
candidates_path: "data/candidates/candidates_latest.json"  # 候选股票路径
out_dir: "data/review"         # 输出目录
prompt_path: "agent/prompt.md"  # 提示词模板路径
```

### 7.2 配置调整建议

- **模型选择**：根据需要调整模型，如使用更强大的 `models/gemini-1.5-pro` 但会增加调用成本
- **调用间隔**：如果遇到限流问题，可以适当增加 `request_delay` 参数
- **分数门槛**：根据市场环境调整 `suggest_min_score` 参数，提高或降低推荐标准
- **断点续跑**：设置 `skip_existing: true` 可以在中途停止后继续运行

## 8. 扩展与自定义

### 8.1 添加新的 LLM 支持

1. 继承 `BaseReviewer` 类
2. 实现 `review_stock` 方法，调用新的 LLM API
3. 在 `main` 函数中添加新的 Reviewer 实例化逻辑

### 8.2 自定义提示词

修改 `agent/prompt.md` 文件，可以：

- 调整分析维度和权重
- 修改评分标准
- 调整输出格式
- 自定义交易员点评模板

### 8.3 调整评分权重

在 `prompt.md` 文件中修改权重分配：

```
# 三、权重
trend_structure：0.20
price_position：0.20
volume_behavior：0.30
previous_abnormal_move：0.30
```

### 8.4 添加新的信号类型

在 `prompt.md` 文件中添加新的信号类型：

```
# 五、信号类型

必须且只能选择一个：
trend_start
rebound
distribution_risk
new_signal_type  # 新信号类型
```

## 9. 性能优化

### 9.1 断点续跑

设置 `skip_existing: true` 可以跳过已存在的结果，支持断点续跑，提高效率。

### 9.2 并行化潜力

AI 复评模块设计支持并行化处理，可以通过以下方式优化性能：

- 使用多线程/多进程并行调用 LLM API
- 调整 `request_delay` 参数，平衡调用频率和限流风险

### 9.3 模型选择

根据需要选择合适的模型：
- `models/gemini-1.5-flash`：速度快，成本低，适合大规模处理
- `models/gemini-1.5-pro`：更强大，分析更准确，但速度较慢，成本较高

## 10. 常见问题与解决方案

### 10.1 Gemini API 调用失败

**问题**：运行 `gemini_review` 时出现 API 调用失败

**解决方案**：
- 检查 `GEMINI_API_KEY` 环境变量是否正确设置
- 提高 `request_delay` 参数，避免限流
- 检查网络连接是否正常
- 确认 API 密钥是否有足够的调用额度

### 10.2 缺少 K 线图

**问题**：提示缺少日线图，跳过股票

**解决方案**：
- 确保 `dashboard/export_kline_charts.py` 已成功运行
- 检查 `data/kline/日期` 目录下是否存在对应的 K 线图文件
- 确认股票代码和日期是否正确

### 10.3 JSON 解析失败

**问题**：无法从 LLM 响应中提取 JSON

**解决方案**：
- 检查 `prompt.md` 文件中的输出格式要求是否正确
- 调整 `temperature` 参数，降低随机性
- 确保模型返回的内容包含正确的 JSON 格式

### 10.4 评分结果不准确

**问题**：AI 评分结果与预期不符

**解决方案**：
- 调整 `prompt.md` 文件中的评分标准和权重
- 尝试使用更强大的模型
- 优化 K 线图的生成质量

## 11. 总结

AI 复评模块是 AgentTrader 项目的重要组成部分，它结合了计算机视觉和自然语言处理技术，能够模拟专业交易员的分析思路，对股票的波段爆发潜力进行评估。该模块具有良好的扩展性，可以支持不同的 LLM 模型和自定义提示词，适合不同的投资策略和市场环境。

通过调整配置参数和提示词模板，可以灵活适应不同的投资风格和市场条件，为投资者提供有价值的股票推荐。