# fund-estimate cron bug 诊断报告（2026-05-14）

## 事件

cron日报（ae7ea84f01a9，2026-05-14 14:38）交付后，用户发现总涨跌数据严重失真。

## cron 报告 vs 真实值（2026-05-14 14:30 快照）

| 基金 | cron日涨跌 | 真实日涨跌 | cron总涨跌 | 真实总涨跌 |
|------|---------|---------|---------|---------|
| 万家有色C | −2.42% | **−3.59%** | −1.80% | −0.83% |
| 南方有色E | −2.42% | −3.56% | **+2.84%** | −1.02% |
| 南方有色A | −2.44% | −3.56% | +0.45% | −3.00% |
| 中信保诚有色800C | −2.43% | −3.55% | −0.13% | −3.55% |
| 南方新能源C | −1.79% | −3.08% | −2.52% | **+6.60%** |
| 永赢卫星通信C | −3.43% | −5.01% | −2.50% | **+12.26%** |
| 景顺红利低波C | −0.30% | −0.54% | +0.74% | −0.93% |
| 华泰柏瑞沪深300C | −0.88% | −1.68% | −2.52% | **+5.18%** |

| 汇总指标 | cron报告 | 真实值 |
|---------|---------|--------|
| 总资产 | 256,401.54 元 | **92,257.59 元** |
| 今日盈亏 | −5,610.34 元（−2.14%） | **−2,914.14 元（−3.17%）** |
| 总盈亏 | −1,123.53 元（−0.44%） | **+376.80 元（+0.41%）** |

## 根本原因

cron job 混淆了两套数据体系：

1. **天天基金 JSONP**：`dwjz`（昨日净值）+ `gsz`（估算净值），场外NAV
2. **场内 ETF 实时价格**：同花顺HQ页面的买卖报价，与 dwjz 完全无关

**错误模式 A**（混入场内价格）：
```
gsz(场外) / 场内ETF价格(不等同于dwjz) → 错误的日涨跌%
```
**错误模式 B**（旧NAV硬编码）：
```
dwjz 使用了上周/历史净值而非当日最新值 → 总资产大幅失真
```

## 诊断 Python（可重跑验证）

```python
import json, urllib.request, re
from datetime import datetime

positions = [
    {"code": "018490", "name": "万家有色C", "shares": 10994, "cost": 1.8191},
    {"code": "010990", "name": "南方有色E", "shares": 9620, "cost": 2.0788},
    {"code": "004432", "name": "南方有色A", "shares": 4634, "cost": 2.1551},
    {"code": "013081", "name": "中信保诚有色800C", "shares": 3111, "cost": 3.2136},
    {"code": "012832", "name": "南方新能源C", "shares": 14168, "cost": 0.7058},
    {"code": "024195", "name": "永赢卫星通信C", "shares": 1922, "cost": 1.5607},
    {"code": "016129", "name": "景顺红利低波C", "shares": 7613, "cost": 1.3005},
    {"code": "006131", "name": "华泰柏瑞沪深300C", "shares": 7894, "cost": 1.1400},
]

def get_tianfund_gz(code):
    url = f"https://fundgz.1234567.com.cn/js/{code}.js?rt={int(datetime.now().timestamp())}000"
    req = urllib.request.Request(url, headers={
        'User-Agent': 'Mozilla/5.0',
        'Referer': 'https://fund.eastmoney.com/'
    })
    with urllib.request.urlopen(req, timeout=10) as r:
        text = r.read().decode('utf-8')
    m = re.search(r'jsonpgz\((.+)\)', text)
    data = json.loads(m.group(1))
    return {'gsz': float(data['gsz']), 'dwjz': float(data['dwjz'])}

total_assets = 0
for p in positions:
    d = get_tianfund_gz(p['code'])
    mktval = d['gsz'] * p['shares']
    total_assets += mktval
    print(f"{p['name']}: gsz={d['gsz']}, dwjz={d['dwjz']}, 市值={mktval:.2f}")

print(f"\n总资产（实时）: {total_assets:,.2f} 元")
```

## 修复要求

cron job prompt 需更新，确保：
1. **dwjz 必须从天天基金 JSONP 的 `dwjz` 字段读取**，不得用其他来源
2. **日涨跌 = `(gsz/dwjz − 1)×100`**，不得用 `cost` 作分母
3. **成本永远从 memory 读取**，不在任何接口中获取
4. 每次计算完成后验证：`Σ(gsz × 份额)` 应等于报告的总资产（误差<1元）

## 关联变更

- 用户于 2026-05-14 将 016129 成本价从 1.3048 更新为 1.3005（已写入 memory）
- 技能文件 SKILL.md 已增加 PITFALL 段落（diagnosis hash: `20260514-fund-nav-formula`）
