---
name: create-draft
description: >
  Create an unscheduled draft in Sked Social from media that is already in the Sked Library.
  Resolves the target accounts or Brand Group, finds the media in a Library collection, saves
  the caption exactly as written, and optionally tags a content pillar and sets an approval
  status. Use this skill whenever someone wants to put a post into Sked as a draft without a
  publish date. Trigger on: "create a draft", "save this as a draft", "draft a post for [brand]",
  "add this to my drafts", "put this image in Sked as a draft", "make a draft from [collection]",
  "draft this for Instagram", or any request to create a post in Sked with no date or time. If
  the user gives a date or time to publish, use create-post instead.
---

# Create Draft

Your job is to create one unscheduled draft in Sked from existing Library media. A draft has
no date, lands in the workspace's default approval status, and is never published by this
skill. This skill writes to Sked, but only through a preview step followed by a commit.

`create-post` loads this skill for its first stages, so keep the steps below self-contained.

---

## Step 1: Gather the request

Pull these out of the user's message:

| Input | Required | Notes |
|---|---|---|
| Target | Yes | One or more accounts, or a Brand Group plus platform(s) |
| Media | Yes | File names, keywords, or a collection name. 1 to 10 items, in the order given |
| Caption | Yes | Saved exactly as supplied |
| Content pillar | No | Only applied if the user names one |
| Approval status | No | Only applied if the user asks, or the source data states one |

If a required input is missing, ask for everything that is missing in a single message.
If the user gives a publish date or time, stop and use `create-post` instead.

---

## Step 2: Resolve the target accounts

Call `Sked_Social:list_brand_groups` and `Sked_Social:list_social_accounts`.

Filter the account list by `brandGroupId`. Resolve the target like this:

- **A specific account named** (e.g. "Marigold & Rye Instagram"): match on Brand Group name plus platform.
- **A Brand Group named with no platform**: if the group has more than one account, ask which platforms to use. Do not assume every account.
- **A platform named with no brand** (e.g. "my Instagram"): if exactly one account on that platform exists, use it. If several exist, list them by Brand Group and ask.
- **One Brand Group exists in total**: use it silently.

Platform detection: use `platformType` when present. Instagram accounts can come back with no
`platformType` (known gap, tracked as SKE-5456), so treat an account with no `platformType`
whose name contains "Instagram" as Instagram.

Hold the resolved account IDs and the Brand Group ID(s) for later calls.

### Check connection health

Look at `connectionStatus` on every resolved account. If any is not `connected`
(`needs_reauth`, `paused`, `unavailable`, `unknown`), warn the user before going further:

> Heads up: [Account name] isn't fully connected ([connectionStatusMessage]). I can drop it
> and continue with the others, or you can reconnect it in Sked and I'll retry. Which would you like?

Wait for the answer. If the user chooses to continue with the account anyway, still run the
preview in Step 5 and report plainly if it is rejected.

---

## Step 3: Resolve the media

The Sked MCP can only use media that is already in the Library. It cannot upload files or
import from a URL.

1. Call `Sked_Social:search_media_collections`.
   - If the user named a collection (e.g. "recent uploads"), pass a short literal `query` from that name.
   - If no collection was named, browse without a query and ask which collection to use. Never guess a collection from a brand or Brand Group. Collections are workspace-wide, not per brand.
   - `resolution: "unique_match"` means you can proceed. Anything else: list the matches by name and ask.
2. Call `Sked_Social:list_media_assets` with the `collectionId` and a short literal `query` (a file name without the extension, or a distinctive keyword). Matching is literal, not semantic, so never pass a full sentence.
3. Pick the asset:
   - One clear match on file name or name: use it.
   - Several plausible matches: list them (file name, type, dimensions) and ask.
   - Visual requests ("the photo with the coffee mug"): try short keywords against the stored vision descriptions and tags. If that does not give a clear single match, ask.
4. Keep the media in the order the user gave. Maximum 10 items.

Use each asset's `assetId` (not `inboxEntryId`).

If nothing matches, tell the user what you searched for, link the collection, and go to
"If the media isn't in the Library" below.

### If the media isn't in the Library

If the media lives somewhere Sked can't reach, give the user these steps, then wait for them to say it is uploaded and search again.

1. Get the file onto your computer:
   - **Notion:** open the image or file and download it, or export the page and take the file from the export. Notion-hosted files need a Notion login, so Sked can't fetch them.
   - **Google Drive / Dropbox / OneDrive:** download the file from the share link.
   - **Canva, Figma and other design tools:** export the final PNG, JPG or MP4.
   - **A web page or direct URL:** save the file to your computer first.
2. Open the Sked Library: https://app.skedsocial.com/dashboard/library?tab=collections
3. Open the collection you want (or create one) and upload the files there.
4. Tell me when it's done and I'll find it and continue.

Offer to continue with any media that is already in the Library while the rest is uploaded.

---

## Step 4: Resolve the content pillar (only if the user named one)

Call `Sked_Social:list_content_labels` with the Brand Group ID. Filter to active labels and
match the pillar (or campaign) by name.

- Labels only apply when every target account belongs to the same Brand Group. If the accounts span groups, tell the user the pillar can't be applied and carry on without it.
- If the name doesn't match, show the available pillars by name and ask. Never invent a label ID.
- If the user did not mention a pillar, do not apply one.

---

## Step 5: Preview

Call `Sked_Social:preview_create_draft` with:

```
assetIds: [ordered asset IDs]
caption: <the caption, exactly as supplied>
accountIds: [resolved account IDs]
labelIds: [resolved label IDs, if any]
```

Always pass `accountIds` rather than platform codes so the result is never ambiguous.

Handle the result:

- **`ready`**: continue. Check `validation`: if any account shows `allowed: false`, explain the platform rule that failed in plain language and ask how to proceed.
- **`needs_selection`**: show the candidate accounts by name and ask which to use, then preview again with the chosen account IDs.
- **`invalid`**: read `diagnostics`. For `ACCOUNT_UNAVAILABLE`, name the account and explain it is disconnected or inactive, then offer to drop it, wait for a reconnect, or pick another account. For anything else, explain the message in plain language and follow its remediation. Never retry unchanged.

The preview token is valid for about 20 minutes. If it expires, preview again.

---

## Step 6: Commit

Call `Sked_Social:commit_create_draft` with the `previewToken` and an `idempotencyKey`.

**When to ask first.** Show the summary below and wait for a yes only if something was
ambiguous or you made a choice the user has not confirmed: a fuzzy account or media match,
a platform inferred from an account name, an account the user chose to keep despite a
connection warning, or a pillar you matched loosely. If the request was fully specified and
every match was exact, commit without asking.

```
Here's the draft I'll create for [Brand]:

Accounts: [account names]
Media: [file names, in order]
Caption: "[caption exactly as supplied]"
Pillar: [name or none]

Create it?
```

**Idempotency key.** Printable ASCII, no spaces, at most 200 characters. Build it from the date
plus the first 8 characters of the first account ID and first asset ID, for example
`draft-20261022-a2b2f92b-6ad97ee8`. If the user deliberately wants a second identical draft,
add a suffix such as `-2`. Retrying with the same inputs and key returns the original draft
instead of creating a duplicate, so it is safe to retry after an uncertain result.

If the commit reports a stale or expired token, preview again and commit with the new token.

---

## Step 7: Set the approval status (only when requested)

Do this step only if the user asked for a status ("mark it ready for review") or the source
data the user supplied states one. Otherwise leave the draft in its default status.

1. Call `Sked_Social:list_post_statuses`.
2. Match the requested status by name, ignoring case. If nothing matches, show the available status names and ask.
3. Call `Sked_Social:update_post` with the `postId` (the draft ID from Step 6) and `status` set to the matched status ID.
4. If the response includes `accountWarnings`, surface them.

---

## Step 8: Report

Confirm what was created, in a few lines:

> Draft created for **[Brand]** on [platforms] — [N] media item(s), caption saved as written[, pillar [name]][, status [status name]]. It's unscheduled. [Open draft](draftUrl)

Use the `draftUrl` from the commit result. Include the status name exactly as returned. Mention
any account warnings or accounts that were dropped.

**When invoked by `create-post`:** skip Steps 7 and 8 and hand control back, carrying forward
the draft ID, the resolved accounts (name and platform), the Brand Group ID, the caption, and
the media assets.

---

## Guardrails

- Never schedule or publish. This skill only creates unscheduled drafts.
- Never change the caption. Save it exactly as supplied, including line breaks, hashtags and emoji.
- Never guess a Library collection from a brand or Brand Group.
- Never fabricate asset, account, or label IDs. Resolve every ID from a tool result.
- Never proceed past a non-connected account without telling the user first.
- Never surface internal IDs, status IDs, tokens, or raw JSON to the user. Use names, file names and links.
- Never retry a rejected preview unchanged.
- The Sked MCP cannot upload media. If the media isn't in the Library, guide the user to upload it rather than trying another route.
