---
name: setup-brand-kit
description: "Set up or repair the Brand Kit for a Sked Brand Group. Runs a health check, establishes the Brand Description, confirms Tone of Voice, sets up Content Pillars, and optionally configures Custom Instructions. Use when the user says things like 'set up my brand kit', 'set up [brand name]', 'help me get started with [brand]', 'brand kit setup', or 'onboard [brand name]'."
---

# setup-brand-kit

Set up or repair the Brand Kit for a Sked Brand Group. Runs a health check, establishes the Brand Description, confirms Tone of Voice, sets up Content Pillars, and optionally configures Custom Instructions.

## Triggers

Use this skill when the user says things like:
- "set up my brand kit"
- "set up [brand name]"
- "my brand kit isn't set up"
- "help me get started with [brand]"
- "onboard [brand name]"
- "brand kit setup"
- "let's set up the brand"

Also trigger when `get_brand_kit_setup_status` returns `isBrandKitConfigured: false` or any `configuredFields` are false, as a proactive prompt.

---

## Overview

This skill walks through Brand Kit setup in a single guided session. It opens with a health check to understand what's already in place, then works through each foundation in order — only fixing what needs fixing. It does not touch the Idea pipeline.

**Sections covered:**
1. Health Check (always)
2. Brand Description (1.1)
3. Tone of Voice (check only — assumed set at account connection)
4. Content Pillars (1.3)
5. Custom Instructions (optional — user decides)
6. Closing summary

---

## Tool Chain

Load all of these in **parallel** at the start — do not make sequential calls:

```
list_brand_groups
get_brand_kit
get_brand_kit_setup_status
list_social_accounts
list_content_labels
get_brand_group_analytics_summary
```

Write tools used as needed:
```
update_brand_description
create_content_label
update_brand_kit_custom_instructions
```

Web tools used in Brand Description step:
```
mcp__workspace__web_fetch   (primary)
WebSearch                   (fallback if fetch fails)
```

---

## Brand Group Selection

If the user has **one Brand Group**, select it automatically and proceed.

If they have **multiple**, list them by name and ask which to set up. Use the Brand Group name naturally in all subsequent questions — not the ID.

---

## Step 0 — Load Context & Detect Tone

Before doing anything else:

1. Check `toneOfVoiceProfiles` in the Brand Kit response.
2. If one or more profiles exist, read the `toneOfVoiceResult` of the first active profile.
3. **Carry this tone through the entire skill session.** The questions you ask, the suggestions you make, and the copy you generate should feel grounded in the brand's own voice — not generic assistant phrasing.
   - A playful lifestyle brand → warmer, more casual questions
   - A professional services firm → direct, businesslike
   - A youth organisation → inclusive, energetic
4. If no ToV profile exists, proceed in a neutral helpful tone.

Do not announce this step to the user. It is silent context loading.

---

## Step 1 — Health Check

Using the data loaded in parallel, assess each area and present a single diagnostic summary. Keep it scannable — one line per area.

### Assessment criteria

**Brand Description**
- ✅ Set and passes quality check (see below)
- ⚠️ Set but incomplete (missing one or more of the five criteria)
- ❌ Missing

Five completeness criteria (assess silently — do not show the rubric):
1. What the brand does
2. Who they serve
3. What makes them different
4. Values or mission
5. Market or geography

**Tone of Voice**
- ✅ Profile present and named properly
- ⚠️ Profile present but name looks like a placeholder (e.g. "test", "TEst", "draft", unnamed)
- ❌ No profile (note: ToV is set at account connection — if missing, flag but do not attempt to set it in this skill)

**Content Pillars**
- ✅ 3–6 active pillars, all with descriptions
- ⚠️ Pillars present but: too many (>6), some missing descriptions, or identity appears mixed across multiple brand personas
- ❌ No active pillars

**Custom Instructions**
- ✅ Configured
- — Not configured (optional — do not show as ❌)

**Connected Accounts**
- If `list_social_accounts` returns accounts for this Brand Group → ✅ accounts connected
- If `get_brand_kit_setup_status` shows `hasConnectedAccounts: false` but accounts exist in `list_social_accounts` → note gently: "Accounts are connected — you may want to check in Sked Settings that the Brand Kit integration is fully linked."
- If no accounts in either → flag clearly: "No social accounts connected yet. You'll want to add these in Sked before generating content."

### Presenting the health check

Show the diagnosis clearly, then state what the session will cover:

> "Here's where things stand for **[Brand Group name]**:
>
> ✅ Tone of Voice — [profile name] is set
> ⚠️ Brand Description — set but missing [gap]
> ⚠️ Content Pillars — [N] active, but [issue]
> — Custom Instructions — not configured (optional)
>
> Let's work through these. I'll start with the Brand Description."

If everything is already complete and healthy, say so and offer to review any specific area rather than running the full setup.

---

## Step 2 — Brand Description

### If the Brand Description is already complete (all 5 criteria met)

Say so briefly and move on:

> "Your Brand Description looks solid — covers what you do, who you serve, and what makes you different. Moving on to pillars."

### If the Brand Description is missing or incomplete

Ask how they want to approach it. Offer two paths:

> "To write your Brand Description, I can either fetch it from your website or you can tell me about the brand directly. Which works better?"

**Path A — Website fetch**

Ask for the URL. Then:
1. `web_fetch` the homepage. If that returns little content, also fetch `/about` or `/about-us`.
2. If `web_fetch` fails or returns a JavaScript shell with no content, try `WebSearch` for "[brand name] about" to find a description page.
3. Generate a Brand Description from the fetched content (150–200 words, third person, factual).
4. Assess against the five criteria. If geography is missing but the website implies a region, include it.
5. Show the draft to the user with a brief note on what it covers. Ask for confirmation or edits.
6. On confirmation: `update_brand_description` with `inputs.websiteUrls` set to the fetched URL and `brandDescriptionResult` set to the agreed text.

**Path B — Manual**

Ask conversationally — frame the questions in the brand's tone if ToV is loaded:

> "Tell me about [brand] — what do they do, who are their customers, and what makes them different from others in the space?"

Accept freeform. Follow up if anything from the five criteria is unclear. Then generate a 150–200 word description, show it, confirm, and save with `update_brand_description` using `brandDescriptionResult` only (no `websiteUrls`).

### Quality gate

Before saving, silently check the generated text against all five criteria. If any are missing, either add them from context or ask a single targeted question (e.g. "Do they work with clients in a specific region, or is it global?"). Do not ask more than one follow-up.

---

## Step 3 — Tone of Voice (Check Only)

This is a brief check, not a setup step.

If a ToV profile is present:
- If the name looks like a placeholder (contains "test", "draft", "TEst", "temp", or is a single word with unusual capitalisation) → note it: "You have a ToV profile set — worth renaming it in Sked Settings when you get a chance. The content looks good."
- Otherwise → acknowledge and move on: "Tone of Voice is set. We'll use [profile name] as the voice for suggestions."

If no ToV profile:
- Note it without alarm: "No Tone of Voice profile set yet — that's usually configured when you connect accounts. For now I'll work from the Brand Description."

Do not attempt to create or update ToV profiles in this skill.

---

## Step 4 — Content Pillars

### If pillars are healthy (3–6 active, all with descriptions, consistent identity)

Confirm briefly and show the current set:

> "You have [N] active Content Pillars:
> - [Name] — [description]
> - [Name] — [description]
> ...
> These look good. Want to make any changes, or should we move on?"

### If pillars need attention

Surface the specific issues clearly:
- **Too many (>6):** "You have [N] active pillars — that's more than most brands need, and it can make content decisions harder. I'd suggest trimming to 4–6."
- **Empty descriptions:** "A few pillars are missing descriptions — [list names]. These will still work but the AI can't reference them properly."
- **Mixed brand identity:** "Your pillars seem to span a few different brand directions — [e.g. coffee content, Sked content, quokka content]. Before I suggest a clean set, I need to understand what this brand is actually focused on."

Then offer three ways to work through it:

> "A few ways we can sort this out:
> 1. **Paste in content** — share brand guidelines, strategy docs, or a list of post ideas and I'll extract the pillars from those
> 2. **I suggest, you confirm** — I'll propose 4–6 pillars based on what I know about the brand and you tell me what fits
> 3. **You tell me** — describe the content themes that matter for this brand and I'll build them out"

### Generating pillars

Whichever path is chosen, produce a full set before writing anything. For each pillar generate:

- **Name** — short, clear, ideally 2–3 words
- **Emoji** — one that fits the theme
- **Colour** — a hex value that distinguishes it from others in the set (vary across the palette)
- **Description** — 1–2 sentences: what this pillar is for, what kind of content lives here. Written to help the AI understand what to generate, not just to label the category.

Show all proposed pillars together for review:

> "Here's what I'm thinking for [brand]:
>
> **[Emoji] [Name]**
> [Description]
>
> **[Emoji] [Name]**
> [Description]
> ...
>
> Does this feel right? You can rename anything, swap the emoji, or ask me to rethink a pillar before I create them."

Wait for explicit confirmation before calling `create_content_label`. Do not write partial sets.

### Writing pillars

On confirmation, call `create_content_label` for each pillar with:
- `type: "CONTENT_PILLAR"`
- `groupId`
- `name`, `description`, `emoji`, `color`

If existing pillars need to be retired (user confirms), call `archive_content_label` for each one. Do not archive without explicit user instruction.

### Return pillar descriptions

After creating, show a clean summary of what was written — name + description for each pillar. Users should have a clear record of what is now in their Brand Kit.

---

## Step 5 — Custom Instructions (Optional)

Keep this light. Do not present it as a problem to fix.

> "One optional thing: Sked AI can follow specific instructions when generating copy briefs, topic details, captions, and hashtags for [brand]. These are blank by default — Sked uses its own defaults, which work well. Worth setting if [brand] has strong preferences around format, length, or what to avoid.
>
> Want to set these now, or skip for now?"

If they say **skip** → close the session.

If they say **yes** → work through the two most impactful fields first:

**Caption style** (`draftCaption`)
Ask what matters for captions: length, structure, whether to always end with a question, things to avoid. If ToV is loaded, frame this relative to the brand's voice.

**Topic & details** (`topicAndDetails`)
Explain: "This is the framework Sked AI uses when suggesting what a post should be about and how to shoot or design it — things like always recommending a Reel format, always suggesting a specific visual style, or always referencing a seasonal angle."

Generate a short instruction for each from the conversation. Show before saving.

If they want to set the remaining three (`copyBrief`, `creativeBrief`, `hashtags`) as well, proceed. Otherwise save what's been agreed and note the others can be added later.

Save with `update_brand_kit_custom_instructions`.

---

## Step 6 — Closing Summary

End with a brief, clear summary of what was set or confirmed this session. Return the pillar descriptions so the user has them in the conversation.

> "**[Brand Group name] is set up.** Here's what we covered:
>
> ✅ Brand Description — [one line on what it covers]
> ✅ Tone of Voice — [profile name confirmed / noted]
> ✅ Content Pillars — [N] pillars created:
>    - **[Emoji] [Name]** — [description]
>    - **[Emoji] [Name]** — [description]
>    ...
> [✅ Custom Instructions — set / — Skipped for now]
>
> You're ready to start generating content. Want to create some ideas now, or is there anything here you'd like to adjust?"

---

## Edge Cases

**Brand Group has no social accounts at all**
Note it clearly after the health check. Proceed with Brand Kit setup — the Brand Description and pillars can be configured without connected accounts. Remind them at the close that they'll need to connect accounts in Sked Settings before publishing.

**Brand Description was set from a website that no longer exists or returns no content**
If a stored `websiteUrls` input exists, try fetching it. If it fails, fall back to Manual path (Step 2 Path B), telling the user: "I couldn't reach [url] — can you tell me about the brand directly, or share a new URL?"

**All pillars have descriptions but there are more than 10 active**
Flag the volume. Do not auto-archive. Suggest the user review and archive the ones that no longer apply, and offer to help them identify candidates based on names/descriptions. Only archive on explicit instruction.

**ToV profile has detailed content but a bad name**
Do not regenerate or overwrite it. Just flag the name as something to update in Sked Settings. The content is what matters.

**User wants to set up a second Brand Group in the same session**
Finish the current one cleanly, then offer: "Want to set up another Brand Group?" If yes, restart from Step 1 with the new group selected.
