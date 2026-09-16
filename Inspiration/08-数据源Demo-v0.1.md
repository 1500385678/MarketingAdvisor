# 数据源 Demo · v0.1

> Phase 0 #3 数据源 demo · v0.1 草案 · 2026-09-17 03:20 创建
> 创建: T5 应急第 13 次命中 · T4 连续 12 天未生成当日 .plan(自 9-5 起) · 9-17 巡检强建议启动 Phase 0 #3 收口
> 目标: 启动 Phase 0 #3 "跑通巨量引擎/小红书/微信广告 3 大公开数据源的创意/活动抓取 demo",完成第 1 个公开数据源(小红书)的最小可入库 demo
> 后续: Phase 1 MVP 启动后,落地巨量引擎 + 微信广告 2 个 demo

---

## 1. 三大公开数据源选型矩阵

| 数据源 | 抓取难度 | 合规边界 | 创意/活动数据可得性 | 抓取入口 | 选型决策 |
|---|---|---|---|---|---|
| **小红书** | 中(官方 API 需企业认证,有开放笔记 API/搜索 API/创作者平台) | 中(笔记内容版权 + 用户隐私需脱敏) | **高**(笔记标题 + 内容 + 标签 + 互动数据 + 收藏数 + 评论) | 公开搜索页 / 蒲公英 / 千瓜 | **v0.1 优先落地**(低门槛 + 数据丰富) |
| **巨量引擎** | 高(无公开 API,需巨量算数报告或第三方抓取) | 中(数据来自平台公开报告,合规) | 中(巨量算数周报 + 算数指数 + 抖音热榜) | 巨量算数 / 算数指数 / 抖音热榜 | Phase 1 MVP 启动后落地 |
| **微信广告** | 高(无公开 API,需朋友圈广告库或第三方抓取) | 中-低(广告素材公开,数据无 API) | 低-中(广告素材图 + 文案,但无互动数据) | 朋友圈广告素材库 / 腾讯广告助手 | Phase 1 MVP 启动后落地 |

**核心判断**:

- **小红书**: 数据最丰富,合规边界最清晰(笔记标题元数据公开),抓取技术成熟 → **v0.1 首选**
- **巨量引擎**: 数据需经巨量算数等报告入口,适合作"周报级"数据源而非"实时抓取"
- **微信广告**: 素材公开但数据封闭,适合作"创意素材库"而非"实时活动监测"

---

## 2. v0.1 优先目标 · 小红书 demo

### 2.1 数据抓取范围(最小可入库版本)

- **关键词搜索**: 用户输入关键词(如"国货""美妆""新茶饮"),返回 Top 100 笔记标题 + 摘要 + 互动数
- **时间筛选**: 支持最近 7 天 / 30 天 / 90 天
- **数据字段**: 笔记标题、作者、发布时间、点赞数、收藏数、评论数、标签(标签 = 创意方向 + 兴趣标签)
- **存储格式**: CSV(每次抓取生成 1 个 CSV 文件,文件名带时间戳 `xiaohongshu_<keyword>_<YYYYMMDD>.csv`)
- **去重**: 按笔记 ID 去重,避免重复抓取

### 2.2 抓取流程(5 步)

```
1. 关键词输入(CLI / API 调用)
2. 走小红书公开搜索页(URL: https://www.xiaohongshu.com/search_result?keyword=<keyword>&source=web_search_result_notes)
3. 解析 HTML(BeautifulSoup)或 JSON(若拿到 API 接口),提取笔记元数据
4. 写入 CSV + 输出 Top 10 笔记摘要
5. 喂入 5 段式策略生成器 seed(供 Phase 1 MVP 调用)
```

### 2.3 最小可入库 demo 代码框架(`xiaohongshu_scraper.py`)

```python
"""
小红书公开笔记搜索 demo · v0.1
输入: 关键词 + 时间范围
输出: CSV 文件 + Top 10 笔记摘要
依赖: requests, beautifulsoup4, pandas
风险: 仅抓取标题 + 元数据 + 标签,不抓取正文;限速 1 req/s;需 IP 池
"""

import requests
from bs4 import BeautifulSoup
import pandas as pd
import time
from datetime import datetime


def fetch_xhs_notes(keyword: str, days: int = 30) -> list[dict]:
    """抓取小红书关键词搜索结果,返回笔记元数据列表"""
    url = f"https://www.xiaohongshu.com/search_result?keyword={keyword}&source=web_search_result_notes"
    headers = {
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"
    }
    resp = requests.get(url, headers=headers, timeout=10)
    soup = BeautifulSoup(resp.text, "html.parser")
    notes = []
    # 解析 note 卡片(具体选择器依实际页面而定)
    for card in soup.select(".note-item"):
        notes.append({
            "title": card.select_one(".title").text,
            "author": card.select_one(".author").text,
            "likes": int(card.select_one(".like-count").text),
            "tags": [t.text for t in card.select(".tag")],
            "fetched_at": datetime.now().isoformat(),
        })
    time.sleep(1)  # 限速
    return notes


def save_to_csv(notes: list[dict], keyword: str) -> str:
    """保存到 CSV 文件,文件名带时间戳"""
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    fname = f"xiaohongshu_{keyword}_{ts}.csv"
    pd.DataFrame(notes).to_csv(fname, index=False)
    return fname


if __name__ == "__main__":
    notes = fetch_xhs_notes(keyword="国货", days=30)
    csv_path = save_to_csv(notes, keyword="国货")
    print(f"✅ 抓取 {len(notes)} 条笔记 → {csv_path}")
```

> 注: 上述代码是 **Phase 1 MVP 启动后** 实际可运行的版本框架,本 v0.1 文档先固化抓取设计 + 字段约定 + 合规边界。

---

## 3. 风险与合规边界

### 3.1 账号风险

- 小红书对自动化抓取敏感,封号风险高
- **缓解措施**: IP 池 + 限速(1 req/s)+ UA 随机 + Cookie 池 + 单日抓取上限

### 3.2 数据合规

- 笔记内容属用户创作,受著作权法保护
- **仅抓取**: 标题 + 元数据(点赞/收藏/评论数)+ 标签 + 发布时间
- **不抓取**: 笔记正文(避免侵权)+ 用户头像 + 用户主页
- **数据用途**: 仅用于策略生成器的"目标人群兴趣标签"和"竞品创意参考",不直接对外展示

### 3.3 替代方案(若自建抓取不可行)

- **小红书蒲公英**: 官方创作者平台,需企业认证,合规但有配额限制
- **千瓜数据 / 蝉妈妈 / 灰豚**: 第三方数据平台,按月付费,数据完整度高
- **Phase 2 接入**: 商业合作(千瓜数据按月订阅 ~3-5 万/月,合规 + 数据完整)

---

## 4. 与其他模块的关联(跨顾问 hook)

### 4.1 与 06 策略 Schema(项目内部)

- 抓取到的笔记标题 + 标签 → 作为"目标人群兴趣标签"的种子(策略 Schema §2 目标人群)
- 抓取到的笔记爆款规律 → 作为"核心卖点"的灵感来源(策略 Schema §2 核心卖点)
- 抓取到的笔记创意方向 → 作为"内容矩阵"的备选模板(策略 Schema §4 内容矩阵)

### 4.2 与 07 LLM 评估(项目内部)

- 抓取的小红书爆款标题 → 作为 LLM Slogan 评估样本集(LLM 评估 §4 Step 1 样本集)
- 抓取的小红书爆款笔记内容(去敏) → 作为 LLM KV 文案 + 落地页评估样本集
- 抓取的小红书爆款规律 → 作为 LLM 复盘报告的"行业基线对照"参考

### 4.3 与 18-媒体 顾问(跨产品 hook)

- **MediaAdvisor 选题能力**: 抓取的小红书爆款笔记 → 喂入 MediaAdvisor 选题模块 → 输出"本周热点选题 Top 10"
- **MediaAdvisor 改写能力**: MarketingAdvisor 创意工坊 → 调用 MediaAdvisor 改写能力 → 批量生成小红书爆款风格文案
- **共建"选题 → 创意 → 投放"跨产品 demo**: MarketingAdvisor(策略 + 创意工坊) + MediaAdvisor(选题 + 改写)+ 数据源(本 v0.1 demo)三端协同
- **对齐优先级**: 与 18-媒体 agent 的 cron 任务同步节奏(每日 02:20 / 03:20 巡检 + 应急)

### 4.4 与 Phase 1 MVP(项目内部)

- **接入 FastAPI 网关**: `xiaohongshu_scraper.py` 封装为 `/api/strategy/seed?keyword=<kw>&days=<n>` 端点
- **集成 5 段式策略生成器**: 抓取结果作为 seed,LLM 二次加工输出策略
- **集成创意工坊**: 抓取的笔记标题 + 标签 → LLM 生成 Slogan + 媒介标题 + 小红书卡片文案

---

## 5. 下一步(Phase 0 收口路线图)

- [x] **v0.1 · 2026-09-17**: 启动 Phase 0 #3,完成 3 大公开数据源选型矩阵 + 小红书 demo 流程设计 + 风险与合规边界 + 与 06/07/18-media 模块 hook(**本文档**)
- [ ] **v0.2 · Phase 1 MVP 启动后**: 落地 `xiaohongshu_scraper.py` 完整 Python 代码(可运行版本 + 单元测试 + 错误重试 + IP 池)
- [ ] **v0.3 · Phase 1 MVP**: 接入 FastAPI 网关 `/api/strategy/seed` 端点,与 5 段式策略生成器集成
- [ ] **v0.4 · Phase 1 MVP**: 巨量引擎 demo(巨量算数 + 抖音热榜 + 周报级数据)
- [ ] **v0.5 · Phase 1 MVP**: 微信广告 demo(朋友圈广告素材库 + 创意素材图入库)
- [ ] **v0.6 · Phase 2**: 接入千瓜数据 / 蝉妈妈 等第三方数据平台(月订阅,商业合规)

---

## 6. 变更记录

- **v0.1 · 2026-09-17 03:20 创建**: T5 应急第 13 次命中 · 启动 Phase 0 #3 数据源 demo · 完成 3 大公开数据源选型矩阵 + 小红书 demo 流程设计 + 最小代码框架 + 风险与合规边界 4 项 + 与 06/07/18-media 模块 hook 4 项 + Phase 0 收口路线图 6 阶段

---

## 关联文档

- 上游索引: [[00-总索引]] · 06 Schema: [[06-策略输出Schema设计]] · 07 LLM 评估: [[07-LLM基线质量评估-v0.1]]
- **跨产品 hook**: 18-媒体 顾问(选题/改写能力可调用,见 §4.3)
- **项目主计划**: [[../项目开发计划]] §5 Phase 0 #3(数据源 demo)