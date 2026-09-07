# 06 · 策略输出 Schema 设计 v1.0

> MarketingAdvisor 策略生成器 MVP 的"输出契约" · 2026-09-08 建
> 喂给:[#91 策略生成器 MVP](项目开发计划.md) · 复用 [01-策略-Index](01-策略-Index.md) 5 段式映射建议
> 状态:**v1.0 草案**,待 Phase 0 结束后冻结为 v1.0,再迭代 v1.1(加入预算约束、行业微调等)

## 1. 为什么需要这个 Schema

策略生成器是 MarketingAdvisor 的"门面"——用户输入品牌 + 目标 + 预算,机器输出 5 段式策略。要保证:

- **机器可读**:可被下游模块(人群洞察 / 创意工坊 / 复盘归因)直接消费
- **人可审**:CMO / 品牌经理一眼能挑错(每段 ≤ 5 个 bullet,每 bullet ≤ 30 字)
- **LLM 可生成**:Schema 本身就是 LLM 的 output parser 目标 + few-shot 模板
- **跨顾问协作**:与 18-媒体(MediaAdvisor)、05-数据(DataAdvisor)的字段衔接(下文 §6 给出 hook 点)

## 2. 顶层结构(5 段式 + 元信息)

```json
{
  "schema_version": "marketing.strategy.v1.0",
  "generated_at": "2026-09-08T03:20:00+08:00",
  "generator": "MarketingAdvisor/strategy-mvp-v1.0",
  "input_brand": "string, 必填, ≤ 40 字",
  "input_objective": "enum[认知/兴趣/转化/复购/口碑], 必填",
  "input_budget_rmb": "number, 必填, ≥ 0",
  "input_industry": "string, 可选, 用于微调(默认 '通用')",
  "input_horizon_days": "number, 必填, 7-365",

  "target_audience": { ... },   // §3 目标人群
  "value_proposition": { ... }, // §4 核心卖点
  "channel_mix":      { ... },  // §5 渠道组合
  "content_matrix":   { ... },  // §6 内容矩阵
  "kpi_plan":         { ... },  // §7 KPI

  "risk_flags":       [ ... ],  // §8 风险提示(可选)
  "reference_cases":  [ ... ]   // §9 引用案例(可选,关联 05-Campaigns)
}
```

## 3. 目标人群 `target_audience`

**输出形态**:3 个分层(核心 / 拓展 / 防御),每层 3 维(基础属性 / 消费偏好 / 媒介触达),共 9 个分层属性。

```json
"target_audience": {
  "core": {
    "label": "都市精致妈妈",
    "basic":     { "age": "28-35", "gender": "女", "city_tier": "新一线/一线", "income_rmb_month": "1.5-3 万" },
    "preference":{ "category_top3": ["母婴","美妆","家居"], "price_band": "中高端", "purchase_decision": "KOC+测评" },
    "media":     { "primary": "小红书", "secondary": "抖音", "tertiary": "微信私域", "peak_hours": "20:00-22:00" }
  },
  "expand": { ... },   // 拓展人群,格式同上
  "defend": { ... }    // 防御人群(竞品已占据但我们要抢的),格式同上
}
```

**校验规则**:
- 3 层人群的 `media.primary` 至少 2 个不同平台(避免单渠道风险)
- 每层 `purchase_decision` 必须给出触发词(不写"KOL"等空话,写"KOC 测评 + 知乎长文")

## 4. 核心卖点 `value_proposition`

**输出形态**:1 句价值主张 + 3 条支撑(USP 范式)+ 1 句反共识/差异化锚点。

```json
"value_proposition": {
  "headline": "让宝宝睡得香,妈妈睡得稳",       // ≤ 20 字,可直接做 Slogan
  "supporting": [
    "国家专利 0 甲醛母婴面料",
    "已服务 100w+ 新生儿家庭,复购率 38%",
    "三甲医院儿科主任推荐"
  ],
  "differentiator": "市面唯一通过 GB 31701-2015 婴幼儿用品 A 类认证的床品", // ≤ 40 字
  "usp_source": "07-思想与方法/USP 理论"  // 引用知识库
}
```

**校验规则**:
- `headline` 必须有动词("让/给/帮/做"等)
- `supporting` 第 1 条必须有数据/认证/专利(避免自嗨)
- `differentiator` 必须能回答"为什么选你不选竞品"

## 5. 渠道组合 `channel_mix`

**输出形态**:3 档分级(主投 / 辅投 / 测试),每档给出渠道 + 预算占比 + 角色定位。

```json
"channel_mix": {
  "primary": [
    { "channel": "小红书 信息流", "budget_pct": 40, "role": "种草+沉淀", "kpi": "CPE ≤ 8 元" }
  ],
  "secondary": [
    { "channel": "抖音 千川",   "budget_pct": 30, "role": "转化收割", "kpi": "ROAS ≥ 2.5" }
  ],
  "test": [
    { "channel": "视频号 朋友圈", "budget_pct": 10, "role": "私域复购", "kpi": "复购率 ≥ 25%" }
  ],
  "reserve_pct": 20,  // 机动预算(用于追投/调整)
  "rationale": "主投小红书打核心人群,辅投抖音收割,测试视频号跑私域闭环,留 20% 机动应对爆款",
  "mmo_model": "08-应用建模/MMO 模型 v1"  // 引用知识库
}
```

**校验规则**:
- `primary + secondary + test` 预算占比之和 ≤ 80%,余下 20% 留作 `reserve_pct`
- 至少 1 个 `secondary` 渠道(不能"全压一头")
- `rationale` ≤ 100 字,说明 3 档之间的接力逻辑

**协作 hook**:与 18-媒体顾问(MediaAdvisor)对齐时,`channel.kpi` 字段可调用 MediaAdvisor 的渠道打分能力(见 §10)。

## 6. 内容矩阵 `content_matrix`

**输出形态**:4 类内容资产(Slogan / KV / 落地页 / 媒介标题),每类给 2-3 个版本 + 平台适配。

```json
"content_matrix": {
  "slogans": [
    { "text": "睡得香,长得棒",          "version": "主推版",  "scenario": "KV 大标题" },
    { "text": "妈妈的安心,宝宝的香甜", "version": "情感版",  "scenario": "落地页 hero" },
    { "text": "0 甲醛,从第一夜开始",   "version": "卖点版",  "scenario": "信息流标题" }
  ],
  "kv_copies": [
    { "headline": "...", "subline": "...", "image_brief": "...", "platform": "小红书" },
    { "headline": "...", "subline": "...", "image_brief": "...", "platform": "抖音" }
  ],
  "landing_pages": [
    { "section": "hero",     "copy": "..." },
    { "section": "evidence", "copy": "..." },
    { "section": "cta",      "copy": "..." }
  ],
  "media_titles": [
    { "platform": "巨量",     "title": "...", "char_limit": 30 },
    { "platform": "小红书",   "title": "...", "char_limit": 20 },
    { "platform": "微信",     "title": "...", "char_limit": 14 }
  ]
}
```

**校验规则**:
- 每类 ≥ 2 个版本(主推 + 备选),便于 A/B 测试
- `media_titles` 严格遵守 `char_limit`(平台审核硬约束)

**协作 hook**:与 18-媒体顾问对齐时,`slogans` / `kv_copies` 可调用 MediaAdvisor 的选题+改写能力(见 §10)。

## 7. KPI `kpi_plan`

**输出形态**:北极星指标 1 个 + 过程指标 3-4 个 + 警戒线 + 数据来源。

```json
"kpi_plan": {
  "north_star":  { "name": "GMV",   "target": 3000, "unit": "万元", "horizon_days": 90 },
  "process": [
    { "name": "CTR",        "target": 2.5,  "unit": "%",   "source": "巨量/小红书后台" },
    { "name": "CVR",        "target": 3.0,  "unit": "%",   "source": "落地页 GA" },
    { "name": "CAC",        "target": 80,   "unit": "元",  "source": "归因模型" },
    { "name": "复购率",     "target": 25,   "unit": "%",   "source": "CRM" }
  ],
  "guardrails": [
    { "name": "ROAS 底线",  "value": 1.5,   "action_if_below": "立即下调主投渠道预算 20%" }
  ],
  "attribution_model": "08-应用建模/末次点击归因 v1"
}
```

**校验规则**:
- `north_star.target` 必须 = `horizon_days` 内的累计值,不是日均
- `process` 至少 1 个来自曝光/点击层、1 个转化层、1 个财务层
- `guardrails` 至少 1 条,带触发动作(不能只报警不动作)

## 8. 风险提示 `risk_flags`(可选)

```json
"risk_flags": [
  { "type": "data_compliance", "level": "中", "desc": "巨量/小红书用户数据需做脱敏" },
  { "type": "creative_collision", "level": "低", "desc": "Slogan '睡得香' 与某竞品 2024 春季广告撞车" },
  { "type": "budget_risk", "level": "高", "desc": "主投渠道 40% 占比过高,单渠道依赖" }
]
```

`type` 限定 4 类:`data_compliance` / `creative_collision` / `budget_risk` / `strategy_subjectivity`,便于后期按类统计。

## 9. 引用案例 `reference_cases`(可选)

策略生成时,引用 05-Campaigns 里的 1-3 个相似范本,作为"AI 辅助决策"的可信度背书。

```json
"reference_cases": [
  { "id": "#12 华为 Mate 70",     "relevance": "科技品类+新品发布+长线叙事", "reused_field": "value_proposition.differentiator" },
  { "id": "#7  安踏 巴黎奥运",    "relevance": "情绪调动+国民品牌升级",     "reused_field": "value_proposition.supporting" }
]
```

## 10. 跨顾问协作 Hook 点

| Hook 字段 | 对接顾问 | 调用能力 | 时机 |
|---|---|---|---|
| `channel_mix.primary[].kpi` | 18-媒体-Media | 渠道打分(历史 ROI/受众匹配度) | 生成渠道档位时 |
| `content_matrix.slogans` | 18-媒体-Media | 改写能力(平台调性适配) | 生成 Slogan 后追加改写 |
| `target_audience.*.media` | 05-数据-Data | 行业 TGI / 触达偏好 | 生成媒介触达字段时 |
| `kpi_plan.attribution_model` | 05-数据-Data | 归因模型选型 | 生成 KPI 时建议归因方式 |

## 11. LLM Prompt 模板(给策略生成器 MVP 直接用)

```text
你是 CMO 贴身 AI 营销顾问 MarketingAdvisor。

# 输入
- 品牌: {input_brand}
- 目标: {input_objective}
- 预算: {input_budget_rmb} 元
- 行业: {input_industry}
- 周期: {input_horizon_days} 天

# 输出
严格按以下 JSON Schema 输出(不要任何额外说明文字):
{{...嵌入 §2 顶层结构...}}

# 约束
- 每个 bullet ≤ 30 字
- 数据必须可验证(给出来源)
- 引用 05-Campaigns 时,只引"relevance"最贴近的 1-3 个

# 知识库(可选上下文)
{此处注入 01-策略-Index 摘要 + 05-Campaigns Top-3 相似案例}
```

## 12. 实施 checklist(给 MVP 任务拆)

- [ ] 把 §2 顶层 + §3-7 五个子结构翻译成 Pydantic / TypeScript 实体
- [ ] 写 JSON Schema 校验文件,挂到 FastAPI Gateway 入参校验
- [ ] 把 §11 Prompt 模板做成 LangChain `PromptTemplate`,挂在 `marketing.strategy.v1` chain 上
- [ ] 落 1-2 个 e2e fixture(从真实 05-Campaigns #7 #8 #12 各抽 1 个反推 input,验证输出可解析)
- [ ] 跨顾问 hook 暂时留 TODO,等 18-媒体 顾问出 MediaAdvisor API 文档后再接

## 13. 版本与变更

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 草案 | 2026-09-08 | 初版:5 段式 + 元信息 + 风险 + 引用案例 + 跨顾问 hook |

## 14. 关联文档

- [01-策略-Index](01-策略-Index.md) — 5 段式策略生成器映射建议(本文件是其工程化落地)
- [00-总索引](00-总索引.md) — Inspiration 库总体结构
- [05-Campaigns/](05-Campaigns/) — 12 个种子案例(策略生成时的 few-shot 来源)
- [项目开发计划.md §5 #86](项目开发计划.md) — 本任务的来源条目
- [项目开发计划.md §6 #91](项目开发计划.md) — 策略生成器 MVP(本 Schema 的消费者)
