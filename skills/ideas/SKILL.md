---
name: ideas
description: >
  Manage the Ideas backlog for a Sked Brand Group — view what's in the pipeline, write new ideas
  to Sked from your own notes or brainstorm, generate fresh ideas grounded in your Brand Kit, or
  import an idea directly from a URL. Use this skill whenever someone wants to see, add, create,
  or import ideas into Sked. Trigger on: "show me my ideas", "what's in my backlog", "add an
  idea", "save these ideas to Sked", "generate ideas", "brainstorm ideas for [brand]", "create
  some content ideas", "give me ideas for next week", "fill my content gaps", "import this URL
  as an idea", "turn this into an idea", "add this link to my ideas", or any request to view,
  create, or manage ideas in the Sked Idea Planner.
---

# Ideas

Manage the Sked Idea Planner for a Brand Group. Four modes — view backlog, write ideas, generate
ideas, or import from URL. Claude detects intent from the message and routes to the right mode.
All writes require a confirmation step before calling the Sked API, except Mode D where
`create_idea_from_url` saves immediately by design.

---

## Step 1: Resolve the Brand Group

Call `sked-client:list_brand_groups`.

- If only one group is returned — use it silently.
- If multiple groups are returned:
  - If the user named a brand — match it and proceed silently.
  - If no brand was specified — present the list by name and ask which brand to use.
- Hold the resolved `groupId` for all subsequent calls.

---

## Step 2: Detect mode

Read the user's message and route to the correct mode:

| Signal | Mode |
|---|---|
| "show me my ideas", "what's in my backlog", "list ideas" | **A — View backlog** |
| User pastes or describes ideas they've already written | **B — Write ideas** |
| "generate ideas", "brainstorm", "give me ideas", "fill my gaps" | **C — Generate ideas** |
| URL present in message, "turn this into an idea", "add this link" | **D — URL import** |

If the signal is ambiguous, default to **C — Generate ideas**.

---

## Step 3: Load shared context

Run these calls before entering the mode — they inform writing, generating, and tagging.

**Brand Kit:** Call `sked-client:get_brand_kit` with `groupId`.
- Extracts: brand description, tone of voice profile (name + profile ID), brand values, content pillars with their label IDs.
- If the Brand Kit is incomplete or not set up: note it silently and proceed — the skill still works with less context.
- **Default behaviour:** Brand Kit is always used as the creative brief. If the user says "ignore the brand kit" or provides their own brief, use their brief only. If they say "use this as well as the brand kit", layer it on top.

**Active pillars:** Call `sked-client:list_content_labels` with `groupId`.
- Filter to `status: "ACTIVE"` and `type: "CONTENT_PILLAR"`.
- Hold the list of `{ _id, name }` pairs — used for tagging and pillar assignment.

**Connected accounts (Mode C and B only):** Call `sked-client:list_social_accounts` with `groupId`.
- Extract the set of active platform types — used for platform suggestions when generating or enriching ideas.

---

## Mode A — View backlog

Show the active ideas for the brand group.

1. Call `sked-client:list_ideas_by_group` with `groupId` and `status: "active"`.
2. Output a table sorted by stage (`working` before `pad`), then by `createdAt` descending:

| Title | Stage | Pillar | Platforms | Topic |
|---|---|---|---|---|
| [title] | Working / Pad | [pillar name or —] | [platform shortcodes or —] | [topicDetails truncated to 60 chars or —] |

Stage display: `"working"` → **Working**, `"pad"` → **Pad**

Pillar: resolve `labels[]` against the active pillar list from Step 3. Show pillar name(s), or `—` if none.

Platform shortcodes: IG, FB, TT, YT, LI, PI, GMB — comma-separated, or `—` if none.

3. Add a one-line count below the table: *"[N] ideas — [X] Working, [Y] in the Pad."*

---

## Mode B — Write ideas to Sked

The user has already formed their ideas — from notes, a project, a brainstorm doc, or a previous
Claude session. Claude's job is to receive them, map them to the Sked schema, and write them in.

### Step B1: Parse the ideas

Extract each idea from whatever the user has provided (numbered list, freeform text, pasted doc).
For each idea, identify what's present:

| Field | Source |
|---|---|
| `title` | Required — extract or ask if missing |
| `topicDetails` | Any description, notes, or brief they included |
| `labels` (pillar) | Explicit pillar name → resolve to label ID; otherwise leave blank for enrichment |
| `platforms` | Explicit platform names → map to shortcodes; otherwise leave blank |
| `creativeBrief` | Any creative direction they included |
| `copyBrief` | Any copy direction they included |
| `stage` | Default `"pad"` unless the user says "working" |

### Step B2: Offer enrichment once

Before saving, make a single offer — not per-idea, not blocking:

> *"I can enrich these before saving — suggest content pillars, platform targets, creative angles,
> and design notes based on your Brand Kit. Want me to improve them, or save as-is?"*

If the user wants enrichment:
- Assign the best-fit content pillar to each idea (match title + topicDetails against pillar descriptions)
- Suggest platforms based on the idea type and the brand's connected account mix
- Write a one-line `topicDetails` if missing or thin
- Add `creativeBrief` with a visual direction or format suggestion where it adds value
- Add `copyBrief` with a caption angle or hook where it adds value

If the user says save as-is — skip enrichment entirely.

### Step B3: Preview and confirm

Show a compact preview of all ideas before writing:

```
Here's what I'll add to [Brand] — [N] ideas:

1. **[Title]** · [Pillar or No pillar] · [Platforms or —]
   [topicDetails if present]

2. ...

Save all to the Pad?
```

Wait for explicit confirmation before writing.

### Step B4: Write to Sked

For each confirmed idea:
- If the idea has only a title (no topicDetails or other enrichment): call `sked-client:smart_create_idea` with `title`, `groupId`, `contentPillarLabelId` (if resolved), `platforms`, and `stage`. Smart Create expands the title using Brand Kit context.
- If the idea has topicDetails, creativeBrief, or copyBrief already filled: call `sked-client:create_idea` with all available fields.

Write ideas sequentially. After all writes complete:

> *"Added [N] ideas to [Brand]'s Pad."*

---

## Mode C — Generate ideas

Claude does the creative work. Start from existing source material if the user has provided any;
otherwise use the Brand Kit as the sole brief.

### Step C1: Source existing material

Check whether the user has shared any of the following in this conversation:
- Notes, bullet points, or a content brief
- A document or article
- A previous brainstorm
- A set of themes or campaign focus areas

If material is present: use it as the primary creative brief, with Brand Kit (tone, pillars, brand
description) layered on top. If the user said "ignore the brand kit", use their material only.

If nothing is provided: the Brand Kit is the brief. Load brand description, tone of voice, and
active pillar list.

**Gap-fill sub-variant:** If the user references content gaps ("fill my gaps for next week",
"I need ideas for Monday and Wednesday") or has previously run `/calendar-review`, identify the
specific gaps (dates + platforms) and use them to constrain generation — target those days and
platforms in the ideas.

### Step C2: Generate the ideas

Produce the ideas using Claude's creative reasoning:
- Default 5 ideas unless the user specified a count
- Distribute across active content pillars proportionally (don't double up on one pillar unless the brand only has one)
- Each idea gets: **title**, **pillar** (matched to active pillar list), **platform(s)** (based on connected accounts), **one-line topicDetails**
- Ideas should feel distinct — vary format (video, carousel, single image, story, poll) and angle (educational, behind the scenes, product, community, seasonal) within the brief
- Ideas should be specific and actionable, not generic ("Share a coffee tip" is weak; "Show the exact grind size we use for our Cold Brew pourover — 30 seconds on the burr grinder" is strong)

### Step C3: Present for review

```
Here are [N] ideas for [Brand]:

1. **[Title]** · [Pillar] · [Platform(s)]
   [topicDetails]

2. ...
```

Ask which to save: *"Which would you like to add to the Pad? Say 'all', list numbers, or 'none'."*

### Step C4: Save selected ideas

For each selected idea, call `sked-client:smart_create_idea` with:
- `title` — the generated title
- `groupId`
- `contentPillarLabelId` — the resolved label ID for the assigned pillar
- `platforms` — the suggested platform list
- `topicDetails` — the one-line brief
- `stage: "pad"`

Write sequentially. Confirm count on completion:
> *"Added [N] ideas to [Brand]'s Pad."*

---

## Mode D — URL import

`create_idea_from_url` saves immediately — there is no draft-then-confirm step. The idea lands in
the Sked Pad as soon as the call is made. Claude's value here is in the strategic layer added after.

### Step D1: Create from URL

Call `sked-client:create_idea_from_url` with:
- `groupId`
- `url` — the URL from the user's message
- `stage: "pad"` (default)
- `contentPillarLabelId` — omit to let Sked auto-assign the first active pillar, unless the user specified one

The API fetches the link preview, sets the title from the page, attaches the URL, and creates the idea.

### Step D2: Read the content

Fetch and read the URL content (using WebFetch or equivalent) to understand what the page is about.
Do not rely on the title alone.

### Step D3: Present what was created + strategic suggestions

Show the user what landed in Sked, then offer 2–3 suggestions grounded in the URL content and
the Brand Kit:

```
Added to [Brand]'s Pad:
**[Title from URL]** · [Auto-assigned pillar]

Here's how this could work for [Brand]:

1. **[Angle]** — [One sentence on how to frame this piece of content for their audience and strategy]
2. **[Angle]** — [e.g. a format suggestion — Reel, carousel, story series]
3. **[Angle]** — [e.g. a caption hook or creative direction rooted in their tone of voice]
```

Suggestions should be specific to the URL content + the brand's strategy. Generic angles ("You
could post about this!") are not useful.

### Step D4: Apply suggestions (optional)

If the user wants any suggestions applied, call `sked-client:update_idea` with:
- `topicDetails` — the chosen angle written as a brief
- `creativeBrief` — visual or format direction
- `copyBrief` — caption hook or copy angle
- `contentPillarLabelId` — if the user wants to override the auto-assigned pillar

---

## Guardrails

- **Never write to Sked without confirmation**, except Mode D where `create_idea_from_url` saves immediately by design. Always make the save behaviour clear to the user upfront.
- **Stage is always `"pad"`** for new ideas unless the user explicitly says "working". Don't promote ideas to `"working"` without being asked.
- **Never fabricate pillar IDs.** Always resolve pillar names against the active label list from `list_content_labels`. If a pillar can't be matched, leave `labels` empty and note it.
- **Brand Kit is the brief by default.** If not set up, proceed without it — don't block the skill.
- **Mode B enrichment is one offer, not a negotiation.** Ask once. If the user says no, save as-is without re-asking per idea.
- **Mode C ideas must be specific.** Reject any generated idea that's too generic before presenting it to the user. Rewrite or replace it.
- **Mode D suggestions must be grounded** in actual URL content read in Step D2 + the Brand Kit. Never suggest angles that could apply to any brand or any URL.
- **Never surface internal IDs, field names, or raw API responses** to the user.
