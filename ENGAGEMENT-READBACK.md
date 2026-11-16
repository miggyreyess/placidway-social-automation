# Engagement Read-back (added after submission, Nov 17)

The main project (see `README.md`) is the version submitted on Nov 12 and is unchanged. This document covers the first stretch goal, which I built afterward: reading each Facebook post's engagement back into the run log.

**File:** `placidway-engagement-readback.json`, a separate n8n workflow called **PlacidWay Engagement Readback**.

---

## How it works

```
TriggerManual ─┐
               ├→ GetPublishedPosts → FetchPostEngagement → ExtractCounts → UpdateLogEngagement
TriggerWeekly ─┘
```

| Node | What it does |
|---|---|
| `TriggerWeekly` | Runs every Monday at 8 AM (workflow timezone) |
| `GetPublishedPosts` | Reads log rows where `status = published` and `fb_post_id` is not empty |
| `FetchPostEngagement` | Facebook Graph API `GET /{fb_post_id}` with `fields = reactions.summary(total_count), comments.summary(total_count), shares`. Retries 3 times, 5 s apart. |
| `ExtractCounts` | Turns the response into `reaction_count`, `comment_count`, `share_count` and `engagement_checked_at`, and keeps the log row's `id` |
| `UpdateLogEngagement` | Updates the matching log row by `id` (update, not insert, so no duplicate rows) |

**New log columns:** `reaction_count`, `comment_count`, `share_count` (Number) and `engagement_checked_at` (Date) in `placidway_post_log`.

---

## Design decisions

- **Separate workflow.** Posting and analytics run on different schedules and fail independently. The main workflow can pause for up to 24 hours waiting for approval, which would otherwise delay engagement checks. Both workflows write to the same log table, so all data stays in one place.
- **Weekly, not daily.** Facebook engagement mostly settles within a few days, so a weekly check keeps the numbers useful with 7× fewer API calls.
- **More retries than publishing.** Reading is safe to repeat, so it gets 3 tries. Publishing gets 2 because a retry could create a duplicate post.
- **Missing fields default to 0.** Facebook leaves out `shares` entirely when a post has no shares. `ExtractCounts` uses `$json.shares?.count ?? 0`, so every row gets a clean number instead of an error or an empty value.
- **Same error handling.** The workflow's Error Workflow is set to **PlacidWay Error Alerts**.

---

## Extra Facebook permission

Reading comment counts needs **`pages_read_user_content`**, since comments are user content. The first run failed with error `#10` until I added it.

Setup:
1. Add `pages_read_user_content` to the app's Pages use case (App Dashboard → Use cases → Customize).
2. Regenerate the long-lived Page token with all four scopes: `pages_show_list`, `pages_manage_posts`, `pages_read_engagement` and `pages_read_user_content`. Keep the original three, or publishing breaks.
3. Update the `Facebook Test Page` credential. Both workflows share it, so updating it once covers both.

---

## Limitations and next steps

- Every published post is re-checked on every run. As the log grows, limit this to posts from the last 30 days, since most engagement happens early.
- Reach and impressions aren't collected. They need Page Insights (`read_insights`).
- A weekly summary email (for example, the top post of the week) could be added after `UpdateLogEngagement`.
