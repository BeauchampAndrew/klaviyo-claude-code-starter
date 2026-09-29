---
name: scheduling-campaigns
description: Use when the user wants to schedule, clone, or prep Klaviyo email or SMS campaigns, e.g. "schedule this week's campaigns", "clone last month's best campaign into a new draft", "point this draft at the buyers segment and schedule it for Tuesday 10am", "check my drafts before they go out", "upload this SMS". Reconciles the plan against the drafts in Klaviyo, gates on subject line, from address, and discount codes, confirms, then patches, schedules, and verifies.
---

# Scheduling Campaigns

Take a send plan (dates, times, audiences), line it up against the drafts in Klaviyo, fix what's off, get a yes, schedule, and verify. Most scheduling mistakes come from clones: the draft carries the previous email's subject, sender, and audiences, and only the name was updated.

## 1. Get the plan

The plan can come from:
- The user's message ("the 3 drafts for this week, 10am Eastern, engaged + buyers, exclude recent purchasers")
- A CSV or spreadsheet export they drop into the folder (read it; the columns usually include name, send date, send time, included audiences, excluded audiences, notes)
- Nothing yet. Then ask for date, time, timezone, and audiences for each send.

## 2. Pull the drafts

```bash
bin/klaviyo list-drafts
bin/klaviyo get-campaign <id> --names      # subject, preview, send time, audiences with names
bin/klaviyo list-audiences                 # name <-> ID mapping
```

Match each planned send to a draft by name. Flag drafts with no plan and planned sends with no draft.

## 3. Check each draft (hard gates)

**Subject and preview.** Read them against the campaign's topic. Stop and confirm if the subject:
- reads like a different email,
- is a placeholder or empty ("Subject line", "TBD", "Test"), or
- matches a recent or adjacent send (the clone fingerprint).

**From address.** Check the sender matches the brand's current sending domain. Clones of old campaigns can carry a dead address that Gmail rejects outright.
```bash
bin/klaviyo api GET /api/campaigns/<id>/campaign-messages/
```

**Discount codes.** If the email has a code:
```bash
bin/klaviyo get-creative <id> --grep code
```
A code in the email doesn't prove it works. Confirm it's live, in this order of preference:
1. It shows up in recent real orders: `bin/klaviyo events --metric <placed_order_id> --properties --limit 50`, then look for it in the discount fields.
2. The user has tested it at checkout, or confirms it's created, active, and valid through the last send that uses it.
A `/discount/CODE` link that redirects is not proof. Shopify redirects invalid codes too. Check the end date covers the final send, especially after a sale gets extended.

**Audiences.** If the draft's audiences don't match the plan, assume the clone brought them along. Plan to set them to match the plan and tell the user what changed. Only ask if the plan itself is unclear.

## 4. Timezone

Times in a plan usually have no timezone. Infer it from recent sent campaigns (`bin/klaviyo list-campaigns --status Sent --limit 10`, compare send times) and confirm with the user. Don't guess.

## 5. Confirm

Show one table and get an explicit yes:

| Campaign | Send (local, TZ) | Subject / Preview | Include | Exclude | Code check |
|---|---|---|---|---|---|

Mention that Klaviyo's UI will show the send time in UTC. That's display only.

Ask reconciliation questions one at a time, not stacked.

## 6. Execute

```bash
bin/klaviyo patch-campaign <id> --date MM-DD-YYYY --time "10:00 AM EDT" \
  --include ID1,ID2 --exclude ID3,ID4
bin/klaviyo patch-message <id> --subject "..." --preview "..."   # only if changing copy fields
bin/klaviyo schedule <id>
bin/klaviyo get-campaign <id> --names                            # verify
```

- **Always pass `--include` and `--exclude` together** when changing audiences. Either one alone either errors or silently wipes the other. `--exclude ""` clears exclusions on purpose.
- Pass the time with its timezone (`EDT`, `CST`, `PDT`...). Don't convert to UTC yourself.
- After scheduling, confirm status is Scheduled and the send time is right. A campaign can briefly show "Adding Recipients" or "Queued without Recipients" first. That's normal.

### Cloning

```bash
bin/klaviyo clone-campaign <source_id> --name "New name"
```
It prints what the clone inherited (subject, from address, audiences). Treat all of it as wrong until checked against the plan.

### Replacing the HTML

```bash
bin/klaviyo set-campaign-html <id> --html work/email.html --expect "new headline" --forbid "old headline"
```

### SMS

```bash
bin/klaviyo upload-sms --name "SMS [MM-DD-YYYY] Name" --body @work/sms.txt \
  --date MM-DD-YYYY --time "2:00 PM EDT" --include SMS_SEGMENT_ID [--exclude ID]
bin/klaviyo schedule <new_id>
```
- Strip any "Reply STOP to opt out" line. Klaviyo appends it.
- Resolve placeholder codes (`[CODE]`, `[TBD]`) to the real code from the matching email.
- Links should use the canonical domain and go to a specific page, not the homepage.
- Use SMS audiences, not email ones. When there's a previous send for the same promo, copy its audience setup.
- Send between 8am and 9pm recipient time.

## Changing something already scheduled

Cancelling is permanent; a cancelled campaign can't be edited. Clone the scheduled one into a fresh draft, fix the draft, schedule it, and tell the user to cancel the original in Klaviyo. Before touching any scheduled send, confirm which campaign they mean by name and send time.

## Rules

- Never schedule without an explicit yes on the final table.
- Configure audiences exactly as the plan says. Don't add exclusions or "best practice" audiences nobody asked for; suggest them instead.
- If an audience ID comes back "not found" but the segment exists, it's deactivated. Ask the user to reactivate it.
- Report at the end: what was scheduled, what was skipped, and why.
