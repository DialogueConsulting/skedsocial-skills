---
name: brand-summary
description: >
  Surface how a Sked Brand Group is performing — reach, interactions, follower growth, content
  pillar performance, and platform breakdown — for the last 30 days compared to the prior period.
  Use this skill whenever someone wants to understand their social media performance, review
  analytics, or get a quick read on how their brand is tracking. Trigger on: "how's [brand]
  doing?", "brand summary", "how did we perform?", "give me the analytics", "performance
  summary", "how is [brand] tracking?", "show me the numbers", "what's our reach?",
  "how are our pillars performing?", "monthly summary", "what's the engagement been like?",
  "how did [brand] do last month?", "show me performance for [brand]", or any request to
  review, report on, or understand social media analytics for a brand group. Always use this
  skill when the user asks about brand performance — even if they don't say "analytics" or
  "summary" explicitly.
---

# Brand Summary

Your job is to pull analytics for a Sked Brand Group and surface how the brand is performing
— headline metrics with period-on-period comparison, content pillar performance, and platform
breakdown. This is a read-only skill.

---

## Step 1: Resolve the Brand Group

Call `sked-client:list_brand_groups`.

- If only one group is returned — use it silently.
- If multiple groups are returned:
  - If the user named a brand — match it and proceed silently.
  - If no brand was specified — present the list by name and ask which brand to review. Wait for their selection.
- If the user asks for multiple brands — run subsequent steps for each group and combine output under separate brand headings.

Hold the resolved `groupId` for all subsequent calls.

---

## Step 2: Resolve the period

**Default:** last 30 days, compared to the 30 days immediately before that.

Compute from today's date in UTC:
- `period.since` = today − 30 days at `00:00:00Z`
- `period.until` = today at `00:00:00Z`
- `comparisonPeriod.since` = today − 60 days at `00:00:00Z`
- `comparisonPeriod.until` = today − 30 days at `00:00:00Z`

**If the user specifies a period** (e.g. "last week", "July", "the last 3 months"):
- Interpret it and compute the equivalent ISO datetimes
- Set `comparisonPeriod` to the prior period of the same length (e.g. "last week" → the week before)

Hold the computed datetimes. You will display dates in the user's local timezone (resolved from the `userTimezone` field returned in Step 3).

---

## Step 3: Fetch analytics and label context

Run both calls before proceeding — they inform every section of the output.

**Analytics:** Call `sked-client:get_brand_group_analytics_summary` with:
```
groupId:          <resolved groupId>
period:           { since: <ISO>, until: <ISO> }
comparisonPeriod: { since: <ISO>, until: <ISO> }
timezoneLocale:   <user's timezone if known, otherwise "UTC">
```

**If the API returns "Account group has no supported Insights accounts":**
Tell the user clearly: this brand group's connected accounts don't have analytics available in Sked. This typically means accounts aren't connected with the right permissions. Stop — do not attempt to show partial data.

The response includes:
- `totals` — headline metrics for the current period
- `comparisonTotals` — same metrics for the prior period
- `changes` — per-metric `delta` (absolute) and `percentChange` (signed float)
- `platformBreakdown` — array of platforms with `totals` and `interactionSharePercent`
- `labelPerformance` — content pillars that had ≥1 post this period, each with `postCount`, `reach`, `impressions`, `interactions`, `averageReach`
- `accounts` — per-account detail including platform-specific `interactionsBreakdown`
- `userTimezone` — IANA timezone to use for all date display

**Labels:** Call `sked-client:list_content_labels` with `groupId`.
Filter to `status: "ACTIVE"` and `type: "CONTENT_PILLAR"`.
This gives the full active pillar list — including pillars with zero posts this period, which `labelPerformance` omits.

---

## Step 4: Prepare the data

### Platform shortcodes
Map `platformType` to shortcodes for display:

| API value | Display |
|---|---|
| INSTAGRAM | IG |
| FACEBOOK | FB |
| GOOGLEMYBUSINESS | GMB |
| TIKTOK | TT |
| YOUTUBE | YT |
| LINKEDIN | LI |
| PINTEREST | PI |

### Number formatting
Apply consistently throughout the response:
- Numbers ≥ 1,000,000: format as M with one decimal (e.g. 1,234,567 → 1.2M)
- Numbers ≥ 10,000: format as K with one decimal (e.g. 340,554 → 340.6K)
- Numbers < 10,000: show as-is with comma separators (e.g. 1,899)
- **Deltas:** always show sign — `+340.6K` or `−377.1K`. Use − (minus) not - (hyphen).
- **Percentages:** always show sign — `+12%`, `−53%`, `0%`. Round to nearest whole percent.
- **Engagement rate:** `interactions ÷ reach × 100`, expressed as `X.XX%`. Show delta in **percentage points (pp)** — not a percentage of a percentage.

### Engagement rate
Compute for both periods:
- Current: `totals.interactions ÷ totals.reach × 100`
- Prior: `comparisonTotals.interactions ÷ comparisonTotals.reach × 100`
- Delta: current minus prior, expressed as `+X.XXpp` or `−X.XXpp`

### Unlabelled posts
`unlabelledPosts = totals.postsPublished − sum(labelPerformance[].postCount)`

These are posts published in the period with no content pillar assigned.

### Pillars with zero posts
Active pillars (from the label list) that do not appear in `labelPerformance` had zero posts this period. Collect these names — surface them after the pillar table.

### GMB identification
Any entry in `accounts[]` with `platformType: "GOOGLEMYBUSINESS"` has structurally different metrics. Its `interactionsBreakdown` contains `directionRequests`, `callClicks`, `websiteClicks`, `businessBookings`, `businessConversations` — not likes or comments. Never aggregate GMB interactions with social platform interactions.

---

## Step 5: Build the output

### Opening line

> Here's your brand summary for **[Brand Group name]** — [period label], [timezone abbreviation].

Period label: `12 Jul – 11 Aug 2026`. Omit the year if it matches the current year.
Timezone abbreviation: derive correctly for the dates in the window (e.g. AEST vs AEDT for Melbourne).

---

### Headline metrics

| Metric | This period | vs prior period |
|---|---|---|
| Posts published | N | ±N (±X%) |
| Reach | N | ±N (±X%) |
| Impressions | N | ±N (±X%) |
| Interactions | N | ±N (±X%) |
| Engagement rate | X.XX% | ±X.XXpp |
| New followers | N | ±N (±X%) |

If `totals.postsPublished` is zero: note that no posts were published this period and show the prior-period numbers for reference only. Still show the full headline table.

---

### Content pillar performance

| Pillar | Posts | Reach | Impressions | Interactions | Avg reach |
|---|---|---|---|---|---|
| [Pillar name] | N | N | N | N | N |
| No pillar | N | — | — | — | — |

Rules:
- Include only pillars from `labelPerformance` (i.e. those with ≥1 post).
- Sort by reach descending.
- Add a **"No pillar"** row at the bottom if `unlabelledPosts > 0`. Show the count; use `—` for all metric columns (the data isn't tracked per-post for unlabelled content).
- After the table, if any active pillar had zero posts this period, add a note in plain text: *"[Pillar A] and [Pillar B] had no posts this period."*
- If `labelPerformance` is empty **and** `unlabelledPosts > 0`: show a single "No pillar" row and note that no pillar tracking is available for this period.
- If there are no active pillars at all: omit this section entirely.

---

### Platform breakdown

Split social platforms and GMB — their metrics are not comparable.

**Social platforms (IG, FB, TT, YT, LI, PI):**

| Platform | Posts | Reach | Impressions | Interactions | Share of interactions |
|---|---|---|---|---|---|
| IG | N | N | N | N | X% |
| FB | N | N | N | N | X% |

"Share of interactions" = `interactionSharePercent` from the API, rounded to nearest whole percent.
Sort by interactions descending.

**Google Business Profile (if present):**
Show as a brief prose line rather than a table — GMB metrics tell a different story:

> GMB: [reach] searches, [N] direction requests, [N] call clicks, [N] website clicks.

Source these from the GMB entry in `accounts[].interactionsBreakdown`. Omit any GMB metric that is zero.

---

## Step 6: Closing insight

Write 2–3 sentences interpreting the data in plain language. The goal is the "so what" — what does a social media manager actually need to hear from these numbers?

Draw on:
- The most significant change vs prior period (positive or negative)
- Which pillar or platform is driving — or dragging — performance
- Anything that warrants attention: sharp declines, unlabelled posts eroding pillar tracking, a platform with high reach but low interaction, a pillar not posting

Do not repeat table contents verbatim. Do not end with a question or an offer to take action — this skill is informational only.

**Example:**
> Reach and interactions fell sharply this period (−53% and −73% respectively), with 2 fewer posts published than the prior period — so volume is the likely driver. Instagram is doing nearly all the engagement work at 90% of interactions, while Facebook's 175.7K reach is generating almost no response (1% interaction share). Only one pillar — Behind the Scenes — has tracking data; the other 4 posts have no pillar assigned, which means pillar analytics are incomplete until those posts are tagged.

---

## Guardrails

- Never fabricate numbers, deltas, or trends.
- Always show the comparison period — raw numbers without context are not useful.
- Format all numbers consistently throughout: K/M for large numbers, signed percentages, pp for engagement rate deltas.
- Never surface internal IDs, field names, status UUIDs, or raw JSON.
- GMB metrics are not social metrics — never aggregate them into platform totals or interaction share.
- If `totals.postsPublished` is zero, do not skip the response — show what data exists and note the zero-post period.
- If `list_brand_groups` returns no groups, tell the user they need to set up a Brand Group in Sked first.
