# VOO + QQQ 定投策略规则

> 程序实现的唯一规则依据。先改文档，再改代码。

---

## 1. 参数总表

放入配置文件（`config.toml` / `.env`），不要硬编码。

| 键名                   | 当前值 | 说明                           |
| ---------------------- | ------ | ------------------------------ |
| `BASE_WEEKLY`          | 500    | 每周基础定投额，两只各 50%     |
| `BUY_WEEKDAYS`         | [3, 4] | 周三、周四都运行提醒           |
| `MAX_ETF_WEIGHT`       | 0.45   | 单只占账户总值上限             |
| `REBALANCE_TARGET`     | 0.40   | 再平衡后降回的占比             |
| `MAX_WEEKLY_BUY_RATIO` | 0.20   | 单周买入上限（占账户总值）     |
| `HOT_VIX`              | 15     | 过热：VIX 低于此值             |
| `HOT_CNN`              | 75     | 过热：CNN 高于此值（极度贪婪） |

- **账户总值** = VOO 市值 + QQQ 市值 + 现金
- **ATH 回撤** = 前一交易日收盘价 / 历史最高收盘价 − 1

---

## 2. 买入规则

- 周三、周四都运行，用**前一交易日收盘价**计算；休市当天跳过
- 每周只买一次：周三已录入成交，则周四只做检查不再建议买入；周三未成交，周四按最新数据重新计算
- 下单避开开盘后、收盘前各 30 分钟，或挂限价单

**回撤倍数表**（每只按各自回撤计算，左闭右开）

| VOO 回撤    | QQQ 回撤    | 倍数 |
| ----------- | ----------- | ---- |
| 0 ~ -5%     | 0 ~ -7%     | 0.5x |
| -5% ~ -10%  | -7% ~ -15%  | 1.0x |
| -10% ~ -20% | -15% ~ -25% | 1.5x |
| -20% ~ -30% | -25% ~ -35% | 2.0x |
| < -30%      | < -35%      | 3.0x |

```
买入额[etf] = BASE_WEEKLY × 50% × 倍数
```

**保护规则**（依次裁剪）

1. 单周合计 ≤ 账户总值 × 20%，超出等比例缩减
2. 买入后单只占比 ≤ 45%，超出部分不买（两只合计 ≤ 90%，自然留出 ≥ 10% 现金）

---

## 3. 卖出规则

### 3.1 再平衡

单只占比 **≥ 45%** 时卖出，降回 **40%**：

```
卖出额 = 该 ETF 市值 − 账户总值 × 40%
```

### 3.2 过热减仓

三个条件**同时满足**（回撤按每只单独判断）：

1. 距 ATH 回撤在 **-3% 以内**
2. VIX **< 15**
3. CNN 恐惧贪婪 **> 75**

动作：该只**暂停本周买入**，并**提醒卖出持仓 5%**（人工确认）。每只每季度最多 1 次，累计不超过持仓 20%。

> 阈值依据：近几个月 VIX 最低约 13.8，`<15` 可覆盖低波动高位区间；CNN 75 为"极度贪婪"起点。随行情每季度复核一次。

---

## 4. 每周流程

1. 拉数据：VOO/QQQ 收盘价与 ATH、VIX、CNN、账户持仓与现金
2. 过热检查 → 命中则暂停该只 + 提醒减仓 5%
3. 再平衡检查 → ≥ 45% 的卖回 40%
4. 计算买入额 → 套用保护规则
5. 推送 Telegram + 写日志
6. 人工下单后更新持仓记录

有新入金时重算 `BASE_WEEKLY`（默认按 20 周摊）。

---

## 5. 数据来源

| 数据            | 来源                                                     | 备注                                          |
| --------------- | -------------------------------------------------------- | --------------------------------------------- |
| VOO / QQQ / VIX | yfinance（`VOO` / `QQQ` / `^VIX`）                       | 用 `curl_cffi` 模拟浏览器；ATH 取历史最高收盘 |
| CNN 恐惧贪婪    | `production.dataviz.cnn.io/index/fearandgreed/graphdata` | 非官方接口，需 `curl_cffi`；失败视为不过热    |
| 账户持仓与资金  | IBKR Flex Web Service（`ibflex`）                         | Token + Query ID，只读报表；不是盘中实时数据    |

### 5.1 IBKR 账户查询

- 使用 `uv` 管理依赖，凭证通过本地 `.env` 或环境变量提供：`IBKR_FLEX_TOKEN`、`IBKR_FLEX_QUERY_ID`。
- Activity Flex Query 使用 XML 格式，启用 Open Positions、Cash Report、Net Asset Value (NAV) Summary in Base。
- 仅下载并读取报表，将各账户的报表日期、持仓、各币种现金和净资产显示在终端；不下单、修改账户或自动保存账户报表。
- 资金区分各币种现金和基础币种总计，不将不同币种直接相加；没有勾选的报表模块标为缺失，不当作零余额或空仓。
- Flex 报表是指定报表日期的数据快照，不能用来读取盘中实时持仓、金额或挂单状态。
- 资金查询测试脚本逐个显示报表中的账户及最新日期的现金、已结算现金和净资产，输出实际返回的账户数量；可用 `--expect-accounts` 校验数量，数量不符时退出码为 2，提示检查查询的账户选择。它不能发现未包含在 Flex Query 中的其他账户，也不把现金当作实时购买力。

```python
from datetime import date
from curl_cffi import requests as curl_requests
import yfinance as yf

session = curl_requests.Session(impersonate="chrome")

def fetch_history(symbol: str):
    df = yf.Ticker(symbol, session=session).history(period="max", interval="1d")
    df = df.dropna(subset=["Close"])
    return df[df.index.date < date.today()]

def get_cnn_fear_greed() -> float | None:
    try:
        r = session.get("https://production.dataviz.cnn.io/index/fearandgreed/graphdata", timeout=10)
        return r.json()["fear_and_greed"]["score"]
    except Exception:
        return None
```

---

## 6. 实现约定

- Python，`uv` 管理依赖
- 只计算和提醒，**不自动下单**
- 持久化：持仓、现金、交易记录、过热减仓历史、ATH 缓存

---

## 7. 风险提示

- 50/50 配置科技股实际占比 > 40%，波动大（QQQ 2000~2002 最大回撤 -83%，2022 年 -35%）
- 慢熊会长期加码，单周 20% 上限防止现金过快耗尽
