# 适用于 Python 的 Facebook 抓取工具

使用 Python 将 Facebook 帖子、评论、短视频、Marketplace、活动和 Meta 广告资料库抓取为 JSON。无需登录 Facebook，无需 Graph API。基于 [Bright Data Facebook 爬虫工具 API](https://www.bright.cn/products/web-scraper/facebook?utm_source=github&utm_medium=referral&utm_content=facebook-scraper) 构建。新账户每月可获得 5,000 点免费额度。

[![实时检查](https://github.com/bright-cn/facebook-scraper/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/facebook-scraper/actions/workflows/live.yml) [![最后验证](https://img.shields.io/badge/last%20verified-9%20Nov%202026-brightgreen)](https://github.com/bright-cn/facebook-scraper/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速入门](#如何使用-python-抓取-facebook-帖子) · [无需登录](#无需登录或-graph-api-即可抓取-facebook) · [CLI](#批量获取-facebook-主页帖子的-cli) · [端点](#抓取-facebook-帖子评论marketplace活动和广告) · [数据](#facebook-帖子数据字段json) · [错误](#facebook-抓取工具错误) · [MCP 和 Claude Code](#facebook-mcp-服务器和-claude-code-技能) · [免费额度](#这个-facebook-抓取工具免费吗) · [文档](https://docs.brightdata.com/products/scrapers/facebook/introduction) · [支持](#支持)

使用 Python 获取 JSON 格式的 Facebook 数据：主页和群组帖子、评论、短视频、主页详情、个人资料、评价、Marketplace 商品、活动以及 Meta 广告资料库。涵盖全部 11 个 Bright Data Facebook 爬虫工具，每种功能对应一个函数。无需登录 Facebook、无需浏览器、没有依赖项。基于 [Bright Data Facebook 爬虫工具 API](https://www.bright.cn/products/web-scraper/facebook?utm_source=github&utm_medium=referral&utm_content=facebook-scraper) 构建。

此外还提供：一个用于获取主页帖子的单命令 CLI，以及完全不需要 Python 的 [Bright Data CLI](#facebook-mcp-服务器和-claude-code-技能)。

## 如何使用 Python 抓取 Facebook 帖子

需要 Python 3.10 或更高版本，无需安装任何内容：仅使用标准库。

```bash
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户每月可获得 [5,000 点免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```python
import json
import os
import time
import urllib.request

API = "https://api.brightdata.com/datasets/v3"
HEADERS = {
    "Authorization": f"Bearer {os.environ['BRIGHTDATA_API_TOKEN']}",
    "Content-Type": "application/json",
}


def call(path, body=None):
    data = None if body is None else json.dumps(body).encode()
    request = urllib.request.Request(API + path, data=data, headers=HEADERS)
    with urllib.request.urlopen(request) as response:
        return json.load(response)


page = {"url": "https://www.facebook.com/Meta", "num_of_posts": 3}
job = call("/trigger?dataset_id=gd_lkaxegm826bjpoo9m5&include_errors=true", [page])
while call(f"/progress/{job['snapshot_id']}")["status"] not in ("ready", "failed"):
    time.sleep(10)
for post in call(f"/snapshot/{job['snapshot_id']}?format=json"):
    if "error" in post:
        raise SystemExit(post["error"])
    print(post["likes"], post["num_comments"], post["url"])
```

```
723 342 https://www.facebook.com/Meta/posts/pfbid0evGHWPACd3JninWg3j3nucQjdFxoHWwJSPgFXKjZXaUiU8MctREDPRZZsAps6CGwl
4169 717 https://www.facebook.com/reel/2300984144015832/
7679 819 https://www.facebook.com/reel/29074877795478111/
```

预计耗时 1 至 3 分钟：API 会运行一个任务，循环则等待任务完成。费用[按帖子计费](https://www.bright.cn/pricing/web-scraper)。

该代码片段中有两项不是可选项。

`num_of_posts` 会限制运行数量及费用。如果不设置，抓取工具会返回其能访问到的主页全部帖子，并按全部帖子收费。

`include_errors=true` 让 API 将失效或私密主页报告为包含 `error` 键的行。API 默认关闭此选项，此时失败的主页与没有帖子的主页看起来相同。

Bright Data [Python SDK](https://github.com/bright-cn/bright-data-sdk-python) 封装了 11 个 Facebook 爬虫中的 5 个，而 API 会拒绝其评论调用发送的 `num_of_comments` 输入。因此，此处及本软件包均使用普通 HTTP。

## 无需登录或 Graph API 即可抓取 Facebook

Facebook 爬虫工具 API 不需要 Facebook 账户、Facebook 登录、Meta 应用或 Graph API 访问令牌。唯一凭据是设置为 `BRIGHTDATA_API_TOKEN` 的 Bright Data API 密钥。本软件包及其命令也能找到由 `bdata login` 存储的密钥。它会采集未登录访客可见的内容：私密群组或个人资料会以错误行返回。

## 批量获取 Facebook 主页帖子的 CLI

此存储库中的命令会同时针对多个主页运行 Python 快速入门，并写入一个 JSON 文件。

```bash
pip install git+https://github.com/bright-cn/facebook-scraper
facebook-scraper Meta Microsoft --limit 2
```

```
Fetching up to 2 recent posts per page, for: Meta, Microsoft
Usually one to three minutes each. Billed per post, 5,000 free credits a month.
asking  Meta...
got     Meta: 2 posts
asking  Microsoft...
got     Microsoft: 2 posts

Saved 4 posts as JSON to facebook.json (44 fields per post)
```

在终端中，`asking` 行会被原地更新的旋转指示器和运行计时器取代，让你可以看到任务正在运行以及已经运行了多长时间。

```
--limit N      posts per page, default 5, minimum 1
--out PATH     output file, default facebook.json
```

主页可以是名称、`@Meta`、`facebook.com/Meta` 或完整 URL。`python -m facebook_scraper` 也可用。该命令会从 `BRIGHTDATA_API_TOKEN` 中查找令牌；如果你运行过一次 `npx -p @brightdata/cli bdata login`，它也会从已存储的登录信息中查找，供你及该终端中的任何编码智能体使用。

也可以导入它而不是直接运行，以便按主页获取 `ok`、`note` 和 `error`，而不是原始行。`scrape` 不会因单个无效主页而引发异常；读取 `posts` 前请检查 `ok`：

```python
from facebook_scraper import scrape

for outcome in scrape(["Meta", "zzqqxx-no-such-page-404"], limit=1):
    if outcome.ok:
        print(f"{outcome.page}: {len(outcome.posts)} posts")
    else:
        print(f"{outcome.page} failed: {outcome.error}")
```

```
Meta: 1 posts
zzqqxx-no-such-page-404 failed: Posts not found
```

不存在的主页会以错误行“Posts not found”返回，并且耗时最长：该次运行用了 459 秒。如果一次运行返回部分帖子和部分错误行，帖子会保留，错误则存入 `note`。

## 抓取 Facebook 帖子、评论、Marketplace、活动和广告

Bright Data 提供 11 个 Facebook 抓取工具，每个都记录在 [Facebook 爬虫工具 API 文档](https://docs.brightdata.com/products/scrapers/facebook/introduction)中。本软件包针对每种调用方式各提供一个函数。以下每个代码片段都完整可用：将其原样粘贴到上面的 `pip install` 后即可。它们每周一都会在 Actions 中运行，每隔一天还会运行一次较小的检查，顶部徽章显示最新结果。

| 你拥有 | 你想要 | 方法 |
| --- | --- | --- |
| 主页 URL | 其近期帖子 | `page_posts(urls, 5)` |
| 公开群组 URL | 其近期帖子 | `group_posts(urls, 5)` |
| 帖子 URL | 这些帖子 | `posts(urls)` |
| 帖子 URL | 其评论 | `comments(urls, 10)`，按评论计费 |
| 主页 URL | 其近期短视频 | `reels(urls, 5)` |
| 主页 URL | 主页详情：关注者、类别、联系信息 | `pages(urls)` |
| 用户的个人资料 URL | 公开个人资料详情 | `profiles(urls)` |
| 某个主页的 `/reviews/` URL | 其评价 | `reviews(urls, 10)` |
| 关键词和城市 | Marketplace 商品 | `marketplace_search("bicycle", "New York", 5)` |
| Marketplace 分类 URL | 其中的商品 | `marketplace_category(url, 5)` |
| Marketplace 商品 URL | 这些商品 | `marketplace(urls)` |
| 场地主页的 `/events` URL | 其即将举行的活动 | `events_by_venue(url, 5)` |
| 活动搜索 URL | 匹配的活动 | `events_search(url, 5)` |
| 活动 URL | 这些活动 | `events(urls)` |
| 关键词和国家/地区 | Meta 广告资料库中的广告 | `ads_search("coffee", "US", 5)` |
| 广告资料库广告 URL | 这些广告 | `ads(urls)` |

每个函数都接受一个 URL 或 URL 列表。一个列表对应一个任务，而不是每个 URL 一个任务。每个函数都会返回行列表；API 无法采集的行会包含 `error` 键，而不是记录。接受限制数量的函数还会将 API 接受的任何其他输入作为关键字参数传递，名称与 API 中的名称相同：`page_posts(url, 10, start_date="10-01-2026")`。

### 在一个任务中抓取多个 Facebook 主页的帖子

```python
from facebook_scraper import page_posts

urls = ["https://www.facebook.com/Meta", "https://www.facebook.com/Microsoft"]
for post in page_posts(urls, 1):
    print(post["page_name"], post["date_posted"], post["likes"], post["url"])
```

```
Microsoft 2025-04-04T22:18:26.000Z 40838 https://www.facebook.com/Microsoft/posts/pfbid0347J66TDSqDpBtUS5Vmhu17t7bpCWTRzgi9yWw1387gGCJuwhAdKfsLzByvA93uRcl
Meta 2026-10-30T19:41:31.000Z 723 https://www.facebook.com/Meta/posts/pfbid0evGHWPACd3JninWg3j3nucQjdFxoHWwJSPgFXKjZXaUiU8MctREDPRZZsAps6CGwl
```

### 抓取 Facebook 群组帖子

仅限公开群组。记录会包含发帖人的姓名；此示例仅输出群组和数量。

```python
from facebook_scraper import group_posts

for post in group_posts("https://www.facebook.com/groups/532855527252024", 2):
    print(post["group_name"][:40], post["group_members"], post["date_posted"], post["num_comments"])
```

```
Jobs, Work and Vacancies in Cape Town an 238700 2026-11-03T18:53:25.000Z 0
Jobs, Work and Vacancies in Cape Town an 238700 2026-11-03T18:48:29.000Z 0
```

### 按 URL 抓取 Facebook 帖子

```python
from facebook_scraper import posts

url = "https://www.facebook.com/Meta/posts/pfbid0evGHWPACd3JninWg3j3nucQjdFxoHWwJSPgFXKjZXaUiU8MctREDPRZZsAps6CGwl"
post = posts(url)[0]
print(post["user_username_raw"], post["likes"], post["num_comments"], post["num_shares"])
print(post["content"][:70])
```

```
Meta 593 342 194
Meta Ray-Ban Display has been out for a while, so if you’re a develope
```

### 抓取 Facebook 帖子的评论

按评论计费，因此请先检查帖子上的 `num_comments`。限制值会作为 `limit_records` 传递给 API。

```python
from facebook_scraper import comments

url = "https://www.facebook.com/Meta/posts/pfbid0evGHWPACd3JninWg3j3nucQjdFxoHWwJSPgFXKjZXaUiU8MctREDPRZZsAps6CGwl"
rows = comments(url, 3)
print(len(rows), "comments; first:", rows[0]["date_created"], rows[0]["num_likes"], "likes")
```

```
3 comments; first: 2026-10-31T01:48:29.000Z 10 likes
```

### 抓取 Facebook 主页的短视频

```python
from facebook_scraper import reels

for reel in reels("https://www.facebook.com/Meta", 2):
    print(reel["date_posted"], reel["video_view_count"], "views", reel["url"])
```

```
2026-10-27T16:44:32.000Z 447000 views https://www.facebook.com/reel/2300984144015832
2026-10-22T16:25:44.000Z 1200000 views https://www.facebook.com/reel/29074877795478111
```

### 抓取 Facebook 主页数据

```python
from facebook_scraper import pages

page = pages("https://www.facebook.com/Meta")[0]
print(page["page_name"], page["followers"], "followers,", page["primary_category"])
```

```
Meta 106000000 followers, Company
```

### 抓取 Facebook 个人资料

仅抓取未登录访客可见的公开个人资料详情。

```python
from facebook_scraper import profiles

profile = profiles("https://www.facebook.com/zuck")[0]
print(profile["name"], "|", profile["city"])
```

```
Mark Zuckerberg | Palo Alto
```

### 抓取 Facebook 商家主页的评价

传入主页的 `/reviews/` URL。许多品牌主页已关闭评价，会返回错误行“Reviews not found”或失效页面，而不是零行。此示例输出评价结论，而不是评价者。

```python
from facebook_scraper import reviews

for review in reviews("https://www.facebook.com/TheGarden/reviews/", 3):
    print(review["company_name"], review["review_time"], "recommends:", review["recommends"])
```

```
Madison Square Garden 2026-10-31T14:43:54.000Z recommends: True
Madison Square Garden 2026-10-21T12:28:03.000Z recommends: True
Madison Square Garden 2026-10-19T22:29:17.000Z recommends: False
```

### 抓取 Facebook Marketplace 商品

```python
from facebook_scraper import marketplace_search

for item in marketplace_search("bicycle", "New York", 3):
    if "error" not in item:  # rows the limit cancelled carry only an error
        print(item["final_price"], item["currency"], item["title"][:40], "|", item["location"])
```

```
100 USD Bikes for sale like new | Perth Amboy, NJ
100 USD bicycle | Linden, NJ
350 USD 700c Whls 24 Spd Medium Frame Cannondale | Brooklyn, NY
```

`marketplace_category(url, 5)` 可从分类 URL（例如 `https://www.facebook.com/marketplace/category/vehicles`）执行相同操作，而 `marketplace(urls)` 会采集你已有 URL 的商品。

### 抓取场地中的 Facebook 活动

传入场地主页的 `/events` URL。仅使用主页 URL 找不到活动。

```python
from facebook_scraper import events_by_venue

for event in events_by_venue("https://www.facebook.com/TheGarden/events", 3):
    print(event["event_date"], event["title"])
```

```
2026-11-04T00:00:00.000Z New York Rangers vs. St. Louis Blues
2026-11-05T00:30:00.000Z New York Knicks vs. Chicago Bulls
2026-11-04T00:00:00.000Z NY Rangers vs St. Louis Blues Premium Seating
```

`events_search(url, 5)` 接受活动搜索 URL，例如 `https://www.facebook.com/events/search/?q=marathon`。其匹配较宽泛：请检查返回的标题。

### 抓取 Facebook 广告资料库

```python
from facebook_scraper import ads_search

for ad in ads_search("coffee", "US", 3):
    if "error" not in ad:  # rows the limit cancelled carry only an error
        print(ad["advertiser_name"], "| active:", ad["is_active"], "| since", ad["start_date"])
```

```
Wall Stadium | active: True | since 2026-04-22T07:00:00.000Z
Biggby Coffee of Wickliffe Ohio | active: True | since 2025-04-29T07:00:00.000Z
Dame Fine Coffee | active: True | since 2026-06-11T07:00:00.000Z
```

`ad_type` 默认为 `"All ads"`，这是 API 验证的值；`"all"` 会被拒绝。当发现任务达到限制时，仍在运行的爬虫会被取消，并各自以错误为“Crawl aborted on job cancel”的行返回。`split()` 会将这些行视为限制正常生效，而不是失败。

### 立即触发，稍后获取

对于规模大于几个主页的任务，不要让进程阻塞一小时。触发任务、保存快照 ID，并在准备好后获取。快照可供下载 30 天。

```python
import time

from facebook_scraper import PAGES, fetch, status, trigger

snapshot = trigger(PAGES, [{"url": "https://www.facebook.com/Microsoft"}])
print("snapshot:", snapshot)
while (state := status(snapshot)) not in ("ready", "failed"):
    time.sleep(10)
print("status:", state)
print("fetched:", fetch(snapshot)[0]["page_name"])
```

```
snapshot: sd_mw12itsz2eyr9m4tcu
status: ready
fetched: Microsoft
```

`trigger` 接受任意数据集、`discover_by` 和 `limit_per_input`，因此可以访问表中的每个抓取工具。每项输入均已于 2026-11-03 被实时 API 接受，其中多项与 API 参考页面不同：

| 抓取工具 | dataset_id | discover_by | 输入 |
| --- | --- | --- | --- |
| 主页帖子 | `gd_lkaxegm826bjpoo9m5` | 无 | `{"url", "num_of_posts"}` |
| 群组帖子 | `gd_lz11l67o2cb3r0lkj3` | 无 | `{"url", "num_of_posts"}` |
| 按 URL 获取帖子 | `gd_lyclm1571iy3mv57zw` | 无 | `{"url"}` |
| 评论 | `gd_lkay758p1eanlolqw8` | 无 | `{"url", "limit_records"}`。`num_of_comments` 会被拒绝 |
| 短视频 | `gd_lyclm3ey2q6rww027t` | 无 | `{"url", "num_of_posts"}` |
| 主页 | `gd_mf124a0511bauquyow` | 无 | `{"url"}` |
| 个人资料 | `gd_mf0urb782734ik94dz` | 无 | `{"url"}` |
| 公司评价 | `gd_m0dtqpiu1mbcyc2g86` | 无 | `{"url", "num_of_reviews"}`，一个 `/reviews/` URL |
| Marketplace 商品 | `gd_lvt9iwuh6fbcwmx1a` | 无 | `{"url"}` |
| Marketplace 搜索 | `gd_lvt9iwuh6fbcwmx1a` | `keyword` | 必须提供 `{"keyword", "city"}` |
| Marketplace 分类 | `gd_lvt9iwuh6fbcwmx1a` | `url` | `{"url"}` |
| 活动 | `gd_m14sd0to1jz48ppm51` | 无 | `{"url"}` |
| 活动搜索 | `gd_m14sd0to1jz48ppm51` | `url` | `{"url"}`，一个 `/events/search/?q=` URL |
| 场地活动 | `gd_m14sd0to1jz48ppm51` | `venue` | `{"url", "upcoming_events_only"}`，一个 `/events` URL |
| 广告 | `gd_mtveskb62h7q65ftgz` | 无 | `{"url"}` |
| 广告搜索 | `gd_mtveskb62h7q65ftgz` | `keyword` | `{"keyword", "country", "ad_type"}`, `ad_type` `"All ads"` |

`bdata pipelines info <dataset_id> <method>` 会输出 API 验证的输入字段。

## Facebook 帖子数据字段（JSON）

大多数人需要的主页帖子字段：

```
url  date_posted  content  likes  num_comments  num_shares  page_name  page_followers
```

代码不会硬编码任何字段列表。API 返回的所有内容都会写入各行及命令生成的文件中。

<!-- fields:start -->
每天根据已发布的模式 [scrapers-full.json](https://docs.brightdata.com/scrapers-full.json) 重新生成，因此不会过时。记录包含适用于它的字段以及 `timestamp` 和 `input`；示例文件中的第一条帖子有 44 个键。

<details>
<summary>全部 52 个主页帖子字段，<code>gd_lkaxegm826bjpoo9m5</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url | Facebook 帖子 URL |
| `post_id` | text | 每条 Facebook 帖子的唯一标识符 |
| `user_url` | url | 每条 Facebook 帖子的唯一标识符 |
| `user_username_raw` | text | 发布内容的用户的用户名 |
| `content` | text | 帖子的文本或媒体内容 |
| `date_posted` | date | 帖子发布的日期和时间 |
| `hashtags` | 数组 | 帖子中使用的标签 |
| `num_comments` | 数字 | 帖子的评论数 |
| `num_shares` | 数字 | 帖子的分享数 |
| `num_likes_type` | 对象 | 收到的点赞类型：数据点“count_reactions_type”包含收到的所有回应。 |
| `page_name` | text | 发布帖子的 Facebook 主页名称 |
| `profile_id` | text | 用户个人资料的唯一标识符 |
| `page_intro` | text | Facebook 主页的简介或说明 |
| `page_category` | text | Facebook 主页的类别 |
| `page_logo` | 图片 | Facebook 主页的徽标 |
| `page_external_website` | text | 与 Facebook 主页关联的外部网站 |
| `page_likes` | 数字 | Facebook 主页的点赞数 |
| `page_followers` | 数字 | Facebook 主页的关注者数量 |
| `page_is_verified` | 布尔值 | Facebook 主页的验证状态 |
| `original_post` | 对象 | 表示帖子是原创内容还是转发内容 |
| `attachments` | 数组 | 帖子中包含的媒体附件 |
| `other_posts_url` | 数组 | 同一用户其他帖子的网页地址 |
| `post_external_link` | url | 帖子中包含的外部链接 |
| `post_external_title` | text | 外部链接的标题 |
| `post_external_image` | url | 帖子外部链接的图片 |
| `page_url` | url | Facebook 主页的网页地址 |
| `header_image` | 图片 | Facebook 主页的页眉图片 |
| `avatar_image_url` | text | Facebook 主页头像图片的 URL |
| `profile_handle` | text | Facebook 主页的账号名或用户名 |
| `has_handshake` | text | 表示帖子是否包含握手 |
| `is_sponsored` | 布尔值 | 表示帖子是否为赞助内容 |
| `sponsor_name` | text | 如果帖子是赞助内容，则为赞助商名称 |
| `shortcode` | text | 帖子 URL 的短代码 |
| `video_view_count` | 数字 | 如果帖子包含视频，则为观看次数 |
| `likes` | 数字 | 帖子的点赞数 |
| `post_image` | 图片 | 帖子图片 URL（如果存在） |
| `post_type` | text | 帖子类型 |
| `following` | 数字 | 关注该账号的个人资料数量 |
| `link_description_text` | text | 链接标题下方显示的文本 |
| `count_reactions_type` | 数组 | 收到的回应类型 |
| `is_page` | 布尔值 | 帖子来自主页还是个人资料 |
| `page_phone` | text | 主页上列出的公开电话号码 |
| `page_email` | text | 主页上列出的公开电子邮件地址 |
| `page_creation_time` | date | 主页创建日期 |
| `page_reviews_score` | text | 主页平均评价分数 |
| `page_reviewers_amount` | 数字 | 主页评价者总数 |
| `page_price_range` | text | 主页上显示的价格区间标签 |
| `about` | 数组 | 从主页“简介”部分提取的键值对 |
| `active_ads_urls` | 数组 | 指向 Facebook 广告资料库中该主页有效广告的链接 |
| `delegate_page_id` | text | 显示为负责此主页的主页 ID |
| `privacy_and_legal_info` | text | 个人资料的隐私和法律信息 |
| `play_count` | 数字 | 表示视频开始播放（或被点击播放）的次数 |

</details>

<details>
<summary>全部 44 个群组帖子字段，<code>gd_lz11l67o2cb3r0lkj3</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url |  |
| `post_id` | text | 每条 Facebook 帖子的唯一标识符 |
| `user_url` | url | 每条 Facebook 帖子的唯一标识符 |
| `user_username_raw` | text | 发布内容的用户的用户名 |
| `content` | text | 帖子的文本或媒体内容 |
| `date_posted` | date | 帖子发布的日期和时间 |
| `hashtags` | 数组 | 帖子中使用的标签 |
| `num_comments` | 数字 | 帖子的评论数 |
| `num_shares` | 数字 | 帖子的分享数 |
| `num_likes_type` | 对象 | 收到的点赞类型（例如回应、点赞） |
| `group_name` | text | 发布帖子的 Facebook 群组名称 |
| `group_id` | text | 用户个人资料的唯一标识符 |
| `group_url` | url | Facebook 群组的网页地址 |
| `group_intro` | text | Facebook 群组的简介或说明 |
| `group_category` | text | Facebook 群组的类别 |
| `group_logo` | 图片 | Facebook 群组的徽标 |
| `group_external_website` | text | 与 Facebook 群组关联的外部网站 |
| `group_members` | 数字 | Facebook 群组的成员数量 |
| `group_created_at` | date | 群组创建日期 |
| `user_is_verified` | 布尔值 | 发布帖子的 Facebook 用户的验证状态 |
| `original_post` | 对象 | 表示帖子是原创内容还是转发内容 |
| `attachments` | 数组 |  |
| `other_posts_url` | 数组 | 同一用户其他帖子的网页地址 |
| `post_external_link` | url | 帖子中包含的外部链接 |
| `post_external_title` | text | 外部链接的标题 |
| `post_external_image` | url | 帖子外部链接的图片 |
| `header_image` | 图片 | Facebook 主页的页眉图片 |
| `avatar_image_url` | text | Facebook 主页头像图片的 URL |
| `profile_handle` | text | Facebook 主页的账号名或用户名 |
| `has_handshake` | text | 表示帖子是否包含握手 |
| `is_sponsored` | 布尔值 | 表示帖子是否为赞助内容 |
| `sponsor_name` | text | 如果帖子是赞助内容，则为赞助商名称 |
| `shortcode` | text | 帖子 URL 的短代码 |
| `video_view_count` | 数字 | 如果帖子包含视频，则为观看次数 |
| `likes` | 数字 | 帖子的点赞数 |
| `post_image` | 图片 | 帖子图片 URL（如果存在） |
| `post_type` | text | 帖子类型 |
| `following` | 数字 | 关注该账号的个人资料数量 |
| `price` | 数字 | 与帖子关联的价格 |
| `location` | text | 与帖子关联的位置 |
| `profile_id` | text |  |
| `publisher_image_url` | url |  |
| `num_reaction_type` | 数组 | 收到的点赞类型（例如回应、点赞） |
| `delegate_page_id` | text | delegate_page_id |

</details>

<details>
<summary>全部 38 个按 URL 获取的帖子字段，<code>gd_lyclm1571iy3mv57zw</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url | URL |
| `post_id` | text | 每条 Facebook 帖子的唯一标识符 |
| `user_url` | url | 每条 Facebook 帖子的唯一标识符 |
| `user_username_raw` | text | 发布内容的用户的用户名 |
| `content` | text | 帖子的文本或媒体内容 |
| `date_posted` | date | 帖子发布的日期和时间 |
| `hashtags` | 数组 | 帖子中使用的标签 |
| `num_comments` | 数字 | 帖子的评论数 |
| `num_shares` | 数字 | 帖子的分享数 |
| `num_likes_type` | 数组 | 收到的点赞类型（例如回应、点赞） |
| `profile_id` | text | 用户个人资料的唯一标识符 |
| `page_logo` | url | Facebook 主页的徽标 |
| `page_likes` | 数字 | Facebook 主页的点赞数 |
| `page_followers` | 数字 | Facebook 主页的关注者数量 |
| `page_is_verified` | 布尔值 | Facebook 主页的验证状态 |
| `original_post` | 对象 | 表示帖子是原创内容还是转发内容 |
| `attachments` | 数组 | 附件 |
| `post_external_link` | url | 帖子中包含的外部链接 |
| `post_external_title` | text | 外部链接的标题 |
| `page_url` | url | Facebook 主页的网页地址 |
| `header_image` | url | Facebook 主页的页眉图片 |
| `avatar_image_url` | text | Facebook 主页头像图片的 URL |
| `profile_handle` | text | Facebook 主页的账号名或用户名 |
| `has_handshake` | text | 表示帖子是否包含握手 |
| `is_sponsored` | 布尔值 | 表示帖子是否为赞助内容 |
| `sponsor_name` | text | 如果帖子是赞助内容，则为赞助商名称 |
| `shortcode` | text | 帖子 URL 的短代码 |
| `video_view_count` | 数字 | 如果帖子包含视频，则为观看次数 |
| `likes` | 数字 | 帖子的点赞数 |
| `post_image` | url | 帖子图片 URL（如果存在） |
| `post_type` | text | 帖子类型 |
| `user_handle` | text | 用户账号名 |
| `play_count` | 数字 | 播放次数 |
| `video_title` | text | 视频标题 |
| `marketplace_price` | price | Marketplace 商品的价格。此值适用于分享到群组中的 Marketplace 商品。 |
| `delegate_page_id` | text | 被委派为“负责此主页”的主页 ID |
| `marketplace_is_sold` | text | 表示商品是否已售出，仅适用于 Marketplace 帖子 |
| `comments` | 数组 | 帖子的评论 |

</details>

<details>
<summary>全部 22 个评论字段，<code>gd_lkay758p1eanlolqw8</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url | 包含评论的主页或帖子的 URL |
| `post_id` | text | 每条帖子的唯一标识符 |
| `post_url` | url | 特定帖子的 URL |
| `comment_id` | text | 每条评论的唯一标识符 |
| `user_name` | text | 发表评论的用户姓名 |
| `user_id` | text | 每个用户的唯一标识符 |
| `user_url` | url | 用户个人资料的 URL |
| `date_created` | date | 评论创建的日期和时间 |
| `comment_text` | text | 评论的实际文本内容 |
| `num_likes` | 数字 | 评论收到的点赞或回应数量 |
| `num_replies` | 数字 | 评论的直接回复数量 |
| `attached_files` | 数组 | 与评论关联的任何文件或附件 |
| `video_length` | 数字 | 视频内容的时长（如适用） |
| `source_type` | text | 来源类型（例如 Facebook、外部链接） |
| `subtype` | text | 帖子或评论的子类别或具体类型 |
| `type` | text | 帖子或评论的一般类别 |
| `commentator_profile` | url | 评论者个人资料 |
| `comment_link` | url | 评论 URL |
| `reply` | 布尔值 | 回复 |
| `parent_comment_id` | text | 父评论 ID |
| `commentator_profile_url` | url | 评论者个人资料图片的 URL |
| `attached_images` | 数组 | 与评论关联的任何图片 |

</details>

<details>
<summary>全部 27 个短视频字段，<code>gd_lyclm3ey2q6rww027t</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url | 短视频的 URL |
| `post_id` | text | 每条 Facebook 短视频的唯一标识符 |
| `user_url` | url | 每条 Facebook 短视频的唯一标识符 |
| `user_username_raw` | text | 发布内容的用户的用户名 |
| `content` | text | 短视频的文本或媒体内容 |
| `date_posted` | date | 短视频发布的日期和时间 |
| `hashtags` | 数组 | 短视频中使用的标签 |
| `num_comments` | 数字 | 帖子的评论数 |
| `num_shares` | 数字 | 帖子的分享数 |
| `video_view_count` | 数字 | 观看次数 |
| `likes` | 数字 | 短视频的点赞数 |
| `page_name` | text | 发布短视频的 Facebook 主页名称 |
| `profile_id` | text | 用户个人资料的唯一标识符 |
| `page_logo` | url | Facebook 主页徽标的 URL |
| `page_likes` | 数字 | Facebook 主页的点赞数 |
| `page_followers` | 数字 | Facebook 主页的关注者数量 |
| `page_is_verified` | 布尔值 | Facebook 主页的验证状态 |
| `thumbnail` | url | 短视频缩略图的 URL |
| `external_link` | url | 短视频中包含的外部链接 |
| `page_url` | url | Facebook 主页的网页地址 |
| `header_image` | url | Facebook 主页页眉图片的 URL |
| `avatar_image_url` | url | Facebook 主页头像图片的 URL |
| `profile_handle` | text | Facebook 主页的账号名或用户名 |
| `shortcode` | text | 帖子 URL 的短代码 |
| `length` | 数字 | 短视频时长 |
| `audio` | text | 短视频的音频 |
| `video_url` | url | 短视频的视频 URL |

</details>

<details>
<summary>全部 28 个主页字段，<code>gd_mf124a0511bauquyow</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | text | 内部记录 ID |
| `url` | url | Facebook 主页 URL |
| `page_name` | text | 主页显示名称 |
| `username` | text | 主页账号名（用户名） |
| `entity_type` | text | 主页或个人资料 |
| `summary_text` | text | 简介标语文本 |
| `primary_category` | text | 主要类别 |
| `work` | 数组 | 工作经历 |
| `education` | 数组 | 教育经历 |
| `current_city` | text | 当前城市 |
| `hometown` | text | 家乡 |
| `relationship` | 对象 | 感情状态信息 |
| `address` | 对象 | 商家/地点地址 |
| `phones` | 数组 | 电话号码 |
| `emails` | 数组 | 电子邮件地址 |
| `websites` | 数组 | 网站和社交链接 |
| `hours` | 对象 | 营业时间 |
| `rating` | 对象 | 评分摘要 |
| `contact_and_basic_info` | 对象 | 简介 > 联系方式和基本信息 |
| `page_transparency` | 对象 | 简介 > 主页透明度 |
| `details_about` | 对象 | 简介 > 详细信息 |
| `privacy_and_legal_info` | text | 隐私/条款/法律说明 |
| `followers` | 数字 | 关注者数量 |
| `logo` | url | 主页徽标 |
| `is_verified` | 布尔值 | 主页是否已验证 |
| `group_members` | 数字 | Facebook 群组的成员数量（如果该主页是群组主页） |
| `group_created_at` | date | 群组创建日期（如果该主页是群组主页） |
| `num_posts_last_month` | 数字 | 上个月的帖子数量（如果该主页是群组主页） |

</details>

<details>
<summary>全部 11 个个人资料字段，<code>gd_mf0urb782734ik94dz</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url | URL |
| `name` | text | 用户姓名 |
| `id` | text | Facebook 个人资料 ID |
| `profile_photo` | 图片 | 个人资料照片 |
| `cover_photo` | 图片 | 个人资料的封面照片 |
| `work` | 数组 | 个人资料中的工作地点 |
| `college` | 数组 | 个人资料中的大学数据 |
| `high_school` | 数组 | 个人资料中的高中数据 |
| `photos` | 数组 | 个人资料中的照片 |
| `location` | text | 个人资料所属的地点 |
| `city` | text | 个人资料所属的城市 |

</details>

<details>
<summary>全部 18 个评价字段，<code>gd_m0dtqpiu1mbcyc2g86</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `company_name` | text | 被评价公司的名称。此字段标识评价针对的商家，有助于按公司对反馈进行分类和分析。 |
| `company_id` | text | 每家被评价公司的唯一标识符。此字段有助于区分不同公司，并便于引用和汇总与同一公司相关的评价 |
| `company_url` | url | 被评价公司的 URL |
| `url` | url | Facebook 平台上特定评价的 URL。此链接允许用户直接查看原始评价 |
| `review_time` | date | 评价发布的日期和时间。这有助于了解反馈的时间线和新近程度 |
| `recommends` | 布尔值 | 表示评价者是否推荐该公司（true 表示推荐，false 表示不推荐） |
| `review_content` | text | 评价的文本内容，详细描述评价者对公司的体验和看法 |
| `review_attachments` | 数组 | 一个 URL 数组，指向评价中包含的图片或视频等附件。这些附件可提供与评价相关的更多背景或证据 |
| `number_of_reactions_to_review` | 数字 | 其他 Facebook 用户对该评价作出的回应总数（例如赞、爱心、哇）。这表示评价的互动程度和可见度 |
| `review_reaction_types` | 数组 | 一个数组，详细列出评价收到的回应类型（例如赞、爱心、哈哈、哇、难过、愤怒）。这有助于了解受众对评价的情绪反应。 |
| `reviewer_profile_url` | url | 评价者 Facebook 个人资料的 URL。此链接可用于访问评价者的个人资料以了解更多背景 |
| `reviewer_name` | text | reviewer_name |
| `reviewer_profile_picture_url` | url | reviewer_profile_picture_url |
| `recommended_perc` | text | 推荐评价所占百分比 |
| `likes` | 数字 | 主页的点赞数 |
| `followers` | 数字 | 主页的关注者数量 |
| `total_reviews` | 数字 | 评价总数 |
| `replies` | 数组 | 回复 |

</details>

<details>
<summary>全部 23 个 Marketplace 商品字段，<code>gd_lvt9iwuh6fbcwmx1a</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url | Facebook Marketplace 商品页面的直接 URL |
| `title` | text | 卖家发布的商品标题 |
| `initial_price` | 数字 | 商品打折前的原始标价 |
| `final_price` | 数字 | 商品的当前价格或折后价格 |
| `currency` | text | 标价所使用的货币 |
| `product_id` | text | Facebook Marketplace 为商品分配的唯一标识符。 |
| `breadcrumbs` | 数组 | 表示商品分类层级结构的导航面包屑项目列表 |
| `condition` | text | 卖家报告的商品状况 |
| `description` | text | 卖家撰写的商品完整文本说明 |
| `location` | text | 大致位置 |
| `country_code` | text | 两个字母的 ISO 3166-1 alpha-2 国家/地区代码 |
| `root_category` | text | 商品在 Facebook Marketplace 中所属的根类别 |
| `images` | 数组 | 指向卖家上传的商品图片的 URL 列表 |
| `seller_description` | text | 卖家 Facebook Marketplace 个人资料中的简短简介或说明 |
| `color` | text | 商品颜色（如适用且卖家已提供） |
| `brand` | text | 商品的品牌或制造商（如适用） |
| `videos` | 数组 | 指向卖家作为商品信息一部分上传的任何视频内容的 URL 列表 |
| `profile_id` | text | 发布该商品的卖家的唯一 Facebook 个人资料 ID |
| `listing_date` | date | 商品最初发布到 Facebook Marketplace 的日期和时间 |
| `car_miles` | 数字 | 卖家报告的车辆里程。仅与汽车/车辆商品相关 |
| `is_sold` | 布尔值 | 表示商品是否已售出 |
| `transmission` | text | 车辆详情中显示的变速箱类型。仅与汽车/车辆商品相关 |
| `vehicle_details` | 对象 | 从 Facebook Marketplace 汽车商品的“关于此车辆”部分提取的详细车辆特定属性。非车辆商品为 Null。 |

</details>

<details>
<summary>全部 30 个活动字段，<code>gd_m14sd0to1jz48ppm51</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `event_id` | text | 活动的唯一标识符 |
| `url` | url | 活动的直接 URL 链接 |
| `main_image` | url | 活动主图片的 URL |
| `event_date` | date | 活动日期 |
| `title` | text | 活动名称或标题 |
| `people_responded` | 数字 | 标记为“已回应”活动的人数 |
| `event_by` | 数组 | 用于指定活动组织者的对象数组，每个对象包含名称和 URL 字段 |
| `location` | 对象 | 包含地址和 URL 的嵌套对象 |
| `access_level` | 数组 | 表示活动的隐私或访问级别（例如公开、私密） |
| `description` | 对象 | 包含文本、链接和标签字段的对象，用于提供详细的活动说明 |
| `hashtags` | 数组 | 与说明分开，这是一个提供与活动关联的一般标签的数组 |
| `hosts` | 数组 | 用于指定活动主办方的对象数组，包括名称、URL 和验证状态 |
| `suggested_events` | 数组 | 列出其他推荐或类似活动的数组，每个活动包含名称、URL、日期、感兴趣人数和地点字段 |
| `tickets` | 对象 | 详细说明票务信息的对象，包括 URL、提供商、最低/最高价格和货币 |
| `duration` | 对象 | 定义活动持续时间的对象 |
| `main_image_downloadable` | url | 可下载的主图片 |
| `responded_object` | 对象 | 回应对象 |
| `event_start_time` | date | 活动开始时间 |
| `event_end_time` | date | 活动结束时间 |
| `event_end_date` | date | 如果活动持续超过一天，则为活动的结束日期 |
| `full_address` | text | 活动地址 |
| `longitude` | text | 经度 |
| `latitude` | text | 纬度 |
| `event_timezone` | text | 活动时区 |
| `unformatted_description_text` | text | 保留原始结构的说明文本 |
| `alternative_dates` | 数组 | 备选日期 |
| `alternative_date_time` | 数组 | 备选日期时间 |
| `redirected_event_url` | url | 重定向后的活动 URL |
| `main_video` | url | 主视频（如果有） |
| `is_cancelled` | 布尔值 | 表示活动是否已取消 |

</details>

<details>
<summary>全部 32 个广告资料库广告字段，<code>gd_mtveskb62h7q65ftgz</code></summary>

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | url | Meta 广告资料库广告详情页 URL |
| `library_id` | text | 唯一的 Meta 广告资料库 ID（保留为文本，因为超出安全整数范围） |
| `is_active` | 布尔值 | 表示广告当前是否正在投放 |
| `status` | text | 广告状态标签（投放中、未投放） |
| `start_date` | date | 广告开始投放的日期 |
| `end_date` | date | 广告停止投放的日期（投放期间为空） |
| `total_active_time` | text | 显示的总投放时长（例如 8 小时） |
| `ad_category` | text | 特殊广告类别（住房、就业、信贷、社会议题、选举或政治）；商业广告为空 |
| `has_multiple_versions` | 布尔值 | 表示广告是否有多个创意版本 |
| `versions_count` | 数字 | 创意版本数量（来自“第 1 个，共 N 个”指示信息） |
| `creative_reuse_count` | 数字 | 使用此创意和文本的广告数量 |
| `platforms` | 数组 | 广告投放的平台 |
| `platforms_count` | 数字 | 广告投放的平台数量 |
| `description` | text | 广告正文 |
| `title` | text | 广告标题 |
| `link_description` | text | 标题下方显示的链接说明 |
| `caption` | text | 标题上方显示的域名（例如 ADIDAS.EU） |
| `cta_text` | text | 显示的行动号召按钮文本（例如“立即购买”） |
| `cta_type` | text | 标准化的行动号召类型（例如 SHOP_NOW、LEARN_MORE、SIGN_UP） |
| `link_url` | url | 广告的目标 URL |
| `image_url` | url | 主创意图片 URL |
| `video_url` | url | 主创意视频 URL |
| `video_preview_image` | url | 视频缩略图 URL |
| `ad_versions` | 数组 | 广告的所有创意版本或轮播卡片 |
| `advertiser_name` | text | “关于广告主”部分中的广告主名称 |
| `advertiser_profile_picture` | url | 广告主个人资料图片 URL |
| `advertiser_description` | text | 广告主“更多信息”说明文本 |
| `advertiser_social_accounts` | 数组 | “关于广告主”部分列出的广告主社交账户 |
| `advertiser_and_payer` | 数组 | 广告主和付款方条目（仅限欧盟及相关地区） |
| `eu_total_reach` | 数字 | 广告在欧盟的总触达人数 |
| `transparency_by_location` | 数组 | 各地点的触达和定向详情（仅限欧盟及相关地区） |
| `is_eu_ad` | 布尔值 | 表示广告是否在欧盟投放并包含 DSA 透明度部分 |

</details>
<!-- fields:end -->

<details>
<summary>来自 <code>facebook-scraper Meta --limit 1</code> 的真实输出文件开头</summary>

```json
{
  "generated_at": "2026-11-03T19:25:27.615005+00:00",
  "pages": [
    {
      "page": "Meta",
      "posts": [
        {
          "url": "https://www.facebook.com/Meta/posts/pfbid0evGHWPACd3JninWg3j3nucQjdFxoHWwJSPgFXKjZXaUiU8MctREDPRZZsAps6CGwl",
          "post_id": "1181668677855674",
          "user_url": "https://www.facebook.com/Meta",
          "user_username_raw": "Meta",
          "content": "Meta Ray-Ban Display has been out for a while, so if you’re a developer, this one’s for you.\n\nbuild a game, a productivity tool, a science app — really, anything you love — and watch it run on an in-lens display, controlled by the Meta Neural Band.\n\nneed inspo? here’s what people have already cooked up: \n\nMeta Geology by @inna.sparrow.xr\nVistathlon by @stspanho\nSee DJ by @levinriegner\nSightline by @fasai.creates",
          "date_posted": "2026-10-30T19:41:31.000Z",
          "num_comments": 343,
          "num_shares": 194,
          "num_likes_type": {
            "type": "Like",
            "num": 593
          },
          "page_name": "Meta",
  ...
```

完整文件（包含一个具有所有字段的帖子）位于 [examples/sample_output.json](examples/sample_output.json)。

</details>

## Facebook 抓取工具错误

| 你看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 退出码 2，在发出任何请求之前。请设置令牌或运行 `bdata login`。 |
| `failed  page: Posts not found` | 退出码 1。主页不存在、是私密主页或没有公开帖子。请检查名称。 |
| `failed  page: timeout: snapshot ... not ready after 900s` | 退出码 1。请重新运行，或稍后使用 `fetch()` 获取快照。 |
| `failed  page: HTTP 401: ...` | 令牌已设置，但不正确。 |

任何失败都会以状态码 1 退出，因此可以安全地使用运行结果控制脚本。

在函数中，相同情况如下所示：

| 你看到的内容 | 含义 |
| --- | --- |
| `TokenMissing` | 没有 `BRIGHTDATA_API_TOKEN`，也没有 `bdata login`。 |
| `ApiError: HTTP 400: ...validation_error...` | API 不接受的输入字段。消息中会给出其名称；请与上表比较。 |
| 包含 `error` 键的行 | API 对一个输入的响应：“Posts not found”、失效页面、“Reviews not found”或“There are no events.”。其他行不受影响。 |
| `TimeoutError` | 任务超过了 900 秒等待时间。将 `timeout=` 传给 `run`，或使用 `trigger` 并稍后获取。 |

## Facebook MCP 服务器和 Claude Code 技能

无需 Python，也无需安装任何内容。粘贴这两行；第一行会打开一次浏览器，也可以通过 SSH 和 CI 使用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_lkaxegm826bjpoo9m5 collect_by_url --url "https://www.facebook.com/Meta" --num_of_posts 2 --pretty
```

2 条记录，最多 38 个字段，2026 年 11 月 3 日耗时 74 秒。CLI 会省略值为 null 的字段。

CLI 通过数据集 ID 运行抓取工具，包括所有 Facebook 抓取工具。`bdata pipelines search facebook` 会列出其 ID 和方法，`--limit-per-input N` 会限制发现数量，默认为 10。

`npx skills add brightdata/skills` 会向 Claude Code、Cursor 和 Codex 提供这些命令及文档，使其之后可以理解自然语言。完整指南：[面向编码智能体的 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

对于没有终端的托管助手，可以使用 [Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use)。它通过其 `scrapers` 组中的 `search_scrapers` 和 `run_scraper` 运行任何预构建抓取工具，包括 Facebook：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

智能体还可以自行创建账户，无需填写注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 连接的其他所有平台（从 LangChain 到 Zapier 和 n8n）：[集成](https://docs.brightdata.com/integrations/introduction)。

## 这个 Facebook 抓取工具免费吗？

新的 Bright Data 账户每月可获得 [5,000 点免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。每个 Facebook 爬虫工具均按记录计费：按帖子、评论、短视频、主页、个人资料、评价、商品、活动或广告计费。[定价](https://www.bright.cn/pricing/web-scraper)中列出了费率。

## 支持

此存储库中的错误：[创建 issue](https://github.com/bright-cn/facebook-scraper/issues)，[CONTRIBUTING.md](CONTRIBUTING.md) 说明了需要提供的信息。有关 API、你的账户或额度的任何问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

[Apache 2.0](LICENSE)。

## 适用于 Node.js 的 Facebook 抓取工具

同一款 Facebook 抓取工具也提供 Node.js 版本：[bright-cn/facebook-scraper-node](https://github.com/bright-cn/facebook-scraper-node)。

## 更多 Bright Data 开源抓取工具

其他以相同方式构建并提供实时验证示例的 Python 抓取工具：

- [适用于 Python 的 Instagram 抓取工具](https://github.com/bright-cn/instagram-scraper-python)
- [适用于 Python 的 TikTok 抓取工具](https://github.com/bright-cn/tiktok-scraper)
- [适用于 Python 的 LinkedIn 抓取工具](https://github.com/bright-cn/linkedin-scraper-python)
