# RSI 计算参考

## RSI(14) 计算步骤

### 公式

```python
def calc_rsi(klines, period=14):
    closes = [float(k['close']) for k in klines]
    if len(closes) < period + 5:
        return None
    deltas = [closes[i] - closes[i-1] for i in range(1, len(closes))]
    gains = [d if d > 0 else 0 for d in deltas]
    losses = [-d if d < 0 else 0 for d in deltas]

    avg_gain = sum(gains[-period:]) / period
    avg_loss = sum(losses[-period:]) / period

    # Wilder 平滑（从 period-1 位置向前迭代）
    for i in range(len(gains) - period - 1, -1, -1):
        avg_gain = (avg_gain * (period - 1) + gains[i]) / period
        avg_loss = (avg_loss * (period - 1) + losses[i]) / period

    if avg_loss == 0:
        return 100.0
    rs = avg_gain / avg_loss
    return round(100 - 100 / (1 + rs), 2)
```

### MA 计算

```python
def calc_ma(klines, period=20):
    closes = [float(k['close']) for k in klines]
    if len(closes) < period:
        return None
    return round(sum(closes[-period:]) / period, 2)
```

### RSI 区间参考

| RSI 区间 | 信号 | 操作建议 |
|---------|------|---------|
| > 70 | 极度超买 | ⚠️ 耐心等待，不追高 |
| 55–70 | 偏暖 | 中性，不急于补仓 |
| 45–55 | 中性 | 正常持有 |
| 30–45 | 偏冷 | 关注建仓机会 |
| < 30 | 超卖 | 建仓区间（需结合回撤条件） |

---

## 数据获取

### 东方财富 K 线接口（推荐）

```
URL: https://push2his.eastmoney.com/api/qt/stock/kline/get
  ?secid={secid}
  &fields1=f1,f2,f3,f4,f5,f6
  &fields2=f51,f52,f53,f54,f55,f56,f57,f58
  &klt=101   # 日K线
  &fqt=1     # 前复权
  &end=20500101
  &lmt=90    # 取近90日
```

**返回格式**：JSON `{"data":{"klines":["2026-05-06,1.234,1.250,1.220,1.240,123456",...]}}`
字段顺序：日期,开盘,最高,最低,收盘,成交量

### 新浪财经 ETF K 线接口（兜底）

```
URL: https://money.finance.sina.com.cn/quotes_service/api/json_v2.php/CN_MarketData.getKLineData
  ?symbol={sz/sh + ETF代码}
  &scale=240
  &ma=5
  &datalen=90
```

返回：`[{"day":"2026-01-02","open":"1.234","high":"1.250","low":"1.220","close":"1.240","volume":"123456"}]`

### 天天基金历史净值（计算 RSI 的备选兜底）

当东方财富 K 线接口不稳定时，使用：
```
URL: https://api.fund.eastmoney.com/f10/lsjz?callback=jQuery&fundCode={code}&pageIndex=1&pageSize=90
```
返回：jQuery(JSON) — 需正则提取内层 JSON，数据倒序（最新日期在前）。

---

## 关键约束

- **Wilder 平滑需要至少 period+5=19 条 K 线数据**才能得到有效 RSI 值
- 025209（半导体）成立时间短（约1个月历史净值），用基金净值本身计算 RSI 样本量不足，统一用 159813 的 RSI 作为建仓信号依据
- K 线接口不稳定时（返回空数据 / `exit:52`）→ 立即切换新浪财经 ETF K 线接口

---

## 各 ETF RSI 速查（2026-05-13 快照）

| ETF | 代码 | secid | RSI(14) | 状态 |
|-----|------|-------|---------|------|
| 半导体ETF | 159813 | 0.159813 | 81.9 | 🔴 极度超买 |
| 电网设备ETF | 560660 | 1.560660 | 67.9 | 🟡 偏高 |
| 有色ETF | 560860 | 0.560860 | 59.4 | 🟡 偏高 |
| 有色金属ETF | 512400 | 0.512400 | 55.0 | 🟢 正常 |
| 卫星ETF | 159206 | 0.159206 | 64.5 | 🟡 偏高 |
| 新能源ETF | 516160 | 1.516160 | 57.3 | 🟢 正常 |