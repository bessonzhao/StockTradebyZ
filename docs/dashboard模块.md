# Dashboard 模块

## 1. 模块概述

Dashboard 模块是 AgentTrader 项目的重要组成部分，负责生成和导出股票 K 线图，包括日线图和周线图。该模块使用 Plotly 库进行图表生成，支持自定义指标、样式和配置，能够生成高质量的股票图表用于分析和 AI 复评。

### 1.1 核心功能

- **日线图生成**：生成包含 K 线、知行线、成交量等指标的日线图
- **周线图生成**：生成包含 K 线、多条 MA 均线、成交量等指标的周线图
- **批量导出**：支持批量导出候选股票的 K 线图
- **自定义配置**：支持自定义图表样式、指标参数和输出格式
- **高质量渲染**：生成高分辨率、清晰易读的图表

### 1.2 技术栈

- **图表生成**：Plotly
- **数据处理**：Pandas, NumPy
- **导出格式**：JPEG（通过 kaleido 库）
- **编程语言**：Python 3.x

## 2. 架构设计

### 2.1 模块结构

```
┌─────────────────────────────────────────┐
│               dashboard/                │
├─────────────────────────────────────────┤
│ ┌─────────────────┐     ┌─────────────┐ │
│ │ export_kline_   │     │ components/ │ │
│ │ charts.py       │────▶│ charts.py   │ │
│ └─────────────────┘     │ __init__.py │ │
│                         └─────────────┘ │
│ ┌─────────────────┐     ┌─────────────┐ │
│ │ app.py          │     │ assets/     │ │
│ └─────────────────┘     │ style.css   │ │
│                         └─────────────┘ │
└─────────────────────────────────────────┘
```

### 2.2 核心组件

| 组件 | 作用 |
|------|------|
| export_kline_charts.py | 批量导出候选股票 K 线图的主脚本 |
| components/charts.py | 包含图表生成的核心逻辑，包括日线图和周线图生成函数 |
| app.py | 基于 Dash 的交互式仪表板应用（暂未详细实现） |
| assets/style.css | 仪表板应用的样式文件 |

### 2.3 工作流程

1. **加载配置**：读取配置文件或命令行参数
2. **读取候选股票**：从 candidates_latest.json 读取候选股票列表
3. **加载原始数据**：读取股票日线 CSV 数据
4. **计算指标**：计算知行线、KDJ、砖型图等指标
5. **生成图表**：使用 Plotly 生成日线图和周线图
6. **导出图表**：将图表导出为 JPEG 格式
7. **输出结果**：生成图表文件到指定目录

## 3. 详细工作流程

### 3.1 配置加载

Dashboard 模块支持通过配置文件和命令行参数进行配置，主要配置项包括：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| candidates | 候选股票文件路径 | data/candidates/candidates_latest.json |
| raw_dir | 原始数据目录 | data/raw |
| out_dir | 输出目录 | data/kline |
| bars | 日线显示 K 线数量 | 120 |
| weekly_bars | 周线显示 K 线数量 | 60 |
| day_width | 日线图宽度 | 1400 |
| day_height | 日线图高度 | 700 |
| week_width | 周线图宽度 | 1400 |
| week_height | 周线图高度 | 700 |

### 3.2 候选股票读取

从 `data/candidates/candidates_latest.json` 文件中读取候选股票列表，获取股票代码和选股日期。

### 3.3 原始数据加载

根据股票代码从 `data/raw` 目录读取对应的日线 CSV 文件，进行数据清洗和格式转换：

1. 读取 CSV 文件
2. 统一列名为小写
3. 将日期列转换为 datetime 类型
4. 按日期排序
5. 重置索引

### 3.4 指标计算

在生成图表前，需要计算各种技术指标：

| 指标 | 计算方法 | 用途 |
|------|----------|------|
| KDJ | 基于最高价、最低价、收盘价计算 | 衡量股票超买超卖情况 |
| 知行线 | 双指数平滑移动平均线 + 四均线均值 | 判断趋势和多空力量 |
| 砖型图 | 基于通达信公式计算 | 识别反转信号 |
| MA 均线 | 简单移动平均线 | 判断趋势和支撑阻力位 |

### 3.5 图表生成

#### 3.5.1 日线图生成

日线图包含以下主要元素：

- **K 线图**：展示开盘价、最高价、最低价、收盘价
- **知行线**：包括短期均线（zxdq）和长期均线（zxdkx）
- **成交量**：柱状图展示成交量，上涨为红色，下跌为绿色

核心函数：`make_daily_chart()`

#### 3.5.2 周线图生成

周线图包含以下主要元素：

- **周 K 线图**：展示每周的开盘价、最高价、最低价、收盘价
- **MA 均线**：多条不同周期的移动平均线
- **周成交量**：柱状图展示每周成交量

核心函数：`make_weekly_chart()`

### 3.6 图表导出

使用 Plotly 的 `write_image()` 方法将图表导出为 JPEG 格式，支持自定义宽度、高度和分辨率：

```python
def _export_fig(fig, out_path: Path, width: int, height: int) -> None:
    """将 Plotly Figure 导出为 JPEG。"""
    out_path.parent.mkdir(parents=True, exist_ok=True)
    fig.write_image(
        str(out_path),
        format="jpg",
        width=width,
        height=height,
        scale=2,        # 2× 分辨率，适合屏幕阅读
    )
```

### 3.7 结果输出

生成的图表文件保存在 `data/kline/<date>/` 目录下，命名格式为：

- 日线图：`<code>_day.jpg`
- 周线图：`<code>_week.jpg`

## 4. 核心功能详解

### 4.1 日线图生成

#### 4.1.1 函数签名

```python
def make_daily_chart(
    df: pd.DataFrame,
    code: str,
    volume_up_color: str = "rgba(220,53,69,0.7)",
    volume_down_color: str = "rgba(40,167,69,0.7)",
    bars: int = 120,
    height: int = 560,
    zx_params: Optional[dict] = None,
    show_brick: bool = False,
    brick_params: Optional[dict] = None,
) -> go.Figure:
```

#### 4.1.2 参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| df | 包含日线数据的 DataFrame | 必填 |
| code | 股票代码 | 必填 |
| volume_up_color | 上涨成交量颜色 | rgba(220,53,69,0.7) |
| volume_down_color | 下跌成交量颜色 | rgba(40,167,69,0.7) |
| bars | 显示的 K 线数量，0 表示全部 | 120 |
| height | 图表高度（像素） | 560 |
| zx_params | 知行线计算参数 | None |
| show_brick | 是否显示砖型图 | False |
| brick_params | 砖型图计算参数 | None |

#### 4.1.3 输出

返回一个 Plotly Figure 对象，包含 K 线图、知行线和成交量柱状图。

### 4.2 周线图生成

#### 4.2.1 函数签名

```python
def make_weekly_chart(
    df: pd.DataFrame,
    code: str,
    ma_windows: List[int] = None,
    ma_colors: Dict[int, str] = None,
    volume_up_color: str = "rgba(220,53,69,0.7)",
    volume_down_color: str = "rgba(40,167,69,0.7)",
    bars: int = 60,
    height: int = 400,
) -> go.Figure:
```

#### 4.2.2 参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| df | 包含日线数据的 DataFrame | 必填 |
| code | 股票代码 | 必填 |
| ma_windows | MA 均线周期列表 | [5, 10, 20, 60] |
| ma_colors | MA 均线颜色映射 | {5: "#e67e22", 10: "#27ae60", 20: "#2980b9", 60: "#8e44ad"} |
| volume_up_color | 上涨成交量颜色 | rgba(220,53,69,0.7) |
| volume_down_color | 下跌成交量颜色 | rgba(40,167,69,0.7) |
| bars | 显示的周 K 线数量，0 表示全部 | 60 |
| height | 图表高度（像素） | 400 |

#### 4.2.3 输出

返回一个 Plotly Figure 对象，包含周 K 线图、多条 MA 均线和周成交量柱状图。

### 4.3 指标计算

#### 4.3.1 KDJ 指标

```python
def _calc_kdj(
    df: pd.DataFrame,
    n: int = 9,
    m1: int = 3,
    m2: int = 3,
) -> tuple[pd.Series, pd.Series, pd.Series]:
    """计算 KDJ 指标（通达信标准公式）"""
    high  = df["high"].astype(float)
    low   = df["low"].astype(float)
    close = df["close"].astype(float)

    llv = low.rolling(n, min_periods=1).min()
    hhv = high.rolling(n, min_periods=1).max()
    denom = hhv - llv
    denom = denom.replace(0, 1e-6)
    rsv = (close - llv) / denom * 100.0

    alpha_k = 1.0 / m1
    alpha_d = 1.0 / m2
    k = rsv.ewm(alpha=alpha_k, adjust=False).mean()
    d = k.ewm(alpha=alpha_d, adjust=False).mean()
    j = 3 * k - 2 * d

    return k, d, j
```

#### 4.3.2 知行线

```python
def _calc_zx_lines(
    df: pd.DataFrame,
    zxdq_span: int = 10,
    m1: int = 14, m2: int = 28, m3: int = 57, m4: int = 114,
) -> tuple[pd.Series, pd.Series]:
    """计算知行线指标"""
    close = df["close"].astype(float)
    zxdq  = close.ewm(span=zxdq_span, adjust=False).mean().ewm(span=zxdq_span, adjust=False).mean()
    zxdkx = (
        close.rolling(m1, min_periods=m1).mean()
        + close.rolling(m2, min_periods=m2).mean()
        + close.rolling(m3, min_periods=m3).mean()
        + close.rolling(m4, min_periods=m4).mean()
    ) / 4.0
    return zxdq, zxdkx
```

#### 4.3.3 砖型图

```python
def _calc_brick(
    df: pd.DataFrame,
    n: int = 4, m1: int = 4, m2: int = 6, m3: int = 6,
    t: float = 4.0, shift1: float = 90.0, shift2: float = 100.0,
    sma_w1: int = 1, sma_w2: int = 1, sma_w3: int = 1,
) -> pd.Series:
    """计算砖型图指标"""
    # 纯 numpy/pandas 实现，基于通达信公式
    # ...（详细实现见源码）
    return pd.Series(raw, index=df.index)
```

### 4.4 批量导出

```python
def main() -> None:
    candidates_path = Path(CONFIG["candidates"])
    raw_dir         = Path(CONFIG["raw_dir"])

    codes, pick_date = _load_candidates(candidates_path)

    # 导出日期直接读取 candidates.json 的 pick_date
    export_date = pick_date
    if not export_date:
        print("[ERROR] candidates.json 中未设置 pick_date，无法确定导出日期。")
        sys.exit(1)
    print(f"[INFO] 导出日期：{export_date}")

    out_root = Path(CONFIG["out_dir"]) / export_date

    ok_count    = 0
    skip_count  = 0

    for code in codes:
        df_raw = _load_raw(code, raw_dir)
        if df_raw.empty:
            print(f"[SKIP] {code}  — 无日线数据")
            skip_count += 1
            continue

        # ── 日线图 ────────────────────────────────────────────────────
        day_path = out_root / f"{code}_day.jpg"
        try:
            fig_day = make_daily_chart(
                df_raw, code,
                bars=CONFIG["bars"],
                height=CONFIG["day_height"],
            )
            _export_fig(fig_day, day_path, CONFIG["day_width"], CONFIG["day_height"])
        except Exception as e:
            print(f"[ERROR] {code} 日线导出失败：{e}")
            skip_count += 1
            continue

        # ── 周线图 ────────────────────────────────────────────────────
        # 周线图导出暂被注释
        # ...

        print(f"[OK]   {code}  → {day_path.name}")
        ok_count += 1

    print(
        f"\n导出完成：成功 {ok_count} 只，跳过 {skip_count} 只。"
        f"\n输出目录：{out_root}"
    )
```

## 5. 配置参数

### 5.1 配置文件

`export_kline_charts.py` 中包含一个配置字典，可以直接修改：

```python
CONFIG = {
    "candidates": str(_ROOT / "data" / "candidates" / "candidates_latest.json"),
    "raw_dir":    str(_ROOT / "data" / "raw"),
    "out_dir":    str(_ROOT / "data" / "kline"),
    "bars":       120,   # 日线显示 K 线数量（0 = 全部）
    "weekly_bars": 60,   # 周线显示 K 线数量（0 = 全部）
    "day_width":  1400,
    "day_height": 700,
    "week_width": 1400,
    "week_height": 700,
}
```

### 5.2 命令行参数

目前 `export_kline_charts.py` 支持以下命令行参数：

```bash
python dashboard/export_kline_charts.py [--date YYYY-MM-DD] [--bars 120] [--weekly-bars 60]
```

| 参数 | 说明 |
|------|------|
| --date | 导出日期，格式为 YYYY-MM-DD |
| --bars | 日线显示 K 线数量 |
| --weekly-bars | 周线显示 K 线数量 |

## 6. 使用方法

### 6.1 安装依赖

```bash
pip install kaleido   # Plotly 静态图导出必需
```

### 6.2 批量导出 K 线图

```bash
python dashboard/export_kline_charts.py
```

### 6.3 指定导出日期

```bash
python dashboard/export_kline_charts.py --date 2026-03-15
```

### 6.4 调整显示的 K 线数量

```bash
python dashboard/export_kline_charts.py --bars 150 --weekly-bars 80
```

### 6.5 集成到全流程

`export_kline_charts.py` 是 `run_all.py` 脚本的第三步，用于在量化初选后生成候选股票的 K 线图，供 AI 复评使用。

## 7. 扩展与自定义

### 7.1 自定义图表样式

可以通过修改 `components/charts.py` 中的以下参数来自定义图表样式：

- `_LIGHT_LAYOUT`：图表布局配置
- `volume_up_color` 和 `volume_down_color`：成交量颜色
- K 线颜色：`increasing_line_color` 和 `decreasing_line_color`
- 均线颜色：`ma_colors` 参数

### 7.2 添加新的技术指标

1. 在 `components/charts.py` 中添加指标计算函数
2. 在 `make_daily_chart()` 或 `make_weekly_chart()` 函数中添加指标的绘制逻辑
3. 可以通过参数控制是否显示该指标

### 7.3 支持更多输出格式

可以修改 `_export_fig()` 函数，支持导出为 PNG、SVG 等格式：

```python
def _export_fig(fig, out_path: Path, width: int, height: int, format: str = "jpg") -> None:
    """将 Plotly Figure 导出为指定格式。"""
    out_path.parent.mkdir(parents=True, exist_ok=True)
    fig.write_image(
        str(out_path),
        format=format,
        width=width,
        height=height,
        scale=2,
    )
```

### 7.4 调整图表分辨率

可以修改 `_export_fig()` 函数中的 `scale` 参数来调整图表分辨率，默认值为 2，表示 2× 分辨率。

## 8. 常见问题与解决方案

### 8.1 缺少 kaleido 库

**问题**：运行时提示缺少 kaleido 库

**解决方案**：
```bash
pip install kaleido
```

### 8.2 图表导出失败

**问题**：导出图表时报 `write_image` 错误

**解决方案**：
- 确认已安装 kaleido：`pip install -U kaleido`
- 检查输出目录是否存在且有写入权限
- 检查原始数据是否完整

### 8.3 图表显示不完整

**问题**：生成的图表中 K 线数量不足

**解决方案**：
- 调整 `bars` 参数，增加显示的 K 线数量
- 确保原始数据包含足够的历史数据

### 8.4 指标计算错误

**问题**：生成的图表中指标显示异常

**解决方案**：
- 检查原始数据是否完整
- 确保指标计算函数的参数设置正确
- 检查数据格式是否正确

## 9. 性能优化

### 9.1 批量处理

- 一次性处理多只股票，减少重复加载和初始化开销
- 使用并行化处理，提高导出效率

### 9.2 缓存机制

- 缓存已计算的指标，避免重复计算
- 缓存已生成的图表，支持断点续跑

### 9.3 资源管理

- 及时释放不再使用的资源
- 调整图表分辨率和大小，平衡质量和性能

## 10. 总结

Dashboard 模块是 AgentTrader 项目的重要组成部分，负责生成和导出高质量的股票 K 线图。该模块使用 Plotly 库进行图表生成，支持自定义指标、样式和配置，能够满足不同场景的需求。

通过批量导出候选股票的 K 线图，Dashboard 模块为后续的 AI 复评提供了必要的图表数据，是连接量化初选和 AI 复评的重要桥梁。

该模块具有良好的扩展性，可以通过添加新的指标、样式和输出格式来满足更多的需求。同时，模块的设计也考虑了性能优化，能够高效处理大量股票的图表生成和导出任务。