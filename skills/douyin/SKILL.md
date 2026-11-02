---
name: douyin
description: Douyin (抖音, often called Chinese TikTok or TikTok China) data through the douyin MCP tools. Use when the user wants Douyin videos searched by keyword or hashtag, a video's details, engagement or MP4 link, the comments under a video, a creator's profile, followers and recent posts, or Douyin Shop (抖音商城) products with price, monthly sales and commission.
---

# Douyin (抖音)

Five Douyin tools plus Apify's run and dataset tools, served by the `douyin` MCP server. Each Douyin tool starts a paid run on the user's Apify account and returns run metadata. The rows arrive later in a dataset. Every request follows the **run flow** below.

| Tool (client prefixes vary) | Use for |
|---|---|
| `zen-studio--douyin-search-scraper` | Keyword or hashtag video search: caption, engagement, creator, hashtags, music, video links |
| `zen-studio--douyin-video-scraper` | Details for known video links, share links or IDs: engagement, creator, direct MP4 and audio links; optional file saving |
| `zen-studio--douyin-profile-scraper` | Creator profile (bio, followers, total likes, agency, region) and their recent posts |
| `zen-studio--douyin-comments-scraper` | Top-level comments under videos, with replies on paid plans |
| `zen-studio--douyin-product-search-scraper` | Douyin Shop products for a keyword: price, monthly sales, shop, commission |
| `get-actor-run` | Wait for and check a run |
| `get-dataset-items` | Read the rows |
| `fetch-actor-details` | Full input reference and README of any of the five |
| `get-key-value-store-record` | Read a record a run saved in its key-value store |
| `abort-actor-run` | Stop a run the user no longer needs |

If only Apify's generic `call-actor` is available, call it with `actor: "zen-studio/<name>"` (for example `zen-studio/douyin-search-scraper`) and the same input; the run flow is identical.

## Run flow

A tool call returns after at most 45 seconds, whether or not the run has finished. Many runs take longer, so treat the first response as a receipt, not the result.

1. **Start.** Call the Douyin tool with the recipe input and `"waitSecs": 45`. Keep the `runId` from the response. Done when you hold a `runId`.
2. **Poll.** Read `status`. While it is `READY` or `RUNNING`, call `get-actor-run` with `runId` and `waitSecs: 45`, and tell the user the run is still going between calls. Done when `status` is terminal: `SUCCEEDED`, `FAILED`, `ABORTED` or `TIMED-OUT`. A response that is already `SUCCEEDED` skips straight to step 3.
3. **Fetch.** Call `get-dataset-items` with `datasetId` from `storages.datasets.default.id` (the response's `nextStep` also names it). Pass `limit` (20 to 50) and a `fields` projection from **Reading results**, always including `free_preview_notice,limit_reached,message`; Douyin rows carry well over a hundred fields. Dotted fields come back flattened: `fields: "statistics.diggCount"` returns a key named `"statistics.diggCount"`, not a nested object. Page with `offset` until you have what the question needs. Done when every row the answer depends on is read.
4. **Answer.** Summarise for the user: engagement counts, video links (`url`), and anything filtered out. Translate Chinese text when the user writes in another language.

One run per request. A run that is still `RUNNING` keeps working and billing; starting a second run for the same question pays twice. Resume polling the same `runId` instead, including after a client timeout or a new turn.

Terminal states other than `SUCCEEDED`: `TIMED-OUT` and `ABORTED` can hold partial rows, so fetch them and say the result is partial. `FAILED`: report `statusMessage`. Change the input and retry only when the message says the input is invalid; when it says the service is temporarily unavailable, tell the user and do not retry automatically, because on the free plan every run uses up allowance. A `SUCCEEDED` run whose item count reads 0 can lag for a few seconds; fetch once before concluding it is empty. A row with `"limit_reached": true` is not data: it carries the free-plan message in `message` (see **Cost and limits**).

## Chaining the tools

- **Search, then go deeper.** Pass a search row's `url` (`https://www.douyin.com/video/<id>`) or its `id` to Video details or Comments.
- **Search, then the creator.** Pass a search row's `authorMeta.secUid` to the profile tool. Display names and the 抖音号 (`authorMeta.username`) are not accepted.
- **Comments, then the commenters.** Pass `user.secUid` from comment rows to the profile tool.
- **Creator, then their posts.** Profile rows are posts with `url`; pass them to Comments or Video details.
- **Short share links.** `v.douyin.com/...` links work in Video details and the profile tool, not in Comments. For Comments, run Video details first and use the row's `url`, or ask the user for the full link.
- **Products stand alone.** Product rows carry `shop` and `affiliate` data but no creator IDs; there is no tool that lists a product's promoting creators.

## Input recipes

Start small (10 to 50 results) unless the user asks for more; every returned video, post, comment, reply and product is billed.

**Video search** (`douyin-search-scraper`):

```json
{ "keywords": ["露营"], "maxResultsPerQuery": 30, "sort": "most_liked",
  "publishTime": "one_week", "duration": "unlimited" }
```

- `keywords`: a list, up to 50 on paid plans. Each keyword has its own `maxResultsPerQuery` budget, so three keywords at 30 can return up to 90 billed videos. From 13 November 2026 each keyword searched is billed as well. The default `maxResultsPerQuery` is 100; always set it.
- Chinese, English, brand names and hashtags all work. For Chinese topics, use Chinese keywords: they match how creators caption their videos.
- `sort`: `general` (relevance, default), `most_liked`, `latest`.
- `publishTime`: `unlimited` (default), `one_day`, `one_week`, `half_year`. There is no one-month window; use `half_year` and filter by `createDate`, or `one_week`.
- `duration`: `unlimited` (default), `under_1m`, `one_to_five`, `over_5m`.
- `maxResultsPerQuery` goes up to 2,000, or `0` for everything available within that ceiling. A typical keyword returns a few hundred unique videos. Ask before going above a few hundred.
- `shouldDownloadVideos` saves MP4 files (billed per file); `shouldDownloadCovers` and `shouldDownloadSlideshowImages` save covers and photo-post images. Enable them only when the user asks for files.

**Video details** (`douyin-video-scraper`):

```json
{ "videoUrls": ["https://www.douyin.com/video/7702714233150131482", "https://v.douyin.com/iAbc123/"] }
```

- Browser links, app share links (`v.douyin.com`), legacy `iesdouyin.com` share links or bare numeric video IDs. One billed result per video; duplicates across formats are counted once.
- Batch 20 to 50 videos in one run: the start fee is charged once per run.
- Each row has flat `videoUrl` (direct MP4), `audioUrl`, `coverUrl` and `durationSeconds`; they can be empty, and photo posts (`type` other than video) have no MP4. The MP4 link works for about an hour; `videoMeta.cdnUrlExpiresAt` (Unix seconds) says exactly when it stops.
- `shouldDownloadVideos`, `shouldDownloadCovers` and `shouldDownloadSlideshowImages` save the files to the run's key-value store. MP4 saves are billed per file.

**Creator profile** (`douyin-profile-scraper`):

```json
{ "profileUrls": ["MS4wLjABAAAA9CpHmhKxCNlmd3Hig_KcgRgBKplditZhV47xCpDWFEM"],
  "maxPostsPerProfile": 20, "excludePinnedPosts": true }
```

- Profile URLs (`douyin.com/user/<secUid>`), app share links, a bare `secUid` (starts with `MS4wLjABAAAA`) or a numeric user ID. Up to 25 creators per run. The 抖音号 printed on a profile is not accepted; ask for a share link instead.
- `maxPostsPerProfile`: default 100; always set it. `0` returns one row per creator with profile fields at the top level, the cheapest way to compare followers.
- With posts, the output is one row per post, with the creator in `authorMeta`. `recentDays` (for example `30`) or `oldestPostDate` (`YYYY-MM-DD`) keeps only recent posts; `excludePinnedPosts: true` skips old pinned posts.

**Comments** (`douyin-comments-scraper`):

```json
{ "awemeUrls": ["https://www.douyin.com/video/7702714233150131482"],
  "maxCommentsPerAweme": 50, "includeReplies": false }
```

- Full `douyin.com/video/<id>` links, legacy share links or numeric video IDs. Short `v.douyin.com` links are rejected.
- Always set `maxCommentsPerAweme` (default 100, up to 5,000). Every top-level comment and every reply is billed.
- `includeReplies: true` nests replies in `replies[]` and needs a paid Apify plan; `maxRepliesPerComment` (default 100) is then the main cost lever. `replyCount` is Douyin's displayed total and can exceed what was collected.
- `region` is the province or country Douyin shows, when available. `null` means unknown, not overseas.

**Product search** (`douyin-product-search-scraper`):

```json
{ "keywords": ["蓝牙耳机"], "maxResults": 30, "includeDetails": false }
```

- Chinese product keywords work best. About 500 products per keyword is the ceiling; `0` means all of them. Default `maxResults` is 100; always set it.
- `includeDetails: true` adds the detail image gallery, service guarantees and `sales.lifetimeSold`, at a separate per-product charge and with a longer run. Leave it off unless the user needs those fields.
- Rows come back in Douyin's search order (`searchPosition`). Sort by `sales.monthlySold` or `price.amount` yourself.

## Reading results

Project fields with `get-dataset-items` `fields` (dot paths allowed, returned flattened). Check `free_preview_notice`, `limit_reached` and `message` before reading results: a row with `free_preview_notice` belongs to a capped free preview (tell the user the result is a preview), and a row with `limit_reached: true` is a notice, not data. Never re-run with a different input to get around a cap. Good starting sets:

- Video search: `free_preview_notice,limit_reached,message,id,url,text,createDate,statistics.diggCount,statistics.commentCount,statistics.shareCount,statistics.collectCount,authorMeta.name,authorMeta.secUid,authorMeta.followersCount,videoMeta.duration,city,searchPosition`
- Video details: `free_preview_notice,limit_reached,message,id,url,text,createDate,statistics,authorMeta.name,authorMeta.secUid,hashtags,durationSeconds,videoUrl,coverUrl,videoMeta.cdnUrlExpiresAt`
- Creator profile (posts): `free_preview_notice,limit_reached,message,url,text,createDate,isPinned,statistics.diggCount,statistics.commentCount,statistics.shareCount,statistics.collectCount,authorMeta.name,authorMeta.followersCount,authorMeta.totalLikesReceived,authorMeta.mcnInfo`. Profile-only rows (`maxPostsPerProfile: 0`) carry `name`, `signature`, `followersCount`, `totalLikesReceived`, `awemeCount`, `verified`, `customVerifyText`, `mcnInfo`, `ipLocation` at the top level.
- Comments: `free_preview_notice,limit_reached,message,awemeId,text,createDate,likeCount,replyCount,region,isAuthorLiked,user.nickname,user.secUid,replies`
- Product search: `free_preview_notice,limit_reached,message,title,detailUrl,price.amount,sales.monthlySold,sales.goodReviewRatio,shop.shopName,shop.shopScore,affiliate.commissionRatePercent,affiliate.cooperatingCreators,category.third.name,searchPosition`

`statistics.diggCount` is likes and `statistics.collectCount` is saves (收藏). `statistics.playCount` is usually `0`, which means the count is not available; do not report it as zero views. `videoMeta.duration` is in milliseconds, `durationSeconds` in seconds. `createTime` is Unix seconds and `createDate` is `YYYY-MM-DD`. Prices are in yuan (`price.amount`). The video `url` is permanent; direct video links last about an hour and cover images two weeks or more. `fetch-actor-details` returns the full field reference when the user asks for something outside these sets.

Saved files are linked from `videoFile`, `coverFile` and `slideshowFiles` with `kvStoreId`, `kvStoreKey` and `kvUrl`. They are binary (MP4, JPEG), so give the user the `kvUrl` or tell them to open the run in Apify Console; opening a `kvUrl` needs the user's Apify token. `get-key-value-store-record` (`keyValueStoreId` from `kvStoreId`, `recordKey` from `kvStoreKey`) reads a record when you need its metadata or a small text record.

## Cost and limits

- Pricing is per result on each tool's Apify Store page (`https://apify.com/zen-studio/<name>/pricing`) and is billed to the user's Apify account. Quote the page rather than a remembered number. Saved MP4 files and product details have separate charges.
- On Apify's free plan each tool allows a small preview:
  - Video search: 2 runs in total, the first keyword only, 10 videos per run. Do not spend these on tests.
  - Video details: 10 videos in total, across any number of runs.
  - Creator profile: 5 runs, the first creator only, up to 20 posts.
  - Comments: 15 runs, one video per run, one page of about 20 top-level comments, no replies.
  - Product search: 5 runs, the first keyword only, up to 20 products, no product details.
- A capped run says so in its `statusMessage` ("Free tier preview: ..."). When the allowance is used up, the run returns one row with `limit_reached: true` and a message such as "You have used all 2 free runs. Upgrade to a paying plan to continue." At busy times a free run can return "The free preview is temporarily unavailable. Please try again shortly." or say the preview is at capacity and nothing was charged. Tell the user the result is partial or did not run, and that any paid Apify plan removes the caps.
- `maxResultsPerQuery`, `maxPostsPerProfile`, `maxCommentsPerAweme`, `maxRepliesPerComment` and `maxResults` are the cost controls. State the caps you chose in the answer.
