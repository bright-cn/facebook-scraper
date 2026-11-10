# 供在此存储库中工作的编码智能体使用

更改任何内容前请先阅读本文。以下每一行都已于 2026-11-03 使用实时 API、`brightdata-sdk` 2.5.2 或 Bright Data CLI 0.4.1 验证。如果 API 参考页面与实时 API 不一致，此处以实时 API 为准。

## 身份验证

- `token()` 会先读取 `BRIGHTDATA_API_TOKEN`，然后读取存储在 `brightdata-cli/credentials.json` 中的密钥 `bdata login`（Python SDK 读取的是同一个文件）。设置该变量或运行一次登录，是必须由人工完成的唯一步骤；切勿将密钥粘贴到代码中。
- README 中的原始 HTTP 快速入门仅读取 `BRIGHTDATA_API_TOKEN`。它无法读取 CLI 登录信息。

## API 的工作方式

- 此软件包使用标准库调用 `https://api.brightdata.com/datasets/v3`，且没有任何依赖项。请保持这种方式。Python SDK 封装了 11 个 Facebook 抓取工具中的 5 个（按主页、群组和帖子 URL 获取帖子，以及评论和短视频），其 `comments()` 会发送 `num_of_comments`，API 会以 400 验证错误拒绝该参数。
- 11 个数据集均按 URL 采集；其中 3 个还支持发现：
  - 主页帖子 `gd_lkaxegm826bjpoo9m5`、群组帖子 `gd_lz11l67o2cb3r0lkj3`、按 URL 获取帖子 `gd_lyclm1571iy3mv57zw`、短视频 `gd_lyclm3ey2q6rww027t`：使用 `num_of_posts` 限制数量。
  - 评论 `gd_lkay758p1eanlolqw8`：使用 `limit_records` 限制数量。
  - 主页 `gd_mf124a0511bauquyow`、个人资料 `gd_mf0urb782734ik94dz`。
  - 公司评价 `gd_m0dtqpiu1mbcyc2g86`：需要一个 `/reviews/` URL，使用 `num_of_reviews` 限制数量。
  - Marketplace `gd_lvt9iwuh6fbcwmx1a`：可通过 `keyword`（必须提供 `keyword` 和 `city`）或 `url`（分类 URL）发现。
  - 活动 `gd_m14sd0to1jz48ppm51`：可通过 `url`（活动搜索 URL）或 `venue`（场地主页的 `/events` URL；仅使用主页 URL 会返回“There are no events.”）发现。
  - Meta 广告资料库 `gd_mtveskb62h7q65ftgz`：通过 `keyword` 发现，必须提供 `country` 和 `ad_type`。接受 `ad_type` `"All ads"`；拒绝 `"all"`。
  - `bdata pipelines info <dataset_id> <method>` 会输出实时输入模式。
- 每次调用都是异步任务：`/trigger`、轮询 `/progress/<id>`，然后调用 `/snapshot/<id>?format=json`。运行耗时从 12 秒（评价、单个广告）到 234 秒（Marketplace 分类）不等。`TIMEOUT` 为 900。
- 始终发送 `include_errors=true`。API 默认会丢弃错误行，这会让失败的输入看起来像空结果。
- `limit_per_input` 会限制发现数量及费用。达到限制时，仍在运行的爬虫会以错误为“Crawl aborted on job cancel”的行返回。`split()` 不会将这些行计为失败。
- 许多品牌主页已关闭评价：Coca-Cola 返回“Reviews not found”，Starbucks 和 Shake Shack 返回失效页面。`TheGarden/reviews/` 可用。
- 活动搜索的匹配较宽泛：“tech conference”返回了 3 个活动，但没有一个是会议。
- 如果状态轮询因网络错误失败，会持续重试直到截止时间。HTTP 错误仍会引发异常。
- 模式可能随时更改，恕不另行通知。切勿硬编码字段列表。
- 完整文档索引，每个条目对应一个 `.md` 页面：https://docs.brightdata.com/llms.txt

## 在此工作

- `pytest` 可离线运行且不需要令牌。`ruff check .` 必须通过。
- CI 会在一台空白机器上执行 README 自身的命令。每周工作流会针对真实 API 执行 README 中的每个围栏代码块，每日工作流则运行较小规模的实时检查。代码片段必须完整且可单独粘贴运行，展示的输出必须真实。
- 群组帖子、评论和评价会包含普通用户的姓名。README 代码片段仅输出数量、日期和主页名称，不输出谁写了什么。
- README 中的字段表位于 `<!-- fields:start -->` 和 `<!-- fields:end -->` 之间，由 `.github/scripts/field_tables.py` 每天根据 `scrapers-full.json` 重新生成。请勿手动编辑。
- README 顶部的“最后验证”徽章行会由每日运行重写。请勿手动编辑。
- 保持精简。除了状态轮询外，不要添加重试、去重、调度、数据库、异步示例或并发。
