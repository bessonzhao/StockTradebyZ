# AgentTrader 项目文档

## 1. 项目概述

AgentTrader 是一个面向 A 股的半自动选股项目，结合了量化选股和 AI 复评功能，旨在帮助用户快速筛选出具有投资潜力的股票。

### 1.1 核心功能

- **数据采集**：使用 Tushare 拉取 A 股股票日线数据
- **量化初选**：基于预设规则对股票进行初步筛选
- **图表导出**：自动生成候选股票的 K 线图
- **AI 复评**：调用 Gemini 对图表进行 AI 分析和打分
- **结果输出**：生成最终的股票推荐列表

### 1.2 技术栈

- **编程语言**：Python 3.x
- **数据采集**：Tushare API
- **量化分析**：Pandas, NumPy
- **图表生成**：Plotly
- **AI 复评**：Google Gemini API
- **配置管理**：YAML
- **命令行工具**：Click

## 2. 目录结构

```
├── agent/              # LLM 评审逻辑（Gemini）
├── config/             # 配置文件目录
├── dashboard/          # 看盘界面与图表导出
├── docs/               # 项目文档
├── pipeline/           # 数据抓取与量化初选
├── data/               # 运行数据与结果（自动生成）
├── README.md           # 项目主文档
├── requirements.txt    # 依赖列表
├── run_all.py          # 全流程一键入口
```

## 3. 安装与配置

### 3.1 安装依赖

```bash
pip install -r requirements.txt
```

### 3.2 设置环境变量

#### Windows PowerShell（永久写入）

```powershell
[Environment]::SetEnvironmentVariable("TUSHARE_TOKEN", "你的TushareToken", "User")
[Environment]::SetEnvironmentVariable("GEMINI_API_KEY", "你的GeminiApiKey", "User")
```

#### Linux/Mac（临时设置）

```bash
export TUSHARE_TOKEN=你的TushareToken
export GEMINI_API_KEY=你的GeminiApiKey
```

#### Linux/Mac（永久写入，添加到 ~/.bashrc 或 ~/.zshrc）

```bash
echo 'export TUSHARE_TOKEN=你的TushareToken' >> ~/.bashrc
echo 'export GEMINI_API_KEY=你的GeminiApiKey' >> ~/.bashrc
source ~/.bashrc
```

## 4. 快速开始

### 4.1 一键运行全流程

```bash
python run_all.py
```

### 4.2 分步运行

1. **拉取 K 线数据**
   ```bash
   python -m pipeline.fetch_kline
   ```

2. **量化初选**
   ```bash
   python -m pipeline.cli preselect
   ```

3. **导出候选图表**
   ```bash
   python dashboard/export_kline_charts.py
   ```

4. **Gemini 复评**
   ```bash
   python agent/gemini_review.py
   ```

5. **查看推荐结果**
   ```bash
   cat data/review/$(date +%Y-%m-%d)/suggestion.json
   ```

## 5. 配置文件说明

### 5.1 fetch_kline.yaml

```yaml
# 数据抓取配置
start: "20230101"          # 起始日期
end: "20260315"            # 结束日期
stocklist: "pipeline/stocklist.csv"  # 股票池文件
exclude_boards: ["gem", "star", "bj"]  # 排除板块
out: "data/raw"            # 输出目录
workers: 8                 # 并发线程数
```

### 5.2 rules_preselect.yaml

```yaml
# 量化初选规则配置
pick_date: ""               # 选股日期，留空则使用最近交易日
data_dir: "data/raw"        # 数据目录
top_m: 500                  # 流动性股票池大小

# B1 选股规则
b1:
  enabled: true             # 是否启用
  params:
    m: 20                   # 参数 m
    n: 5                    # 参数 n
    slope_limit: 1.0         # 斜率限制
    top_k: 20               # 选取前 k 只股票

# 砖型图选股规则（暂未实现）
brick:
  enabled: false            # 是否启用
  params:
    n: 3                    # 参数 n
    top_k: 10               # 选取前 k 只股票
```

### 5.3 gemini_review.yaml

```yaml
# Gemini 复评配置
model: "models/gemini-1.5-flash"  # 模型名称
request_delay: 2               # 调用间隔（秒）
skip_existing: true            # 是否跳过已存在的结果
suggest_min_score: 8.0         # 推荐分数门槛
kline_dir: "data/kline"        # K 线图目录
candidates_path: "data/candidates/candidates_latest.json"  # 候选股票路径
out_dir: "data/review"         # 输出目录
```

### 5.4 dashboard.yaml

```yaml
# 仪表板配置
port: 8050                    # 服务端口
debug: false                  # 是否开启调试模式
```

## 6. 运行流程详解

### 6.1 数据采集阶段

1. 从 `pipeline/stocklist.csv` 读取股票列表
2. 排除指定板块（如创业板、科创板、北交所）
3. 使用 Tushare API 并发拉取股票日线数据
4. 将数据保存到 `data/raw` 目录，格式为 CSV

### 6.2 量化初选阶段

1. 读取 `data/raw` 目录下的所有股票数据
2. 计算股票的流动性指标，选取前 `top_m` 只股票
3. 应用 B1 选股规则，筛选出符合条件的股票
4. 生成候选股票列表，保存到 `data/candidates/candidates_latest.json`

### 6.3 图表导出阶段

1. 读取候选股票列表
2. 为每只候选股票生成 K 线图
3. 将图表保存到 `data/kline/日期` 目录，格式为 JPG

### 6.4 AI 复评阶段

1. 读取候选股票列表和对应的 K 线图
2. 调用 Gemini API 对每只股票的 K 线图进行分析和打分
3. 生成单股评分文件，保存到 `data/review/日期/代码.json`
4. 汇总所有评分，生成推荐列表，保存到 `data/review/日期/suggestion.json`

## 7. 结果文件说明

### 7.1 候选股票文件

`data/candidates/candidates_latest.json`

```json
{
  "pick_date": "2026-03-15",
  "candidates": [
    {
      "code": "000001.SZ",
      "name": "平安银行",
      "close": 12.34,
      "strategy": "b1",
      "score": 0.95
    },
    // 更多候选股票...
  ]
}
```

### 7.2 单股评分文件

`data/review/2026-03-15/000001.SZ.json`

```json
{
  "code": "000001.SZ",
  "name": "平安银行",
  "score": 8.5,
  "analysis": "这只股票的 K 线图显示出良好的上升趋势...",
  "suggestion": "建议关注"
}
```

### 7.3 推荐结果文件

`data/review/2026-03-15/suggestion.json`

```json
{
  "date": "2026-03-15",
  "min_score_threshold": 8.0,
  "recommendations": [
    {
      "code": "000001.SZ",
      "name": "平安银行",
      "score": 8.5,
      "analysis": "这只股票的 K 线图显示出良好的上升趋势..."
    },
    // 更多推荐股票...
  ],
  "excluded": [
    {
      "code": "000002.SZ",
      "name": "万科A",
      "score": 7.5
    }
    // 更多未达门槛股票...
  ]
}
```

## 8. 常见问题与解决方案

### 8.1 Tushare 数据抓取失败

**问题**：`fetch_kline` 报 token 错误或数据抓取失败

**解决方案**：
- 检查 `TUSHARE_TOKEN` 环境变量是否正确设置
- 确认 token 有效且账号权限正常
- 降低并发线程数（修改 `fetch_kline.yaml` 中的 `workers` 参数）
- 检查网络连接是否正常

### 8.2 图表导出失败

**问题**：导出图表时报 `write_image` 错误

**解决方案**：
- 确认已安装 kaleido：`pip install -U kaleido`
- 检查 `data/kline` 目录是否存在且有写入权限

### 8.3 Gemini API 调用失败

**问题**：`gemini_review` 运行失败

**解决方案**：
- 检查 `GEMINI_API_KEY` 环境变量是否正确设置
- 提高 `request_delay` 参数（修改 `gemini_review.yaml`）
- 检查网络连接是否正常
- 确认 API 密钥是否有足够的调用额度

### 8.4 没有候选股票

**问题**：量化初选后没有生成候选股票

**解决方案**：
- 检查 `data/raw` 目录是否有最新数据
- 放宽初选阈值（如调整 `rules_preselect.yaml` 中的参数）
- 检查 `pick_date` 是否在有效交易日

## 9. 开发与扩展

### 9.1 添加新的选股策略

1. 在 `pipeline/Selector.py` 中继承 `BaseSelector` 类
2. 实现 `select` 方法，定义选股逻辑
3. 在 `pipeline/select_stock.py` 中注册新策略
4. 在 `config/rules_preselect.yaml` 中配置新策略的参数

### 9.2 自定义 AI 复评提示

修改 `agent/prompt.md` 文件，自定义 Gemini 的提示词模板。

### 9.3 扩展图表类型

在 `dashboard/components/charts.py` 中添加新的图表生成函数，支持更多类型的技术分析图表。

## 10. 许可证

本项目采用 [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) 协议发布。

- 允许：学习、研究、非商业用途的使用与分发
- 禁止：任何形式的商业使用、出售或以盈利为目的的部署
- 要求：转载或引用须注明原作者与来源

## 11. 联系方式

如有问题或建议，欢迎通过 GitHub Issues 反馈：

[https://github.com/SebastienZh/StockTradebyZ/issues](https://github.com/SebastienZh/StockTradebyZ/issues)
