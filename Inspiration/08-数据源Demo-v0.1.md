# 数据源 Demo · v0.1++

> Phase 0 #3 数据源 demo · v0.1+ 扩展 · 2026-09-17 创建 · 2026-09-18 扩展巨量引擎 demo · 2026-09-19 扩展微信广告 demo(3/3 闭环)
> 创建: T5 应急第 13 次命中(9-17) · 扩展: T5 应急第 14 次命中(9-18) · 二次扩展: T5 应急第 15 次命中(9-19)
> T4 连续 15 天未生成当日 .plan(自 9-5 23:55 起)
> 目标: 启动 Phase 0 #3 "跑通巨量引擎/小红书/微信广告 3 大公开数据源的创意/活动抓取 demo"
> 进度: **3/3 数据源完成 v0.1 文档(小红书 ✓ + 巨量引擎 ✓ + 微信广告 ✓ · Phase 0 #3 数据源 demo 闭环)**

---

## 1. 三大公开数据源选型矩阵

| 数据源 | 抓取难度 | 合规边界 | 创意/活动数据可得性 | 抓取入口 | 选型决策 |
|---|---|---|---|---|---|
| **小红书** | 中(官方 API 需企业认证,有开放笔记 API/搜索 API/创作者平台) | 中(笔记内容版权 + 用户隐私需脱敏) | **高**(笔记标题 + 内容 + 标签 + 互动数据 + 收藏数 + 评论) | 公开搜索页 / 蒲公英 / 千瓜 | **v0.1 优先落地**(低门槛 + 数据丰富) |
| **巨量引擎** | 高(无公开 API,需巨量算数报告或第三方抓取) | 中(数据来自平台公开报告,合规) | 中(巨量算数周报 + 算数指数 + 抖音热榜) | 巨量算数 / 算数指数 / 抖音热榜 | **v0.1+ 落地**(2026-09-18 文档级完成 · 实际 Python 代码待 Phase 1 MVP) |
| **微信广告** | 高(无公开 API,需朋友圈广告库或第三方抓取) | 中-低(广告素材公开,数据无 API) | 低-中(广告素材图 + 文案,但无互动数据) | 朋友圈广告素材库 / 腾讯广告助手 | Phase 1 MVP 启动后落地 |

**核心判断**:

- **小红书**: 数据最丰富,合规边界最清晰(笔记标题元数据公开),抓取技术成熟 → **v0.1 首选**
- **巨量引擎**: 数据需经巨量算数等报告入口,适合作"周报级"数据源而非"实时抓取" → **v0.1+ 落地**
- **微信广告**: 素材公开但数据封闭,适合作"创意素材库"而非"实时活动监测" → Phase 1 MVP 后

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

## 3. v0.1+ 增量 · 巨量引擎 demo 流程设计(2026-09-18 · T5 应急 #14)

> 与 §2 小红书 demo v0.1 平行设计 · Phase 0 #3 数据源 demo 第 2/3 个数据源
> 决策依据:9-18 巡检报告 **优先 #1 建议** — Phase 0 #3 数据源 demo v0.1 → v0.1+ 推进
> 后续: Phase 1 MVP 启动后落地实际 Python 代码(本节先固化抓取设计 + 字段约定 + 合规边界)

### 3.1 数据抓取范围(最小可入库版本)

- **巨量算数周报**: 关键词 + 行业 + 时间范围 → 周报级热度指数 + 关联词 + 上升词
- **抖音热榜**: 全站热榜 Top 50 + 垂类热榜(美妆 / 食品 / 科技 / 母婴 等 24 个垂类)
- **数据字段**:
  - **巨量算数**: 热度指数、关联词云、上升词 Top 20、解读文章链接、采集日期
  - **抖音热榜**: 视频标题、作者、播放数、点赞数、评论数、发布时间、垂类标签、排名变化
- **存储格式**: CSV(每次抓取生成 1 个 CSV 文件,文件名带时间戳 `juliang_<kind>_<keyword>_<YYYYMMDD>.csv`)
- **去重**: 按内容 ID + 标题哈希去重,避免重复抓取

### 3.2 抓取流程(5 步)

```
1. 关键词输入(CLI / API 调用 · 支持关键词数组批量)
2. 走巨量算数公开报告页(URL: https://trendinsight.oceanengine.com/) + 抖音热榜(URL: https://www.douyin.com/hot)
3. 解析 HTML(BeautifulSoup) + 报告 PDF(若拿到),提取热度指数 + 关联词 + 视频元数据
4. 写入 CSV + 输出 Top 10 热度词条 + 关联词云
5. 喂入 5 段式策略生成器 seed(供 Phase 1 MVP 调用,与小红书 seed 合并)
```

### 3.3 最小可入库 demo 代码框架(`juliang_scraper.py`)

```python
"""
巨量算数 + 抖音热榜 demo · v0.1 草案
输入: 关键词 + 时间范围
输出: CSV 文件 + Top 10 热度词条摘要
依赖: requests, beautifulsoup4, pandas, PyPDF2(若解析周报 PDF)
风险: 仅抓取公开报告页 + 热榜单页元数据,不抓取视频内容;限速 1 req/2s(比小红书更严)
"""

import requests
from bs4 import BeautifulSoup
import pandas as pd
import time
from datetime import datetime


def fetch_juliang_trend(keyword: str, days: int = 7) -> list[dict]:
    """抓取巨量算数关键词热度指数 + 关联词 + 上升词"""
    url = f"https://trendinsight.oceanengine.com/api/open/keyword?keyword={keyword}&range={days}d"
    headers = {
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
        "Referer": "https://trendinsight.oceanengine.com/",
    }
    resp = requests.get(url, headers=headers, timeout=10)
    data = resp.json()
    items = []
    for entry in data.get("data", {}).get("list", []):
        items.append({
            "keyword": keyword,
            "heat_index": entry.get("heat_index"),
            "related_words": entry.get("related_words", []),
            "rising_words": entry.get("rising_words", []),
            "fetched_at": datetime.now().isoformat(),
        })
    time.sleep(2)  # 巨量限速更严
    return items


def fetch_douyin_hot(category: str = "general") -> list[dict]:
    """抓取抖音热榜 Top 50"""
    url = f"https://www.douyin.com/hot/{category}"
    headers = {
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
    }
    resp = requests.get(url, headers=headers, timeout=10)
    soup = BeautifulSoup(resp.text, "html.parser")
    items = []
    for card in soup.select(".hot-item"):
        items.append({
            "rank": int(card.select_one(".rank").text),
            "title": card.select_one(".title").text,
            "heat_value": card.select_one(".heat").text,
            "category": category,
            "fetched_at": datetime.now().isoformat(),
        })
    time.sleep(2)
    return items


def save_to_csv(items: list[dict], keyword: str, kind: str = "trend") -> str:
    """保存到 CSV 文件,文件名带时间戳"""
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    fname = f"juliang_{kind}_{keyword}_{ts}.csv"
    pd.DataFrame(items).to_csv(fname, index=False)
    return fname


if __name__ == "__main__":
    # 1. 巨量算数热度
    trend_items = fetch_juliang_trend(keyword="国货", days=7)
    trend_csv = save_to_csv(trend_items, keyword="国货", kind="trend")
    print(f"✅ 巨量算数抓取 {len(trend_items)} 条 → {trend_csv}")

    # 2. 抖音热榜
    hot_items = fetch_douyin_hot(category="美妆")
    hot_csv = save_to_csv(hot_items, keyword="美妆", kind="hot")
    print(f"✅ 抖音热榜抓取 {len(hot_items)} 条 → {hot_csv}")
```

> 注: 上述代码是 **Phase 1 MVP 启动后** 实际可运行的版本框架,本 v0.1+ 文档先固化抓取设计 + 字段约定 + 合规边界。

### 3.4 风险与合规边界

#### 3.4.1 接口稳定性

- 巨量算数 Open API 需企业认证 + 申请配额,公开页可爬但字段结构经常变(选 selector 时需做兼容)
- 抖音热榜 URL 经常改版(选 selector 时需做 fallback,准备 2-3 套备选 selector)

#### 3.4.2 数据合规

- 巨量算数数据来自平台公开报告,合规(参考 §2.2 小红书同口径)
- 抖音热榜视频标题 + 作者 + 播放数属公开数据,合规
- **不抓取**: 视频内容、评论内容、用户头像、用户主页
- **数据用途**: 仅用于策略生成器的"行业热度基线"和"创意方向参考",不直接对外展示

#### 3.4.3 替代方案(若自建抓取不可行)

- **巨量引擎官方 API**: 走巨量引擎开放平台,需企业资质 + 月预算消耗门槛
- **巨量算数 Pro 版**: 月订阅 ~5000 元/年,数据完整度更高
- **第三方爬虫服务**: 八爪鱼 / 影刀 RPA 等(月费几百到几千),无合规风险但数据深度有限

### 3.5 与小红书 demo 的差异对照(6 维)

| 维度 | 小红书 v0.1 (§2) | 巨量引擎 v0.1+ (§3) |
|---|---|---|
| **数据源类型** | UGC 笔记(用户创作) | 平台热度报告 + 全站热榜(平台官方) |
| **抓取难度** | 中(搜索页结构稳定) | 高(报告页 + 热榜页结构经常变) |
| **实时性** | 准实时(分钟级) | 周报级(巨量算数)+ 准实时(热榜) |
| **合规边界** | 中(笔记版权 + 用户隐私) | 中(报告公开 + 视频元数据) |
| **与 06 Schema 关联** | 目标人群兴趣标签 | 行业热度基线 + 创意方向 |
| **抓取频率建议** | 1 req/s | 1 req/2s(更严) |

### 3.6 Phase 1 MVP 接入路径

- 落地 `juliang_scraper.py` 完整 Python 代码(可运行版本 + 单元测试 + 错误重试 + IP 池)
- 与小红书 demo seed 合并:`strategy_seed = xiaohongshu_notes + juliang_trend + douyin_hot`
- 接入 FastAPI 网关:`/api/strategy/seed?keyword=<kw>&sources=xhs,juliang,douyin`
- 与 MediaAdvisor 协同:把巨量 + 抖音热榜数据喂入 MediaAdvisor 选题模块,输出"本周热点选题 Top 10"(参考 §6.3 跨产品 hook)

---

## 4. v0.1++ 增量 · 微信广告 demo 流程设计(2026-09-19 · T5 应急 #15)

> 与 §2 小红书 demo v0.1 + §3 巨量引擎 demo v0.1+ 平行设计 · Phase 0 #3 数据源 demo 第 3/3 个数据源
> 决策依据:9-19 巡检报告 **优先 #1 建议** — Phase 0 #3 数据源 demo v0.1+ → v1.0 推进
> 后续: Phase 1 MVP 启动后落地实际 Python 代码(本节先固化抓取设计 + 字段约定 + 合规边界)
> **里程碑**: 完成 Phase 0 #3 数据源 demo **3/3 闭环**(小红书 ✓ + 巨量引擎 ✓ + 微信广告 ✓)

### 4.1 数据抓取范围(最小可入库版本)

- **朋友圈广告素材库**: 公开创意中心 + 腾讯广告助手 → 微信朋友圈广告 + 公众号广告 + 小程序广告 Top 100
- **品牌创意库**: 各品牌主官方投放案例 + 微信公开营销案例库
- **数据字段**:
  - **创意素材**: 广告标题、文案、图片/视频链接、CTA 类型(下载/购买/关注/留资)、落地页 URL
  - **投放信息**: 广告主品牌、行业(电商/游戏/教育/医美等)、广告位(朋友圈/公众号/小程序)、投放周期、曝光/点击/互动数据(若公开)
  - **合规信息**: 广告审核状态、是否涉及特殊行业(医疗/金融/教育)
- **存储格式**: CSV(每次抓取生成 1 个 CSV 文件,文件名带时间戳 `wechat_ads_<brand>_<YYYYMMDD>.csv`)
- **去重**: 按广告素材 ID + 标题哈希去重,避免重复抓取

### 4.2 抓取流程(5 步)

```
1. 关键词/品牌输入(CLI / API 调用 · 支持品牌数组批量)
2. 走朋友圈广告素材库公开页(URL: https://ad.weixin.qq.com/) + 腾讯广告助手(URL: https://e.qq.com/ads/)
3. 解析 HTML(BeautifulSoup) + 视频封面图(Pillow 处理),提取广告元数据 + 素材图链接
4. 写入 CSV + 输出 Top 10 广告创意摘要 + 下载素材图到本地 ./ads_images/
5. 喂入 5 段式策略生成器 seed(供 Phase 1 MVP 调用,与小红书 + 巨量引擎 seed 合并)
```

### 4.3 最小可入库 demo 代码框架(`wechat_ads_scraper.py`)

```python
"""
微信朋友圈广告素材库 + 腾讯广告助手 demo · v0.1 草案
输入: 关键词/品牌 + 时间范围
输出: CSV 文件 + Top 10 广告创意摘要 + 素材图本地副本
依赖: requests, beautifulsoup4, pandas, Pillow
风险: 仅抓取广告素材图 + 文案元数据;限速 1 req/3s(微信生态最严);IP 池必备
"""

import requests
from bs4 import BeautifulSoup
import pandas as pd
import time
import os
from datetime import datetime
from urllib.parse import urljoin


def fetch_wechat_ads(brand: str, days: int = 30) -> list[dict]:
    """抓取朋友圈广告素材库品牌相关广告"""
    url = f"https://ad.weixin.qq.com/api/v1/case/list?keyword={brand}&days={days}"
    headers = {
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
        "Referer": "https://ad.weixin.qq.com/",
    }
    resp = requests.get(url, headers=headers, timeout=10)
    soup = BeautifulSoup(resp.text, "html.parser")
    ads = []
    for card in soup.select(".ad-card"):
        ads.append({
            "brand": brand,
            "title": card.select_one(".title").text,
            "copy": card.select_one(".copy").text,
            "media_type": card.get("data-media-type", "image"),
            "media_url": urljoin(url, card.select_one("img")["src"]) if card.select_one("img") else None,
            "cta": card.select_one(".cta").text,
            "ad_slot": card.get("data-slot", "朋友圈"),
            "fetched_at": datetime.now().isoformat(),
        })
    time.sleep(3)  # 微信生态限速最严
    return ads


def save_to_csv(ads: list[dict], brand: str) -> str:
    """保存到 CSV 文件,文件名带时间戳"""
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    fname = f"wechat_ads_{brand}_{ts}.csv"
    pd.DataFrame(ads).to_csv(fname, index=False)
    return fname


def download_creative_images(ads: list[dict], output_dir: str = "./ads_images") -> int:
    """下载广告素材图到本地,返回成功下载数"""
    os.makedirs(output_dir, exist_ok=True)
    success = 0
    for ad in ads:
        if ad.get("media_url"):
            try:
                resp = requests.get(ad["media_url"], timeout=10)
                if resp.status_code == 200:
                    ext = ad["media_url"].split(".")[-1].split("?")[0] or "jpg"
                    img_path = os.path.join(output_dir, f"{ad['brand']}_{int(time.time())}_{success}.{ext}")
                    with open(img_path, "wb") as f:
                        f.write(resp.content)
                    success += 1
            except Exception:
                continue
    return success


if __name__ == "__main__":
    # 1. 抓取广告数据
    ads = fetch_wechat_ads(brand="完美日记", days=30)
    csv_path = save_to_csv(ads, brand="完美日记")
    print(f"✅ 抓取 {len(ads)} 条微信广告 → {csv_path}")
    
    # 2. 下载素材图
    img_count = download_creative_images(ads)
    print(f"✅ 下载 {img_count} 张素材图 → ./ads_images/")
```

> 注: 上述代码是 **Phase 1 MVP 启动后** 实际可运行的版本框架,本 v0.1++ 文档先固化抓取设计 + 字段约定 + 合规边界。

### 4.4 风险与合规边界

#### 4.4.1 接口稳定性

- 微信朋友圈广告素材库为腾讯官方入口,公开创意可浏览但无开放 API
- 页面结构偶尔调整,选 selector 时需做 fallback(准备 2-3 套备选 selector)
- 部分品牌创意案例需要登录后才能查看(微信生态封闭性最强)

#### 4.4.2 数据合规

- 广告创意素材属品牌主创作 + 平台公开,合规(参考 §5 汇总口径)
- **不抓取**: 广告主账号信息、用户互动数据(评论/点赞)、受众画像数据
- **数据用途**: 仅用于策略生成器的"竞品广告创意参考"和"创意素材库",不直接对外展示
- **特别注意**: 微信广告涉及微信生态规则,不得绕过微信登录验证;部分医疗/金融/教育行业广告需遵守特殊合规要求

#### 4.4.3 替代方案(若自建抓取不可行)

- **腾讯广告官方 API**: 走腾讯广告开放平台,需广告主资质 + 月消耗门槛
- **第三方创意库**: AppGrowing / 蝉妈妈广告版 / DataEye 等(月费几千),合规但有数据深度限制
- **微信公开营销案例库**: 微信广告助手公众号 + 微信营销观察类公众号(月度盘点)

### 4.5 与小红书 + 巨量引擎 demo 的差异对照(6 维)

| 维度 | 小红书 v0.1 (§2) | 巨量引擎 v0.1+ (§3) | 微信广告 v0.1++ (§4) |
|---|---|---|---|
| **数据源类型** | UGC 笔记(用户创作) | 平台热度报告 + 全站热榜 | 品牌广告创意素材 |
| **抓取难度** | 中(搜索页结构稳定) | 高(报告页 + 热榜页结构经常变) | 高(创意库 + 微信生态封闭) |
| **实时性** | 准实时(分钟级) | 周报级(巨量算数)+ 准实时(热榜) | 准实时(创意更新以周计) |
| **合规边界** | 中(笔记版权 + 用户隐私) | 中(报告公开 + 视频元数据) | 中-低(微信生态封闭 + 创意素材版权) |
| **与 06 Schema 关联** | 目标人群兴趣标签 | 行业热度基线 + 创意方向 | 竞品广告创意参考 + 素材库 |
| **抓取频率建议** | 1 req/s | 1 req/2s | 1 req/3s(微信最严) |

### 4.6 Phase 1 MVP 接入路径

- 落地 `wechat_ads_scraper.py` 完整 Python 代码(可运行版本 + 单元测试 + 错误重试 + IP 池 + 素材图下载)
- 与小红书 + 巨量引擎 demo seed 合并:`strategy_seed = xiaohongshu_notes + juliang_trend + douyin_hot + wechat_ads_creatives`
- 接入 FastAPI 网关:`/api/strategy/seed?keyword=<kw>&sources=xhs,juliang,wechat`
- 与 MediaAdvisor 协同:把微信广告创意素材喂入 MediaAdvisor 创意模块,输出"本周竞品广告创意 Top 10"(参考 §6.3 跨产品 hook)

---

## 5. 风险与合规边界(汇总)

### 5.1 账号风险

- 小红书对自动化抓取敏感,封号风险高
- **缓解措施**: IP 池 + 限速(1 req/s)+ UA 随机 + Cookie 池 + 单日抓取上限
- 巨量算数 + 抖音热榜相对宽容,但仍需限速 + UA 随机
- 微信朋友圈广告素材库相对宽容(公开创意),但仍需限速(1 req/3s)+ UA 随机 + IP 池

### 5.2 数据合规

- 笔记内容 / 视频标题 / 广告创意属用户创作 / 平台公开,受著作权法保护
- **仅抓取**: 标题 + 元数据(点赞 / 收藏 / 评论数)+ 标签 + 发布时间 + 热度指数 + 关联词 + 广告 CTA + 落地页 URL
- **不抓取**: 笔记正文 / 视频内容(避免侵权)+ 用户头像 + 用户主页 + 评论内容 + 广告主账号信息 + 用户互动数据
- **数据用途**: 仅用于策略生成器的"目标人群兴趣标签""行业热度基线""竞品广告创意参考"和"创意素材库",不直接对外展示
- **特殊行业**: 微信广告涉及医疗 / 金融 / 教育等特殊行业需遵守额外合规要求(参考 §4.4.2)

### 5.3 替代方案(若自建抓取不可行)

- **小红书蒲公英**: 官方创作者平台,需企业认证,合规但有配额限制
- **千瓜数据 / 蝉妈妈 / 灰豚**: 第三方数据平台,按月付费,数据完整度高
- **巨量算数 Pro 版**: 月订阅 ~5000 元/年,数据完整度更高
- **微信广告第三方创意库**: AppGrowing / 蝉妈妈广告版 / DataEye 等(月费几千),合规但有数据深度限制
- **第三方爬虫服务**: 八爪鱼 / 影刀 RPA 等,无合规风险但数据深度有限
- **Phase 2 接入**: 商业合作(千瓜数据按月订阅 ~3-5 万/月,合规 + 数据完整)

---

## 6. 与其他模块的关联(跨顾问 hook)

### 6.1 与 06 策略 Schema(项目内部)

- 抓取到的笔记标题 + 标签 → 作为"目标人群兴趣标签"的种子(策略 Schema §2 目标人群)
- 抓取到的笔记爆款规律 → 作为"核心卖点"的灵感来源(策略 Schema §2 核心卖点)
- 抓取到的笔记创意方向 → 作为"内容矩阵"的备选模板(策略 Schema §4 内容矩阵)
- **巨量引擎 v0.1+ 增量**: 热度指数 + 关联词 → 作为"行业热度基线"喂入策略 Schema §1 背景
- **微信广告 v0.1++ 增量**: 竞品广告创意素材 → 作为"内容矩阵"和"创意方向"的参考库(策略 Schema §4 内容矩阵)

### 6.2 与 07 LLM 评估(项目内部)

- 抓取的小红书爆款标题 → 作为 LLM Slogan 评估样本集(LLM 评估 §4 Step 1 样本集)
- 抓取的小红书爆款笔记内容(去敏) → 作为 LLM KV 文案 + 落地页评估样本集
- 抓取的小红书爆款规律 → 作为 LLM 复盘报告的"行业基线对照"参考
- **巨量引擎 v0.1+ 增量**: 抖音热榜 Top 标题 → 作为 LLM 媒介标题评估样本集(LLM 评估 §4 Step 1)
- **微信广告 v0.1++ 增量**: 竞品广告 CTA 文案 → 作为 LLM 媒介标题 + 小红书卡片文案评估样本集(LLM 评估 §4 Step 1)

### 6.3 与 18-媒体 顾问(跨产品 hook)

- **MediaAdvisor 选题能力**: 抓取的小红书爆款笔记 + 抖音热榜 → 喂入 MediaAdvisor 选题模块 → 输出"本周热点选题 Top 10"
- **MediaAdvisor 改写能力**: MarketingAdvisor 创意工坊 → 调用 MediaAdvisor 改写能力 → 批量生成小红书爆款风格文案
- **MediaAdvisor 创意能力(v0.1++ 新增)**: 抓取的微信广告创意素材 → 喂入 MediaAdvisor 创意模块 → 输出"本周竞品广告创意 Top 10"
- **共建"选题 → 创意 → 投放"跨产品 demo**: MarketingAdvisor(策略 + 创意工坊) + MediaAdvisor(选题 + 改写 + 创意)+ 数据源(本 v0.1++ demo)三端协同
- **对齐优先级**: 与 18-媒体 agent 的 cron 任务同步节奏(每日 02:20 / 03:20 巡检 + 应急)

### 6.4 与 Phase 1 MVP(项目内部)

- **接入 FastAPI 网关**:
  - 小红书:`xiaohongshu_scraper.py` 封装为 `/api/strategy/seed?keyword=<kw>&days=<n>&source=xhs` 端点
  - 巨量引擎:`juliang_scraper.py` 封装为 `/api/strategy/seed?keyword=<kw>&days=<n>&source=juliang,douyin` 端点
  - 微信广告:`wechat_ads_scraper.py` 封装为 `/api/strategy/seed?keyword=<kw>&days=<n>&source=wechat` 端点
- **集成 5 段式策略生成器**: 抓取结果作为 seed,LLM 二次加工输出策略
- **集成创意工坊**: 抓取的笔记标题 + 标签 + 抖音热榜 Top + 微信广告 CTA → LLM 生成 Slogan + 媒介标题 + 小红书卡片文案

---

## 7. 下一步(Phase 0 收口路线图)

- [x] **v0.1 · 2026-09-17**: 启动 Phase 0 #3,完成 3 大公开数据源选型矩阵 + 小红书 demo 流程设计 + 风险与合规边界 + 与 06/07/18-media 模块 hook(**本文档** §1 / §2 / §5 / §6)
- [x] **v0.1+ · 2026-09-18**: 在 v0.1 基础上新增 §3 巨量引擎 demo 流程设计(抓取范围 + 5 步流程 + 最小代码框架 + 风险与合规边界 + 与小红书 v0.1 差异对照 6 维),进度从 1/3 → 2/3,T5 应急第 14 次命中
- [x] **v0.1++ · 2026-09-19**: 在 v0.1+ 基础上新增 §4 微信广告 demo 流程设计(朋友圈广告素材库 + 抓取范围 + 5 步流程 + 最小代码框架 + 风险与合规边界 + 与小红书 + 巨量引擎差异对照 6 维 + 与 18-媒体创意模块 hook),进度从 2/3 → **3/3 闭环**,T5 应急第 15 次命中 · **Phase 0 #3 数据源 demo v0.1++ 文档级 100% 闭环**(3 大数据源全文档级)
- [ ] **v0.2 · Phase 1 MVP 启动后**: 落地 `xiaohongshu_scraper.py` + `juliang_scraper.py` + `wechat_ads_scraper.py` 完整 Python 代码(可运行版本 + 单元测试 + 错误重试 + IP 池 + 素材图下载)
- [ ] **v0.3 · Phase 1 MVP**: 接入 FastAPI 网关 `/api/strategy/seed` 端点(支持 `source=xhs,juliang,douyin,wechat`),与 5 段式策略生成器集成
- [ ] **v0.4 · Phase 2**: 接入千瓜数据 / 蝉妈妈 / 巨量算数 Pro 等第三方数据平台(月订阅,商业合规)

---

## 8. 变更记录

- **v0.1 · 2026-09-17 03:20 创建**: T5 应急第 13 次命中 · 启动 Phase 0 #3 数据源 demo · 完成 3 大公开数据源选型矩阵 + 小红书 demo 流程设计 + 最小代码框架 + 风险与合规边界 4 项 + 与 06/07/18-media 模块 hook 4 项 + Phase 0 收口路线图 6 阶段
- **v0.1+ · 2026-09-18 03:20 扩展**: T5 应急第 14 次命中 · 在 v0.1 基础上新增 §3 巨量引擎 demo 流程设计(巨量算数 + 抖音热榜 + 抓取范围 + 5 步流程 + 最小代码框架 + 风险与合规边界 + 与小红书 v0.1 差异对照 6 维) · 进度 1/3 → 2/3 · 微信广告 demo 待 Phase 1 MVP 启动后落地(剩余 1/3) · 决策依据:9-18 巡检报告**优先 #1 建议**
- **v0.1++ · 2026-09-19 03:20 二次扩展**: T5 应急第 15 次命中 · 在 v0.1+ 基础上新增 §4 微信广告 demo 流程设计(朋友圈广告素材库 + 抓取范围 + 5 步流程 + 最小代码框架 + 风险与合规边界 + 与小红书 + 巨量引擎差异对照 6 维 + Phase 1 MVP 接入路径) · 进度 2/3 → **3/3 闭环** · **Phase 0 #3 数据源 demo v0.1++ 文档级 100% 闭环里程碑**(3 大数据源全文档级) · §5 → §6 / §6 → §7 / §7 → §8 章节顺延更新 + §6.1/§6.2/§6.3/§6.4 模块关联追加微信广告条目 + 文档元数据 v0.1+ → v0.1++ · 决策依据:9-19 巡检报告**优先 #1 建议**

---

## 关联文档

- 上游索引: [[00-总索引]] · 06 Schema: [[06-策略输出Schema设计]] · 07 LLM 评估: [[07-LLM基线质量评估-v0.1]]
- **跨产品 hook**: 18-媒体 顾问(选题/改写/**创意(v0.1++)** 能力可调用,见 §6.3)
- **项目主计划**: [[../项目开发计划]] §5 Phase 0 #3(数据源 demo · 9-17 v0.1 + 9-18 v0.1+ + **9-19 v0.1++ 微信广告 demo 3/3 闭环**)

## 文档元数据

- **版本**:v0.1++ (2026-09-19)
- **状态**:Phase 0 #3 数据源 demo **3/3 闭环**(小红书 ✓ + 巨量引擎 ✓ + 微信广告 ✓ · 文档级 100%)
- **下次更新**:Phase 1 MVP 启动后落地 v0.2 实际 Python 代码(三大数据源同步)
- **维护人**:21-营销-Marketing 顾问(每日 03:20 T5 应急 / 02:20 巡检)