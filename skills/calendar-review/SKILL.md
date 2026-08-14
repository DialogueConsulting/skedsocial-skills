---
name: calendar-review
description: >
  Review the upcoming content calendar for a Sked Social Brand Group. Surfaces what's
  scheduled, what's missing, what needs attention, and how content is distributed across
  pillars, platforms, and post types. Use this skill whenever a user wants to check their
  calendar, understand what's coming up, or know what needs action before the week starts.
  Trigger on: "what does my calendar look like?", "am I ready for next week?", "check my
  calendar", "what's scheduled?", "what needs my attention this week?", "are there any gaps
  in my content?", "what's coming up for [brand]?", or any request to review, audit, or
  understand an upcoming content schedule.
---

# Calendar Review

Your job is to pull the upcoming content calendar for a Sked Brand Group, surface what's
scheduled, flag what needs attention, and summarise how content is distributed. This is a
read-only skill — it never creates, edits, or moves content.

---

## Step 1: Resolve the Brand Group

Call `sked-client:list_brand_groups`.

- If only one group is returned — use it silently.
- If multiple groups are returned:
  - If the user named a brand — match it and proceed silently.
  - If no brand was specified — present the list by name and ask which brand to review. Wait for their selection.
- If the user asks for multiple brands — run subsequent steps for each group and combine output under separate brand headings.

Hold the resolved `groupId` (or list of groupIds) for all subsequent calls.

---

## Step 2: Load account and label context

Run both calls before proceeding — they inform display logic and labelling throughout.

**Accounts:** Call `sked-client:list_social_accounts` with the resolved `groupId`.

Build an account map: `{ accountId → { name, platform } }`.

Determine the multi-account rule: for each platform code (FB, IG, LI, TT, YT, PIN, etc.), count how many accounts share it.
- One account for that platform → display platform shortcode only (e.g. `FB`)
- Two or more accounts for that platform → display shortcode + account name (e.g. `FB — Sked Social Official`)

**Labels:** Call `sked-client:list_content_labels` with the resolved `groupId`.

Build a label map: `{ labelId → { name, type } }`. Used to resolve label names on calendar items and to identify content pillars for distribution analysis.

---

## Step 3: Fetch the calendar

Call `sked-client:review_calendar` with:
```
groupId: <resolved groupId>
preset: "fortnight"
```

This returns:
- `items` — scheduled posts and dated ideas in the UTC date window
- `availableDrafts` — undated drafts in the pool
- `summary` — pre-computed counts by platform, label, item type, day of week
- `userTimezone` — IANA timezone name (e.g. `Australia/Melbourne`)

Do not use the `gaps` array from the API response directly — see Step 4.

---

## Step 4: Convert all times to local timezone

All `date` and `time` values on items are in UTC. Convert every item to the user's local
timezone before any display or analysis.

**For each item:**
1. Combine `date` + `time` into a UTC datetime (e.g. `2026-08-09T23:00:00Z`)
2. Convert to `userTimezone`, accounting for DST
3. Store the resulting **local date** and **local time** — the local date may differ from the UTC date

**Timezone label for the response header:**
Derive the correct abbreviation for the dates in the review window (e.g. AEST for Australian winter, AEDT for Australian summer). Format as: `Melbourne time (AEST, UTC+10)`

**Recompute gaps from local dates:**
Do not use the `gaps` array from the API — it is computed from UTC dates and will be inaccurate after timezone conversion.

Instead:
1. Collect all local dates that have at least one item
2. Build the full local date range for week 1 (local day 1 through local day 7 of the window)
3. Any date in that range with no items = a gap

**Split items into two windows:**
- **Week 1** — local dates 1–7. Show all days, including gaps.
- **Week 2** — local dates 8–14. Show only days that have posts. No gap rows.

---

## Step 5: Build the calendar tables

Present two separate tables with identical column structure.

**Columns:** Date | Time | Type | Platforms | Pillar | Status | (link)

**Date:** Local date formatted as `Mon 10 Aug`. For multiple posts on the same local date, show the date on the first row only — leave the cell blank on continuation rows.

**Time:** Local time formatted as `9:00am`. If multiple posts on the same date share a time, show it on the first row only.

**Type:** Full word from the `postType` field: Image, Video, Carousel, Reel, Story, etc.

**Platforms:** Resolve `accountIds` using the account map from Step 2. Display platform shortcodes (FB, IG, LI, TT, YT, PIN). Apply the multi-account rule: if a platform has multiple accounts, display as `FB — Account Name`. If a platform cannot be resolved, display `—`.

**Pillar:** Resolve `labelIds` using the label map, filtering to `type: CONTENT_PILLAR`. Display the pillar name. If no content pillar is assigned, display `⚠ No pillar`.

**Status:** Display `statusName` exactly as returned by the API. Do not map, normalise, or interpret it.

**Link:** `[View](https://app.skedsocial.com/dashboard?postId=<item.id>)`

---

### Week 1 — full grid

Show every day in the 7-day local window as a row.

For **gap days** (no items): show the date and span the remaining columns with `No content scheduled`. Do not show time, type, platforms, pillar, status, or a link.

For **days with items**: one row per item. First item shows the date; continuation items leave the date cell blank.

---

### Week 2 — attention items only

Label this section: **Week ahead — items needing attention**

Show only days that have at least one post. No gap rows. Include all posts in the window — the heading already frames these as things to be aware of.

---

## Step 6: Distribution tables

Show three tables immediately after the calendar tables. Each table is split by week.

Aggregate from the local-date-converted items array. Do not use the API `summary` object — it reflects UTC totals for the full period and will not split correctly by local week.

**Format:**

| | Week 1 | Week 2 |
|---|---|---|
| Row label | N (X%) | N (X%) |

Percentage = count ÷ total items in that window × 100, rounded to nearest whole number.

If a window has zero items, show `—` in that column.

---

**Table 1 — Content pillar distribution**

One row per content pillar that has at least one post in either window. Add a final row for items with no pillar assigned. Omit pillars with zero posts in both windows.

---

**Table 2 — Platform distribution**

One row per platform shortcode that appears in any item. Add a final row for items with no platform resolved. Show 0 (0%) for a platform with no posts in a given window rather than omitting it.

---

**Table 3 — Post type distribution**

One row per post type (Image, Video, Carousel, Reel, Story, etc.). If all posts are the same type, still show the table — a single row confirms the format mix at a glance.

---

## Step 7: Drafts summary

Below the distribution tables, add a short prose paragraph for the drafts pool.

Pattern:
> You have **[N] drafts** in your pool[, covering [pillar names if any are tagged]]. [X] [is/are] marked **[statusName]** and can be reviewed now, independently of scheduling.[If no pillars: None have a content pillar assigned.]

Rules:
- If `availableDrafts.items` is empty — omit this section entirely.
- **Pillar breadth:** resolve label IDs on draft items using the label map. List pillar names if present. If none, note it.
- **Status:** surface the most action-relevant status name. If "Ready for review" and "Work in Progress" both appear, lead with the review-ready count.
- If `availableDrafts.hasMore` is true, append: *"Showing the [N] most recent — there are more in your pool."*

---

## Step 8: Deliver the response

Open with a single context line:

> Here's your calendar for **[Brand Group name]** — [date range], [timezone label].

Then deliver in order:
1. Week 1 table
2. Week 2 table (omit if no items exist in the window)
3. Distribution tables (content pillar → platform → post type)
4. Drafts summary (omit if no drafts)

Close with a **flags paragraph** — 1–3 sentences calling out the highest-signal items that need action. Pick from: posts awaiting review, posts still in progress heading into week 2, posts missing content pillars, drafts awaiting review. Do not repeat the full table contents. Do not end with a question or an offer to fix gaps — this skill is informational only.

Example:
> All 8 posts this week are awaiting client review. 6 posts across the fortnight have no content pillar and won't appear in your pillar analytics until they're tagged. 3 posts in week two are still Work in Progress and need to move before they hit the review queue.

---

## Guardrails

- Never create, edit, move, or delete any content. This skill is read-only.
- Never surface API IDs, status UUIDs, internal field names, or raw JSON to the user.
- Never fabricate post details, pillar names, or platform data.
- Always convert times to local timezone before display. Never show UTC times.
- Always recompute gaps from local-converted dates. Never use the API `gaps` array directly.
- If `list_brand_groups` returns no groups, tell the user they need to set up a Brand Group in Sked before continuing.
- If `review_calendar` returns no items and no drafts, tell the user their calendar is empty for the period and stop there.
- If platform resolution fails for an item (account ID not in the account map), display `—` and continue.
- If the user requests a specific account filter (e.g. "just my Instagram"), filter items to that account's ID before building the tables.
