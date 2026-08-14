# Sked Social Skills for Claude

AI skills that bring your Sked brand data — Brand Kit, content calendar, analytics, and ideas — directly into Claude Desktop or Claude Code.

**Available skills:**

| Skill | What it does |
|---|---|
| `setup-brand-kit` | Full Brand Kit health check and guided setup — Brand Description, Tone of Voice, Content Pillars, Custom Instructions |
| `calendar-review` | 2-week content calendar with local timezone conversion, gap detection, pillar/platform/post-type distribution |
| `brand-summary` | 30-day analytics summary with period-on-period comparison, pillar performance, and platform breakdown |
| `ideas` | View, write, generate, and import ideas into the Sked Idea Planner |

---

## Prerequisites

These skills require:

1. A Sked Social account on a **Professional, Accelerate, or Enterprise plan**
2. The **Sked MCP server** configured in Claude — see the [Sked Help Centre](https://support.skedsocial.com) for setup instructions
3. At least one Brand Group configured in Sked

The skills work by calling the Sked API directly from Claude. Without the MCP connection, the skill instructions will load but no data can be fetched.

---

## Install — Claude Desktop & Claude Code

### Option 1: Install the plugin directly

```
/plugin install DialogueConsulting/skedsocial-skills
```

Or via CLI:

```bash
claude plugin install DialogueConsulting/skedsocial-skills
```

### Option 2: Add as a marketplace

```
/plugin marketplace add DialogueConsulting/skedsocial-skills
```

Then install:

```
/plugin install sked-skills@sked-skills
```

### Option 3: Pin to your project (team install)

Add to `.claude/settings.json` in your project repo so everyone on the team gets it automatically:

```json
{
  "extraKnownMarketplaces": {
    "sked-skills": {
      "source": {
        "source": "github",
        "repo": "DialogueConsulting/skedsocial-skills"
      }
    }
  },
  "enabledPlugins": {
    "sked-skills@sked-skills": true
  }
}
```

Commit and push. Anyone who opens the project in Claude will be prompted to install.

---

## Using the skills

Once installed and connected to Sked, start a conversation and say what you need:

- **"Set up my brand kit"** → `setup-brand-kit` runs a health check and guides you through each section
- **"What does my calendar look like?"** → `calendar-review` pulls the next fortnight with timezone conversion and gap detection
- **"How's [brand] doing this month?"** → `brand-summary` pulls 30-day analytics with period-on-period comparison
- **"Generate 5 ideas for next week"** → `ideas` uses your Brand Kit as the creative brief and presents options to save

Skills can be combined across a session. Run a calendar review, spot a gap, then generate ideas to fill it — all in the same conversation.

---

## ChatGPT (Manual Install)

ChatGPT doesn't yet natively support the Claude plugin format, so these skills can't be installed the same way. Here's what's currently possible.

### What works without the Sked MCP backend

The skill instructions are detailed workflow prompts. You can paste a skill's `SKILL.md` content into a **ChatGPT Project's system prompt** (or a custom GPT's instructions) and use it with data you provide manually — copy-paste your analytics, calendar view, or brand notes into the chat, and the skill will guide the analysis.

**Steps:**
1. Open the `skills/<skill-name>/SKILL.md` file in this repo
2. Copy the full contents
3. In ChatGPT, create a new Project and paste the SKILL.md contents into the Project Instructions
4. In a chat, provide your Sked data manually (e.g. paste your analytics summary, calendar view, or brand guidelines)

This gives you the structured output format and reasoning logic without live API access.

### What requires the Sked MCP backend

All API calls — fetching brand groups, pulling analytics, reading the calendar, writing ideas — require the Sked MCP server. ChatGPT supports MCP natively; when the Sked MCP server supports native connector authentication, the full skill suite will work in ChatGPT exactly as it does in Claude.

Until then, full functionality (live data) is available in **Claude Desktop** and **Claude Code** only.

### Recommended path

If your team is split across Claude and ChatGPT, set up the skills in Claude for live data workflows. Use the manual ChatGPT approach for structured analysis when you already have data to hand.

---

## Updating

```
/plugin marketplace update sked-skills
```

Or:

```bash
claude plugin marketplace update sked-skills
```

---

## Support

- **Sked Help Centre:** [support.skedsocial.com](https://support.skedsocial.com)
- **Issues with these skill files:** [open an issue](https://github.com/DialogueConsulting/skedsocial-skills/issues) in this repo
