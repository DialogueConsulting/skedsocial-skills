---
name: create-post
description: >
  Create and schedule a post in Sked Social: resolve the accounts, find the media in the Library,
  create the draft, assign a publish date and time, then optionally customise captions, first
  comments, media, pillars, times and approval statuses per platform from the user's source
  material (a spreadsheet, Notion page, or other doc). Use this skill whenever someone wants to
  schedule a post in Sked. Trigger on: "schedule a post", "schedule this for Tuesday at 9am",
  "post this to Instagram on [date]", "queue this up for [brand]", "create a post for next week",
  "schedule this image with the caption...", "schedule these from my spreadsheet", "use a
  different caption for each platform", "set up this campaign from my Notion page", or any
  request to create a post in Sked with a publish date or time. For a draft with no date, use
  create-draft instead.
---

# Create Post

Your job is to take a post from "here's the media and caption" to "scheduled in Sked", and,
when the user supplies source material with platform-specific content, to tailor each
platform's post after scheduling.

This skill builds on `create-draft`. It loads that skill for target, media, pillar and draft
creation, then adds scheduling, per-platform customisation and approval statuses.

Order of work: **create the draft → validate all source mappings and times → schedule at the earliest resolved time → customise each platform's post → set statuses → verify and report.**
Platform-specific post updates can only be made after scheduling, because a draft with several
accounts is one post, and scheduling is what splits it into one post per account. Resolve every
source mapping before scheduling so an ambiguous or invalid row never leaves a post scheduled
with unintended content or timing.

---

## Step 1: Gather the request

Collect everything `create-draft` needs (target, media, caption, optional pillar), plus:

| Input | Required | Notes |
|---|---|---|
| Date and time | Yes | In the user's local time |
| Timezone | Resolved in Step 3 | Use the one the user states, otherwise read it from Sked |
| Source for platform-specific content | No | A spreadsheet, Notion page, doc or pasted text with per-platform copy, media, times or statuses |
| Approval status | No | Only applied if the user asks, or the source data states one |

If there is no date or time at all, ask for it now, together with anything else missing, in a single message.
If the user gave no date and wants a draft only, use `create-draft` instead.

---

## Step 2: Create the draft

Invoke the `create-draft` skill and follow it through its commit step (Step 6). In this flow:

- Skip `create-draft` Steps 7 and 8. Statuses are applied after scheduling here, and this skill gives the final report.
- If `create-draft` needs to ask the user anything, fold the date and time into the same message so the user only confirms once.
- Carry forward: the draft ID, the resolved accounts (name and platform), the Brand Group ID, the caption and the media.

If the user supplied platform-specific source material, the caption passed to `create-draft` is
the **base caption**: the shared caption from the source, or the caption in the request. Per-platform
versions are applied in Step 6.

---

## Step 3: Resolve and validate scheduling inputs

Resolve the timezone, read the source if supplied, and validate all account-specific rows
before calling `schedule_draft`.

1. **Timezone.** Use the timezone the user states. Otherwise call `Sked_Social:get_post` with the new draft ID and read `userTimezone`.
2. **Read the source.** Read a spreadsheet, Notion page, Google Doc or Sheet, attachment, or pasted text with the available source tool. If it cannot be read, ask the user to paste or export the relevant content. Do not schedule until the source is readable or the user explicitly asks to proceed without its customisations.
3. **Map rows to accounts.** Match each source row to the resolved accounts by exact account name or platform. If a row has no match or could match more than one account, ask which account it applies to and wait. Do not schedule until every source row is resolved or the user explicitly excludes it.
4. **Resolve fields.** Use the mappings in Step 6. Resolve source media from the Library as in `create-draft` Step 3. If referenced media is not in the Library, do not schedule until it is uploaded or the user explicitly removes that media change. For each source pillar or campaign, call `Sked_Social:list_content_labels` for the account's Brand Group and resolve its name to a label ID. If a name has no exact match or multiple matches, ask the user to clarify before scheduling. If the source or user's request specifies approval statuses, call `Sked_Social:list_post_statuses` and resolve each status by name before scheduling; ask the user to clarify any unmatched status.
5. **Resolve times.** Use each account's source time when supplied; otherwise use the requested base time. Convert every local time using the timezone's correct daylight saving offset for that date. Each resolved instant must be in the future. When accounts have different times, the earliest resolved instant becomes the initial schedule time; the remaining account times are applied after scheduling in Step 6.
6. **Confirm inferred inputs.** Apply the confirmation rules below to all resolved dates, times, timezones, targets, and source mappings before scheduling.

**When to ask first.** Proceed without asking only if the date, time, timezone and targets are
all explicit and nothing was inferred. Ask the user to confirm if any of these apply:

- A date with no time, or a time with no date
- A relative expression with more than one reading ("Friday" said on a Friday, "next week", "this weekend")
- AM or PM is unclear
- No timezone can be established
- The time is in the past, or too close to now to schedule
- It is unclear whether all accounts should post at the same time
- Any account had a connection warning in `create-draft`
- A source row or field mapping needed interpretation

When you do ask, show the exact local date, time and timezone you intend to use, plus the accounts.

`schedule_draft` gives **every** account on the draft the same time. If the user wants
different times per account, schedule at the earliest resolved instant and set the remaining
times in Step 6 with `update_post`.

---

## Step 4: Schedule

Call `Sked_Social:schedule_draft` with:

```
draftId: <draft ID from Step 2>
scheduledAt: <UTC instant from Step 3>
idempotencyKey: <stable key>
```

Build the key from the draft ID and the instant, for example `schedule-<draftId>-20261027T2200Z`.

Handle the result:

- **`SCHEDULE_UNCONFIRMED`**: this may mean it worked. Retry with the identical draft ID, time and key. Never use a new key.
- **`DRAFT_NOT_SCHEDULABLE`**: the draft is already scheduled. Use `update_post` for changes (Step 6).
- **`IDEMPOTENCY_CONFLICT`**: the key was used with different input. Only create a new key if the user really wants the different time.
- **`PLATFORM_VALIDATION_ERROR`**: explain each account diagnostic in plain language and offer a fix (change the media, drop the account). The draft still exists.

Existing approval and publish gates still apply to the scheduled posts.

---

## Step 5: Find the per-account posts

Scheduling creates one post per account. Get each post's ID from the `schedule_draft` result.
If the result does not list them, find them with `Sked_Social:review_calendar` for the
Brand Group around the initial schedule time and identify each post using the resolved account ID,
exact initial scheduled instant, and other available post details. Use a result only when the lookup
identifies exactly one post for that account. Account and caption alone are not sufficient to
identify a post. If the lookup is missing or ambiguous, do not update that account's post; ask
the user to resolve it instead.

Call `Sked_Social:get_post` on each one to confirm it is scheduled and to read its current
account, caption, media, labels, status and time. Confirm the account and initial scheduled
instant match the intended post before any update. Hold this, because Step 6 compares against it.

If there is no platform-specific source material and no status was requested, skip to Step 8.

---

## Step 6: Customise each platform's post (optional)

Do this step when the user provides source material with per-platform content, or asks for
different captions, creative or timing per platform.

### 6a. Map the validated source to the posts

Use the source and account mappings resolved before scheduling in Step 3. Recognise these fields:

| Field in the source | Applied with |
|---|---|
| Caption or copy | `update_post` `caption` |
| First comment | `update_post` `firstComment` |
| Media, creative or file names | `update_post` `mediaAssetIds` |
| Pillar or campaign | Resolve the name to a label ID in Step 3; apply with `update_post` `labelIds` |
| Date and time | `update_post` `scheduledAt` |
| Approval status | Step 7 |

Rules:

- Platforms with nothing different from the base post are left alone.
- If a source row could fit more than one account, resolve it with the user before scheduling. Don't guess.
- Use caption and first-comment text exactly as written in the source.
- A platform-specific time must be a future instant. Convert it from local time as in Step 3.

### 6b. Apply

Show a short plan, then apply it:

```
Platform changes for [Brand]:

Instagram — caption, media (2 items)
LinkedIn  — caption, first comment, time (Wed 28 Oct, 9:00am)
Facebook  — no changes
```

Any mapping that needed interpretation must already have been confirmed in Step 3. Otherwise
apply and report without asking again.

For each post, call `Sked_Social:update_post` with the `postId` and only the fields that differ
from what `get_post` returned in Step 5. Do the posts one at a time. Notes:

- For a source pillar or campaign, merge its resolved label ID(s) with the post's existing
  `labelIds`, removing duplicates. `labelIds` replaces the whole set, so never omit existing
  labels when sending this field.
- If media is hosted outside the Library, use the "If the media isn't in the Library" steps in
  `create-draft`. Do not update that post's media until it is available; tell the user which
  posts are waiting.
- Media is checked against the platform rules of every account on the post. If the update is rejected, explain which rule failed and leave that post unchanged.
- `firstComment` is only supported on some platforms and post types. If it is rejected, say so and carry on with the other fields and posts.
- Labels must belong to the account's Brand Group.
- If the response includes `accountWarnings`, surface them.

---

## Step 7: Set approval statuses (only when requested)

Do this step only if the user asked for a status ("send these for review") or the source data
states one, either for all posts or per platform. Otherwise leave the posts in their current status.

1. Use the status IDs resolved in Step 3.
2. Call `Sked_Social:update_post` with the `postId` and `status` set to the matched status ID, on each post that needs it.

Statuses stay subject to the user's permissions and the workspace's approval rules. If an
update is refused, say which post and why.

---

## Step 8: Verify and report

Call `Sked_Social:get_post` on each post to confirm the final state, then report. Show times in
the user's local timezone, never UTC.

```
Scheduled for [Brand] — [timezone label]

| Platform | Date | Time | Status | Pillar | |
|---|---|---|---|---|---|
| Instagram | Tue 27 Oct | 9:00am | [status name] | [pillar or —] | [View](dashboardUrl) |
| LinkedIn | Wed 28 Oct | 9:00am | [status name] | [pillar or —] | [View](dashboardUrl) |
```

Use the post URL from the tool results rather than building one. Show the status name exactly as
returned. After the table, add one or two sentences for anything the user needs to act on:
posts that are waiting on media to be uploaded, platform changes that were rejected, accounts
that were dropped, or posts that still need approval before they can publish.

---

## When invoked by a batch skill

When `import-posts-from-csv` calls this skill, the whole batch has already been confirmed.

- The date, local time and timezone arrive resolved and confirmed. Skip the "ask first" checks in Step 3 and convert them to UTC as normal.
- Per-row values (first comment, approval status, pillar) arrive from the caller. Apply them in Steps 6 and 7 as if they came from a source document.
- Pass the caller's idempotency key to `create-draft`. The schedule key in Step 4 is built from the draft ID, so a re-run returns the original result.
- If a step can't be completed (a rejected schedule, a failed update), stop, return the reason to the caller, and let it carry on with the next row. Don't question the user about a single row.
---

## Guardrails

- Never schedule without a date and time the user has stated or confirmed.
- Never schedule in the past or guess a timezone. Convert local time to UTC with the correct daylight saving offset.
- Resolve and validate all supplied source mappings, account matches, media, labels and per-account times before scheduling. If clarification is needed, leave the draft unscheduled.
- When per-account times differ, initially schedule at the earliest resolved instant.
- Never change caption or first-comment text from the source or the request. Use it exactly as written.
- Never use a new idempotency key to retry an uncertain schedule result. Retry with the identical inputs and key.
- Read each post with `get_post` before updating it, and send only the fields that changed.
- Never infer a post ID from account and caption alone. Use IDs returned by `schedule_draft` or a lookup that identifies exactly one post, and verify its account and scheduled time before updating.
- Resolve source label names to IDs before scheduling; when updating labels, preserve existing labels and send the full deduplicated set.
- Never claim a post will publish if its approval status may block it. Report the status name as returned.
- Never surface internal IDs, status IDs, tokens, or raw JSON to the user.
- Never guess a Library collection, and never fabricate asset, account or label IDs.
- The Sked MCP cannot upload media or change a draft's date. If media isn't in the Library, guide the user to upload it, as described in `create-draft`.
