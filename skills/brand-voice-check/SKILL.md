---
name: brand-voice-check
description: >
  Check or rewrite content against a brand's Tone of Voice from the Sked Brand Kit. Use this skill
  whenever a Sked user wants to know if content sounds on-brand, or wants it rewritten to match
  their brand voice. Trigger on: "brand voice check", "does this sound like us", "check this for
  brand voice", "is this on-brand", "apply my brand voice to this", "rewrite this in my brand
  voice", "does this match our tone", "flag what's off", "fix this for our brand", or any request
  to evaluate or improve content against a brand's voice or tone of voice profile.
---

# Brand Voice Check

Review or rewrite submitted content against the Brand Kit's Tone of Voice profile for a Sked
Brand Group. Two modes — **Review** (structured critique of what's off-brand and why) or
**Rewrite** (revised content that matches the brand's voice, with a summary of what changed).
Claude detects the mode from the user's intent and routes accordingly.

---

## Step 1: Resolve the Brand Group

Call `sked-client:list_brand_groups`.

- One group → use it silently.
- Multiple groups → if the user named a brand, match and proceed silently. If not, present the list by name and ask which brand to check against.

Hold the resolved `groupId` for all subsequent calls.

---

## Step 2: Detect mode

Read the user's message and route to the correct mode:

| Signal | Mode |
|---|---|
| "check this", "does this sound like us", "is this on-brand", "flag what's off", "review this" | **A — Review** |
| "rewrite this", "apply my brand voice", "fix this for our brand", "make this sound like us" | **B — Rewrite** |

If both signals are present, or the intent is ambiguous: ask one question.

> "Do you want me to flag what's off-brand with notes (Review), or rewrite it to match your voice (Rewrite)?"

---

## Step 3: Silent context load

Run both calls in parallel before doing anything else:

1. `sked-client:get_brand_kit` with `groupId`
2. `sked-client:list_content_labels` with `groupId`

Extract and hold:

| Field | Source | Used for |
|---|---|---|
| Brand description | `brandDescriptionResult` | Fallback when no ToV profile; context on who the brand serves |
| ToV profiles | `toneOfVoiceProfiles[]` | Primary evaluation source |
| Active ToV profile | If there’s only one non-placeholder profile, use it; if multiple exist, ask the user to choose (see Edge Cases) | The voice to evaluate against |
| Content pillars | `list_content_labels` filtered to `status: "ACTIVE"`, `type: "CONTENT_PILLAR"` | Context check — does the content serve a pillar? |
| Custom instructions | `customInstructions` (draftCaption, copyBrief, etc.) | Supplementary rules for caption and copy evaluation |

**Placeholder name detection:** If the profile name contains "test", "draft", "temp", or is a single word with unusual casing, treat it as a placeholder. Note it to the user but still evaluate against the `toneOfVoiceResult` content.

**If no ToV profiles exist:** Note it clearly. Fall back to Brand Description only (see Edge Cases).

---

## Step 4: Accept and validate the content

The user should have already pasted the content to evaluate in their message. If they haven't:

> "Paste the content you'd like me to check — caption, email, social post, product copy, whatever you have."

Wait for the content. Then:

- **Content type:** infer from context (caption, email subject, blog intro, ad copy, etc.). Name it in your output.
- **Length:** if the content is over ~800 words, process it in sections and flag this.
- **Format:** plain text paste only. If the user references a URL or file, note: "I can only check text you paste directly — I can't read files or URLs in this skill."

---

## Mode A — Review

Evaluate the submitted content against the Brand Kit's Tone of Voice profile. Produce a
structured critique that tells the user specifically what's off and why — not a general score.

### Evaluation dimensions

Run silently against all dimensions. Only surface dimensions where there is something to say.

**1. Voice and register** — Does the writing match the brand's overall voice? Reference the `toneOfVoiceResult` profile directly. Quote specific words or phrases from the submitted content that contradict the voice.

**2. Vocabulary** — Are there words the brand's ToV explicitly avoids? Flag exact phrases from the content. If the custom instructions include vocabulary guidance, check against those too.

**3. Tone consistency** — Does the content stay consistent in tone throughout, or does it shift register mid-way? Identify where the drift happens.

**4. Structural and format fit** — For captions: length and structure. For email: subject, preview text, and opening. Apply only the dimensions relevant to the content type.

**5. Pillar alignment (optional)** — If the content is clearly targeting a topic, note which Content Pillar it fits, or flag if it doesn't align with any active pillar. Only include if it adds signal.

### Output format — Review mode

```
## Brand voice check — [Brand Group name]
*Tone of Voice profile: [profile name]*

**Overall:** [One sentence verdict — on-brand / mostly on-brand / off-brand in key areas]

### What's working
[1–3 specific things the content gets right — reference the ToV profile. Quote the content.
Skip this section if there's nothing genuinely worth noting.]

### What's off

**[Dimension — e.g. Voice]**
[Specific critique. Quote the exact phrase from the content. Explain what the brand's ToV
says instead.]

**[Dimension — e.g. Vocabulary]**
[Specific critique...]

### Verdict
[2–3 sentences. Is this fixable with light edits, or does it need a rewrite? What's the
one most important thing to change?]

---
*Want me to rewrite this with the issues fixed? Just say the word.*
```

Do not invent feedback. If the content is genuinely on-brand, say so clearly and specifically.

---

## Mode B — Rewrite

Rewrite the submitted content so it sounds like the brand. Grounded in the `toneOfVoiceResult`
profile — not generic improvements. Do not change the meaning.

### Rewrite principles

- Match the voice described in the ToV profile — vocabulary, register, sentence length, rhythm.
- Apply custom instructions where relevant (caption length, hashtag approach, etc.).
- Do not add claims, facts, or offers not present in the original.
- Preserve the content type — if it's a caption, the rewrite is a caption.
- If the content is long (>800 words), rewrite in sections.

### Output format — Rewrite mode

```
## Brand voice rewrite — [Brand Group name]
*Tone of Voice profile: [profile name]*

### Rewritten
[Full rewritten content — clean, no annotations, ready to use]

---

### What changed
[3–5 bullet points. Each names a specific change and why it was made, referencing the ToV profile.
e.g. "Changed 'leverage' to 'use' — the profile avoids corporate jargon."]

---
[Offer to iterate, then check for save-back — see Step 5]
```

---

## Step 5: Save-back path (Rewrite mode only)

After delivering a Rewrite, check whether the content came from a Sked Idea.

**Signals that this is a Sked Idea:**
- The user mentioned an Idea title or ID, or previously ran the ideas skill in this conversation and the pasted content matches.
- The user says "this is from my Sked", "from my Idea Planner", or "I generated this in Sked".
- A `list_ideas_by_group` or `get_idea_activity` call exists in the conversation and the content matches it.

**If a Sked Idea is identifiable:** After the rewrite output, add:

> "This looks like it came from a Sked Idea — want me to save the rewritten version back? I'll update the caption/copy brief in the Idea Planner."

Wait for confirmation. If yes: call `sked-client:update_idea` with the idea's `_id` and set `copyBrief` to the rewritten version. Confirm: "Updated in Sked — [Idea title] now has the revised version."

**If the content is freeform (not from a Sked object):** Do not offer to save back. The user is workshopping.

---

## Edge cases

**No Tone of Voice profile in the Brand Kit**

> "No Tone of Voice profile is set for [brand] yet — I'll evaluate against your Brand Description instead. You can set up a full ToV profile in `setup-brand-kit` for more precise checking in future."

For Review mode: evaluate against the brand description's identity, audience, and values.
For Rewrite mode: use the brand description as the creative brief.

**Multiple ToV profiles**

> "You have [N] Tone of Voice profiles: [list names]. Which should I check against?"

Wait for selection. If the user says "all", check against the primary profile only and note it.

**ToV profile exists but has a placeholder name**

Use the `toneOfVoiceResult` content as normal. Add a single note: "Your ToV profile is named '[placeholder name]' — worth updating in Sked Settings."

**Content is a single word or emoji**

> "This is pretty short to evaluate — can you share more context? What is this being used for, and what's the full content?"

**User wants both modes**

Run Review first, then offer: "Want me to rewrite it with those issues fixed?"

---

## Guardrails

- Never rewrite without reading the ToV profile. Always load `get_brand_kit` first.
- Never invent negative feedback. If the content is on-brand, say so.
- Never change the meaning. Rewrites match voice, not message.
- Never surface raw API field names, IDs, or profile metadata to the user.
- Custom instructions are supplementary. `toneOfVoiceResult` is the primary signal.
- Save-back requires confirmation. Never call `update_idea` without the user explicitly asking.
- One ToV profile per evaluation. Don't evaluate against multiple profiles in a single run.
