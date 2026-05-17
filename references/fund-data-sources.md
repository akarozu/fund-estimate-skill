# 基金数据源参考

## 数据源稳定性排序

| 优先级 | 数据源 | 适用场景 | 稳定性 |
|--------|--------|---------|--------|
| 1 | 天天基金 JSONP | 场外基金估算净值（gsz/dwjz） | ✅ 高 |
| 2 | 东方财富历史净值 | RSI 技术指标计算 | ✅ 高 |
| 3 | 东方财富实时行情 | 场内 ETF 实时价格 | ⚠️ 间歇性限速 |
| 4 | 新浪财经 ETF K 线 | RSI 备选兜底 | ✅ 稳定 |
| 5 | 同花顺 HQ 页面 | 场内 ETF 联接基金 | ⚠️ 不稳定，失效即降级 |

---

## ✅ 可用接口详情

### 天天基金估算净值（最可靠）

```
URL: https://fundgz.1234567.com.cn/js/{code}.js?rt={unix_ts}
返回: jsonpgz({...})
解析: re.search(r'\((\{[^}]+\})\)', text) → json.loads(m.group(1))
字段: gsz=估算净值, gszzl=估算涨跌幅(str), dwjz=昨日净值, gztime=估算时间
```

> ⚠️ 必须带 `Referer: https://fund.eastmoney.com/` header，否则返回空。

### 东方财富历史净值（RSI 计算）

```
URL: https://api.fund.eastmoney.com/f10/lsjz?callback=jQuery&fundCode={code}&pageIndex=1&pageSize=90
返回: jQuery({...}) — JSONP，需正则提取内层 JSON
解析: re.search(r'jQuery\((.*)\)', text)[1] → json.loads
字段: FSRQ=日期, DWJZ=单位净值, JZZZL=日涨跌幅(str可转float)
⚠️ 数据倒序（最新日期在前），使用前需 reversed()
```

### 东方财富实时行情（场内 ETF）

```
URL: https://push2.eastmoney.com/api/qt/stock/get?secid={secid}&fields=f43,f169,f170
字段: f43=最新价(分), f169=涨跌额(分), f170=涨跌幅(基点×100=%)
⚠️ 间歇性返回 exit:52 或 "Remote end closed"，超时后自动切换天天基金
```

### 东方财富 K 线接口（RSI 首选）

```
URL: https://push2his.eastmoney.com/api/qt/stock/kline/get
  ?secid={secid}
  &fields1=f1,f2,f3,f4,f5,f6
  &fields2=f51,f52,f53,f54,f55,f56,f57,f58
  &klt=101   # 日K线
  &fqt=1     # 前复权
  &end=20500101
  &lmt=90    # 取近90日
返回: {"data":{"klines":["2026-05-06,1.234,1.250,1.220,1.240,123456",...]}}
字段顺序: 日期,开盘,最高,最低,收盘,成交量
```

### 新浪财经 ETF K 线（兜底）

```
URL: https://money.finance.sina.com.cn/quotes_service/api/json_v2.php/CN_MarketData.getKLineData
  ?symbol={sz/sh + ETF代码}&scale=240&ma=5&datalen=90
返回: [{"day":"2026-01-02","open":"1.234","close":"1.240","high":"1.250","low":"1.220","volume":"123456"}]
```

---

## ❌ 失效接口清单

| 接口 | 问题 | 替代方案 |
|------|------|---------|
| `push2his.eastmoney.com`（K线） | `'NoneType' object is not subscriptable` / 空响应 | 改用天天基金历史净值 |
| `push2.eastmoney.com`（实时） | "Remote end closed connection without response" | 降级至天天基金估算 |
| `hq.sinajs.cn`（腾讯/新浪ETF） | `v_pv_none_match="1"` 代码未匹配 | 天天基金 |
| `money.finance.sina.com.cn`（新浪K线） | `None` 返回 | 东方财富历史净值 |
| `api.gold-api.com/price/XAU` | JSON date序列化错误 | 降级至518880代理 |
| `COMEX期货`（Yahoo Finance） | HTTP 404 | 不依赖 |

---

## 黄金 ETF 代理

国内金价实时代理：`518880 国泰黄金ETF`

- 新浪前缀：`sh518880`
- 东方财富 secid：`1.518880`
- 当外盘黄金接口失效时，使用 518880 盘中价格作为金价代理

---

## 今日行情快照（2026-05-15 10:12）

- **518880 国泰黄金ETF**: 96.42元, **-1.56%**
- **512400 南方有色ETF**: 20.65元, **-3.05%**
- **560860 有色金属ETF**: 17.25元, **-3.20%**

### 持仓基金两日连续大跌（5/13→5/14→5/15估算）

| 基金 | 5/14净值涨跌 | 5/15估算涨跌 | 两日累计 |
|------|------------|------------|--------|
| 万家有色C | -3.38% | -3.10% | ≈-6.5% |
| 南方有色E/A | -3.40% | -3.11% | ≈-6.5% |
| 中信保诚有色800C | -3.37% | -3.23% | ≈-6.6% |
| 南方新能源C | -2.92% | -2.10% | ≈-5.0% |
| 景顺红利低波C | -0.51% | -0.22% | 相对抗跌 |
| 沪深300C | -1.53% | -0.81% | -2.3% |

**背景**: 受特朗普关税政策反复影响，市场避险情绪升温，黄金短期承压，有色金属同步大跌。

---

## 补仓决策参考

| 基金 | RSI(14日) | 建议 |
|------|----------|------|
| 万家有色C | 56.4 | 中性，今日不追 |
| 南方有色E/A | ~53 | 中性，等待 RSI<40 |
| 中信保诚有色800C | 52.7 | 中性 |
| 沪深300C | **72.3** | ⚠️ 超买，暂停追加 |
| 景顺红利低波C | 53.7 | 中性，相对安全 |

**核心原则**: RSI<30 超卖区补仓胜率最高；中性区域（50~60）不急于补。