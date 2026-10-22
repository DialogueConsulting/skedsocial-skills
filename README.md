# Sked Social Skills and MCP

Use your Sked Brand Kit, content calendar, analytics, ideas, and brand voice from Claude or ChatGPT. The Sked MCP server is hosted by Sked Social and uses OAuth, so each person signs in with their own Sked account—no API keys or tokens are stored in this repository.

## What you get

| Skill | What it does |
| --- | --- |
| `setup-brand-kit` | Full Brand Kit health check and guided setup for descriptions, tone of voice, content pillars, and custom instructions |
| `calendar-review` | Two-week content calendar review with local timezone conversion, gap detection, and distribution analysis |
| `brand-summary` | Thirty-day analytics summary with period-on-period, pillar, and platform breakdowns |
| `ideas` | View, write, generate, and import ideas into the Sked Idea Planner |
| `brand-voice-check` | Review content against your brand voice or rewrite it to match your Brand Kit |
| `create-draft` | Create an unscheduled draft from media already in your Sked Library, with your caption saved exactly as written and an optional content pillar and approval status |
| `create-post` | Create and schedule a post with a date and time, then optionally tailor captions, first comments, media, times, and approval statuses per platform from your own source material such as a spreadsheet or Notion page |

## Prerequisites

1. A Sked Social account on a **Professional, Accelerate, or Enterprise** plan.
2. At least one Brand Group configured in Sked.
3. Access to Claude or ChatGPT with custom MCP connections enabled by your account or workspace.
4. For `create-draft` and `create-post`: the media you want to post must already be in your **Sked Library**. The skills cannot upload files or import from a link, but they will walk you through uploading anything hosted elsewhere, such as Notion, Google Drive, Dropbox, or a design tool.

## Connection details

| Setting | Value |
| --- | --- |
| MCP server URL | `https://app-mcp.skedsocial.com/mcp` |
| Transport | Remote streaming HTTP |
| Authentication | OAuth sign-in with Sked Social |
| Required scope | `sked:mcp` |

Always use the HTTPS URL above. The server discovers its OAuth configuration automatically; do not add API keys, bearer tokens, client secrets, or custom request headers.

## Install in Claude

### Install the skills and bundled MCP connection

Add this GitHub repository as a Claude marketplace, then install the plugin:

```text
/plugin marketplace add DialogueConsulting/skedsocial-skills
/plugin install sked-skills@sked-skills
```

Or use the Claude Code CLI:

```bash
claude plugin marketplace add DialogueConsulting/skedsocial-skills
claude plugin install sked-skills@sked-skills
```

The plugin includes [`.mcp.json`](.mcp.json), which registers the remote Sked server as `Sked_Social`. When Claude requests the connection, choose **Sign in** and complete the Sked OAuth flow. Then start a new conversation and ask, for example, “What does my calendar look like?”

> **Upgrading from an earlier version?** The server was previously registered as `sked-client` and is now **Sked Social** (`Sked_Social`). If you added it to your own Claude Desktop, Cowork, or project config under the old name, rename that entry to `Sked_Social`. The skills look up tools by this name, so they will not find the Sked tools under the old one.

### Add the remote connector manually

If your Claude surface does not install the bundled connection, add a custom web connector named **Sked Social** with the HTTPS MCP server URL above. Use that exact name so the skills can find the tools. Allow Claude to detect the OAuth configuration, use Claude’s published OAuth identity or automatic registration when offered, then sign in with your Sked account. No fixed request headers are required.

For Team and Enterprise workspaces, an owner or permitted administrator adds the connector once in **Organization settings → Connectors**. Each member then enables the connector and signs in with their own Sked account.

## Install in ChatGPT

ChatGPT connects to the hosted MCP server through Apps & Connectors; it does not install the Claude skill files directly from GitHub. This repository provides the skill package and the documented server URL.

### Individual setup

1. In ChatGPT on the web, open **Settings → Apps & Connectors → Advanced Settings** and enable **Developer Mode**.
2. Return to **Apps & Connectors** and create a connector.
3. Name it **Sked Social** and enter `https://app-mcp.skedsocial.com/mcp` as the Server URL.
4. Choose OAuth authentication and accept the server’s discovered configuration.
5. Review the connector safety prompt, create the connector, and complete Sked sign-in.
6. Enable the Sked Social connector in a new chat, then ask it to work with your Sked data.

### Business, Enterprise, and Edu workspace setup

1. An administrator enables **Developer Mode** under **Settings → Apps & Connectors → Advanced Settings**, as required by the workspace.
2. In **Workspace settings → Apps & Connectors**, create a connector using `https://app-mcp.skedsocial.com/mcp`.
3. Scan the tools, sign in to Sked when prompted, and test both a read action and a confirmation-gated write action.
4. Publish the approved connector to the workspace.
5. Members enable the connector and complete OAuth with their own Sked account.

Workspace administrators should test and approve the tool permissions before publishing. OAuth is per person: users only access Sked data available to their own account.

## Using the skills

Once connected, ask Claude to:

- **“Set up my brand kit”** for a guided Brand Kit health check and setup.
- **“What does my calendar look like?”** for a fortnightly calendar review.
- **“How’s [brand] doing this month?”** for analytics and period comparisons.
- **“Generate 5 ideas for next week”** to create ideas from your Brand Kit.
- **“Does this sound like us?”** to review or rewrite content in your brand voice.
- **“Save this image as a draft for [brand] on Instagram”** to create an unscheduled draft from media in your Library.
- **“Schedule this for Tuesday at 9am”** to create and schedule a post, with optional platform-specific captions, creative, and statuses from your own source material.

The `ideas` skill asks for confirmation before it writes to Sked. Review the proposed change before approving it. `create-draft` and `create-post` write through a preview step and ask for confirmation whenever something in your request is ambiguous, such as a date with no time or more than one possible media match.

`create-post` builds on `create-draft`, so install the plugin as a whole rather than copying a single skill.

## Troubleshooting and access

- **Sign-in does not start:** reconnect using the HTTPS MCP server URL. A `401` response before sign-in is expected; it tells the client where to discover the Sked OAuth flow.
- **Connection expired:** reconnect the Sked Social connector and complete OAuth again. You can also revoke its access from your Sked account or the relevant AI product’s connected-app settings.
- **Skills cannot find the Sked tools:** if you set up the connection before the rename from `sked-client`, rename your server entry to `Sked_Social` (Claude) or reconnect the connector named **Sked Social**.
- **Media not found when creating a post:** `create-draft` and `create-post` only use media already in your Sked Library. Upload the file there first; the skill will guide you if it is hosted elsewhere.
- **No Brand Groups or data:** verify the signed-in Sked account has access to the intended Brand Group and is on an eligible plan.
- **Workspace connector missing:** ask a ChatGPT or Claude workspace administrator to approve and publish the custom connector.

For Sked account and product support, visit the [Sked Help Centre](https://support.skedsocial.com). For issues with these skills or connection documentation, [open an issue](https://github.com/DialogueConsulting/skedsocial-skills/issues) in this repository.

## Updating Claude

```text
/plugin marketplace update sked-skills
```

Or:

```bash
claude plugin marketplace update sked-skills
```
