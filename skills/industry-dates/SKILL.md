---
name: industry-dates
description: Add social media holidays, national days and industry events to a Sked Social brand group's content calendar as calendar notes or ideas, matched to the brand's Brand Kit. Use when someone asks for key dates, holidays or cultural moments to plan content around.
---

# Industry Dates

Add social media holidays, national days and industry events to a brand group's content calendar in Sked Social, as calendar notes or ideas, so you never miss a relevant cultural moment.

This skill adds notes and ideas only. It never creates, schedules or publishes posts.

---

## When to use this skill

Use it when someone asks to:
- Add social media holidays or national days to the calendar
- Find relevant dates or cultural moments for a brand
- Fill the calendar with content hooks for the next 90 days (or another period)
- Be reminded of upcoming moments that suit a brand

---

## Inputs

Collect these before starting. If something required is missing, ask. Don't assume.

1. **Brand group** (required). If several are named, queue them and process one at a time (see "Multiple brand groups").
2. **Timeframe** (optional, default 90 days from today). Accept natural language such as "next quarter" or "rest of this year" and convert it to a date range. Soft limit of 12 months: if asked for more, run 12 months and say so.
3. **Output type** (optional):
   - `notes`: add the relevant dates as calendar notes
   - `ideas`: add the relevant dates as ideas
   - `both`: notes for fixed awareness days, ideas for dates with a clear content angle
   - Not specified: show a ranked shortlist and wait for confirmation before writing anything (Step 6)

---

## Steps

### Step 1: Verify the brand group

Call `list_brand_groups` and match the name the user gave. If there is no clear match, stop, list the available brand groups and ask which one they meant. Don't write anything until the brand group is confirmed.

### Step 2: Load the Brand Kit

Call `get_brand_kit` with the brand group ID. Use the brand description, tone of voice profiles, custom instructions and any content pillars as the filter for Step 4. Also note the brand's location or region if the Brand Kit or accounts make it clear. It decides which regional dates apply.

**If the Brand Kit is empty or missing:** soft block. Say:

> "I can find dates for [brand group], but results will be much better with a Brand Kit set up. It tells me what the brand is about so I can pick dates that fit. Would you like to set up the Brand Kit first, or should I add a general set of dates without filtering?"

If they choose to continue, run with no filter and label every date as unfiltered in the summary.

### Step 3: Build the dates list

Use `WebSearch` to confirm dates for the timeframe. Many dates move year to year. Prefer published social media holiday calendars and official sources, and check the year of the source.

Look across these groups, then keep what fits the brand:
- **Universal awareness days**: for example International Women's Day, Earth Day, World Mental Health Day, World Kindness Day, International Day of Persons with Disabilities, Human Rights Day
- **Food, drink and lifestyle**: for example National Coffee Day, World Chocolate Day, World Vegan Day (only for brands where it fits)
- **Business and creator moments**: for example Small Business Saturday, World Entrepreneur Day, World Creativity Day
- **Platform-native moments**: for example World Emoji Day, World Photography Day
- **Seasonal and retail moments**: for example Black Friday, Giving Tuesday, Christmas, Valentine's Day
- **Regional dates matched to the brand's location**: for example Australia Day, EOFY, Melbourne Cup and the Australian Open (AU); Thanksgiving and Super Bowl (US); Bank Holidays (UK). Use only the region the brand operates in.
- **Local events**: major sporting, cultural or trade events in the brand's city or industry, if the brand fits

Build a master list with: date, name, category (awareness, lifestyle, business, platform, seasonal, regional, local) and a one-line content angle.

Some dates are sensitive or contested in some regions. Include them only if the brand clearly fits, and flag them in the shortlist so the user can decide.

### Step 4: Filter for brand fit

Compare each date with the Brand Kit:
- **Strong fit**: clear overlap with the brand and a natural content angle. Mark as an `idea` candidate.
- **Moderate fit**: broad awareness day that suits the brand's tone. Mark as a `note` candidate.
- **No fit**: unrelated and no plausible angle. Leave it out silently.
- **Unfiltered mode**: keep everything and label it unfiltered.

### Step 5: Check the calendar for existing items

Call `review_calendar` with the brand group ID and `startDate` / `endDate` for the timeframe. Skip any date where a matching note, idea or post already exists, so nothing is doubled up. Count skipped dates for the summary.

### Step 6: Confirm before writing (if output type wasn't specified)

Show a ranked shortlist of the best 10 to 15 matches:

> Here are the best matches for [brand group] over [timeframe]:
>
> 📌 **Notes** (worth acknowledging)
> - [Date]: [Name]
>
> 💡 **Ideas** (clear content angle)
> - [Date]: [Name]: "[content angle]"
>
> Want me to add all of these, or skip any?

Wait for confirmation before writing. If the user wants a date developed into a full idea, they can use their ideas tools alongside this skill.

### Step 7: Write to the calendar

Write in batches of 10 and report progress between batches if there are more than 20 items.

**Notes**: call `list_social_accounts` once and use the accounts in the brand group. Notes belong to accounts, not brand groups. Then call `add_calendar_note` with:
- `accountIds`: the brand group's account IDs
- `date`: YYYY-MM-DD
- `timezone`: the user's timezone (shown in `review_calendar` as `userTimezone`)
- `title`: the date name, for example "World Kindness Day"
- `description`: one line of context or a suggested angle

**Ideas**: call `create_idea` with:
- `groupId`: the brand group ID
- `title`: the content angle, for example "World Soil Day: where great coffee actually starts"
- `topicDetails`: what the day is, why it fits the brand, and one or two concrete directions the team could take
- `scheduledAt`: the date as YYYY-MM-DD, so the idea appears on the calendar that day
- `stage`: `pad`

An idea with a `scheduledAt` date shows on the calendar on that day. It is not a scheduled post.

### Step 8: Report back

> Done! Added [X] items to [brand group]'s calendar for [date range]:
>
> 📌 [X] notes added
> 💡 [X] ideas created
> ⏭️ [X] dates skipped (already on the calendar)
>
> **Best ideas to develop first:**
> - [Idea title], [date]
> - [Idea title], [date]
> - [Idea title], [date]
>
> Want me to run this for another brand group, or look further ahead?

Mention any dates flagged as sensitive, and say if the run was unfiltered.

---

## Multiple brand groups

- Queue them and process one at a time, in the order given
- Run the Brand Kit check and date filtering separately for each
- Finish with one combined summary

---

## Edge cases

| Situation | What to do |
|---|---|
| Brand Kit empty | Soft block: offer to set it up first, or continue unfiltered |
| Date already on the calendar | Skip it and count it in the summary |
| Timeframe over 12 months | Cap at 12 months and say so |
| No strong matches | Say so, then offer to run unfiltered or widen the timeframe |
| Brand group not found | Stop and ask the user to confirm the name |
| Output type not specified | Show the shortlist and wait for confirmation |
| Brand group has no connected accounts | Ideas still work; notes can't be created, so tell the user |
| Brand region unclear | Ask which region to use for regional dates |

---

## Notes

- Confirm the brand group before writing anything
- Always check the calendar first (Step 5) so repeat runs don't add duplicates
- This skill never publishes or schedules posts
- Turning an idea into a draft or post is a separate step in Sked Social

---

## Tools used

- `list_brand_groups`
- `get_brand_kit`
- `review_calendar`
- `list_social_accounts`
- `add_calendar_note`
- `create_idea`
- `WebSearch`
