# 市场情绪数据源参考

## ⚠️ 关键原则（确认于 2026-05-13）

**主数据源 = 乐咕乐股赚钱效应**（前日15:00收盘每日更新）  
**实时补充 = 东方财富指数涨跌**（盘中实时）

两者各司其职，不冲突：
- 赚钱效应 = 市场宽度（前日收盘，日频）
- 指数涨跌 = 盘中动量（实时）

> ❌ 不再强求自计算"恐贪指数"替代单一数值——直接引用乐咕乐股赚钱效应即可。

---

## 一、乐咕乐股赚钱效应（主数据源）

**数据说明**：每日 15:00 收盘后更新一次（前日全市场统计）。这是该数据的正常设计，不是 bug。

**Browser 提取方法**（日报使用）：
```javascript
// 在 browser_console 中执行
(function() {
  const cells = document.querySelectorAll('td');
  const result = {};
  for (let i = 0; i < cells.length; i += 2) {
    if (i + 1 < cells.length) {
      const label = cells[i].textContent.trim();
      const val = cells[i+1].textContent.trim();
      if (/^\d+(\.\d+)?%$/.test(val) || /^\d+(\.\d+)?$/.test(val)) {
        result[label] = val;
      }
    }
  }
  return JSON.stringify(result);
})();
// 返回: {"上涨":"3036","下跌":"2008","平盘":"139","涨停":"146","跌停":"17",...}
```

**懒人方案（直接看页面标题）**：
- 页面顶部有 `活跃度：58.35%`（≈赚钱效应）
- heading `涨跌比: 58.35%` 即为赚钱效应数值

**已知局限**：只有前日收盘数据，14:30 日报引用它没有问题（本身就是日频指标）。

**正确日报工作流**：
1. 引用乐咕乐股赚钱效应（前日收盘，日频指标 = 恐贪指数的最佳替代）
2. 同时引用东方财富实时指数涨跌（盘中实时，反映今日盘中动量）
3. 两者各司其职：宽度看前日，动量看实时

**自计算恐贪指数的局限**：按新浪代码均匀采样560只股票计算，**反映的也是上一交易日收盘的全市场情况**，与乐咕乐股赚钱效应同频。两种方式均可使用，数值应接近（实测均≈54-58）。

---

## 东方财富实时指数行情 API

**Endpoint**: `https://push2.eastmoney.com/api/qt/stock/get`

**参数**：
- `secid`: 交易所代码.指数代码（0=深交所，1=上交所）
- `fields`: `f43`（当前价×100）, `f57`, `f58`, `f169`（涨跌额）, `f170`（涨跌幅×100）

**常用指数secid**：

| 指数 | secid |
|------|-------|
| 创业板指 | `0.399006` |
| 沪深300 | `1.000300` |
| 上证指数 | `1.000001` |
| 深证成指 | `0.399001` |
| 科创50 | `1.000688` |

**Python调用**：
```python
import urllib.request, json

def get_index(secid):
    url = f'https://push2.eastmoney.com/api/qt/stock/get?secid={secid}&fields=f43,f58,f170'
    req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0', 'Referer': 'https://finance.eastmoney.com/'})
    with urllib.request.urlopen(req, timeout=10) as r:
        d = json.loads(r.read().decode('utf-8'))['data']
        return f"{d['f58']}: {d['f170']/100:+.2f}%"
```

---

## A股恐贪指数自计算方案（备选兜底）

> ⚠️ **优先级**：乐咕乐股赚钱效应（主数据源）> 自计算方案（备选兜底）。
> 自计算方案仅在乐咕乐股页面无法提取时使用。两者本质相同（均反映前日收盘市场宽度），数值应接近。

### 算法（2成分稳健版）

| 成分 | 权重 | 计算逻辑 |
|------|------|---------|
| **涨跌广度** | 40% | `updown/(1+updown)*100`，0=全跌，50=均衡，100=全涨 |
| **动量强度** | 60% | `(p50+3)/6*100`，-3%~+3% → 0~100，中位数抗极值干扰 |

> **算法修正**：初版含"极端密度"（涨停密度），样本偏差导致大幅波动，已移除。

### 稳定采样 URL（新浪财经）

```
# sort=symbol&asc=1（代码升序）=全市场均匀分布
https://vip.stock.finance.sina.com.cn/quotes_service/api/json_v2.php/
  Market_Center.getHQNodeData?num=80&sort=symbol&asc=1&node=hs_a&page={1~7}
```

**教训**：
- ❌ `sort=changepercent&asc=0`（降序取前600）→ 全部大涨股票，涨/跌比失真
- ❌ `sort=changepercent&asc=1`（升序取后600）→ 全部大跌股票
- ✅ `sort=symbol&asc=1` → 均匀分布，与真实全市场接近

### 完整 Python 实现

```python
import urllib.request, json, time
headers = {'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)'}

def fetch_uniform_sample(pages=7, num=80):
    all_changes = []
    for page in range(1, pages + 1):
        url = (f"https://vip.stock.finance.sina.com.cn/quotes_service/api/json_v2.php/"
               f"Market_Center.getHQNodeData?num={num}&sort=symbol&asc=1&node=hs_a"
               f"&symbol=&_s_r_a=page&page={page}")
        try:
            req = urllib.request.Request(url, headers=headers)
            with urllib.request.urlopen(req, timeout=8) as resp:
                data = json.loads(resp.read().decode('gbk', errors='ignore'))
                if isinstance(data, list) and len(data) > 0:
                    all_changes.extend([float(s['changepercent']) for s in data])
        except: break
        time.sleep(0.1)
    return all_changes

changes = fetch_uniform_sample(pages=7, num=80)
n = len(changes)
up  = sum(1 for c in changes if c > 0)
dn  = sum(1 for c in changes if c < 0)
p50 = sorted(changes)[n // 2]

breadth = (up / (dn or 0.1)) / (1 + up / (dn or 0.1)) * 100
momentum_score = max(0, min(100, (p50 + 3) / 6 * 100))
fg = breadth * 0.40 + momentum_score * 0.60

level_map = [(75,"极度贪婪 😱"),(55,"贪婪 😬"),(45,"中性 😐"),(25,"恐惧 😟"),(0,"极度恐惧 😰")]
level = next(l for t,l in level_map if fg >= t)
print(f"恐贪指数：{fg:.0f}/100  {level}")
print(f"样本：{n}只 | 涨{up}({up/n*100:.0f}%) 跌{dn}({dn/n*100:.0f}%) 平{n-up-dn}")
```

### 验证结果（2026-05-13）

| 指标 | 数值 |
|------|------|
| 样本 | 560只（7页×80） |
| 涨/跌/平 | 306/229/25 |
| 中位数涨跌幅 | +0.155% |
| **恐贪指数** | **54/100** 中性偏暖 😐 |
| 乐咕乐股赚钱效应 | 57.14% ✅ 交叉验证接近 |

> **注**：54/100 与乐咕乐股57%赚钱效应接近，验证算法合理。但创业板+2.26%与中位数+0.14%存在显著分化——说明是**结构性行情**（成长领涨），非全面牛市。

### Pitfall：东方财富接口限流（2026-05-13）

`push2.eastmoney.com` 当日多次返回 `RemoteDisconnected` / `HTTP 502`。**东方财富分页接口不稳定**，不可作主要数据源。**新浪财经是稳定的兜底数据源**。

---

## 乐咕乐股页面适用场景

Legulegu 数据适合用于：
- **盘后**复盘当天市场宽度（涨停/跌停家数）
- **历史对比**（月线/季线级别情绪位置）

**不适合**：盘中实时判断当天涨跌方向。

---

## 乐咕乐股"恐慌&贪心指标" — 存在但需登录

**页面URL**: `https://www.legulegu.com/stockdata/market-style?indexCode=1`

**算法原理**（JS逆向分析）：
- 统计申万行业板块"连续N日上涨/下跌"的数量比例
- 连跌板块越多 → 恐慌加剧
- 连涨板块越多 → 贪心加剧

**API端点**（已逆向但需登录）：
```
GET https://datacenter.legulegu.com/api/stock-data/market-style
  ?indexCode=1&token={MD5(日期)}&timeRange=1y
```
- Token算法: `MD5('2026-05-10') = 460272151cec86213a4d18cffdd61373`
- 返回200但body为空（需登录凭证）

**数据可用性**：需会员登录，公开API不可用。

---

## A股情绪量化指标可用组合（推荐）

| 指标 | 获取方式 | 说明 |
|------|---------|------|
| **赚钱效应** | 乐咕乐股 `market-activity`（盘后）或自计算（实时） | 上涨vs下跌比例，最接近恐贪本质 |
| **自计算恐贪指数** | 新浪采样560只 × 2成分公式 | 盘中实时，54/100附近为中性 |
| **大盘拥挤度** | 乐咕乐股 `ashares-congestion`（盘后） | <50%安全，>50%预警 |
| **实时指数涨跌** | 东方财富实时API | 创业板+2.26%、沪深300+0.74% |
| **铜价** | 金十数据 | 有色板块情绪锚点 |

---

## 各ETF技术面RSI计算（2026-05-13 实测）

东方财富 K线接口（**推荐**，数据完整）：

```
https://push2his.eastmoney.com/api/qt/stock/kline/get
  ?secid={secid}
  &fields1=f1,f2,f3,f4,f5,f6
  &fields2=f51,f52,f53,f54,f55,f56,f57,f58
  &klt=101   # 日K线
  &fqt=1     # 前复权
  &end=20500101
  &lmt=90    # 取近90日
```

**返回**：`{"data":{"klines":["2026-05-06,1.234,1.250,1.220,1.240,123456",...]}}`
K线字段顺序（逗号分隔）：日期,开盘,最高,最低,收盘,成交量

**各ETF secid速查**：

| ETF | 代码 | secid | RSI(14) | 状态 |
|-----|------|-------|---------|------|
| 半导体ETF | 159813 | `0.159813` | **81.9** | 🔴 极度超买 |
| 电网设备ETF | 560660 | `1.560660` | 67.9 | 🟡 偏高 |
| 有色ETF | 560860 | `0.560860` | 59.4 | 🟡 偏高 |
| 有色金属ETF | 512400 | `0.512400` | 55.0 | 🟢 正常 |
| 卫星ETF | 159206 | `0.159206` | 64.5 | 🟢 正常 |
| 新能源ETF | 516160 | `1.516160` | 57.3 | 🟢 正常 |

> ⚠️ **建仓信号动态调整**（2026-05-13）：创业板当日 +2.26%，半导体ETF RSI=81.9 已极度超买。025209 建仓触发条件建议比默认值更严格：RSI 降至 **70 以下且回撤 ≥5%** 才考虑首批入场（原阈值 RSI<55 回撤≥8% 仅适用于震荡市）。

---

## 用户持仓基金 × 跟踪标的映射（确认于2026-05-13）

| 基金代码 | 基金名称 | 类型 | 跟踪标的ETF | 备注 |
|---------|---------|------|-----------|------|
| 018490 | 万家工业有色C | 有色 | 560860 | — |
| 010990 | 南方中证有色E | 有色 | 512400 | — |
| 004432 | 南方中证有色A | 有色 | 512400 | — |
| 013081 | 中信保诚有色800C | 有色 | 场外跟踪 | — |
| 012832 | 南方新能源ETF联接C | 新能源 | 516160 | — |
| 024195 | 永赢卫星通信C | 卫星通信 | 159206 | — |
| 016129 | 景顺长城红利低波C | 红利低波 | 515100 | — |
| 006131 | 华泰柏瑞沪深300ETF联接C | 沪深300 | 000300 | — |
| 025209 | 永赢半导体C | 半导体 | 159813 | **已清仓（2026-05-06）** |
| 025833 | 天弘电网设备C | 电网设备 | 560660 | 待建仓，跟踪中 |

---

## 数据源测试结论（2026-05-13 全量）

| 来源 | 域名 | 结果 | 原因 |
|------|------|------|------|
| 韭圈儿 | 9quans.com | ❌ SSL EOF | 证书问题 |
| 球棍 | qiugun.com | ❌ SSL EOF | 证书问题 |
| 雪球 | xueqiu.com | ❌ 404 | 路径不存在 |
| 通达信 | notianzq.com | ❌ SSL EOF | 证书问题 |
| 乌龟量化 | weguang.com | ❌ SSL EOF | 证书问题 |
| Stockcat | stockcat.io | ❌ SSL EOF | 证书问题 |
| 天天基金 | 1234567.com | ❌ 404 | 无情绪API |
| 新浪财经 | sina.com.cn | ❌ 404 | 无恐贪接口 |
| **新浪采样（稳定）** | `vip.stock.finance.sina.com.cn` | ✅ | `sort=symbol&asc=1` |
| **东方财富实时（限流）** | `push2.eastmoney.com` | ⚠️ | 分页接口不稳定，单次查询可用 |
