# Klaviyo + Claude Code

This folder connects Claude Code to one Klaviyo account. The person you're working with runs email (and maybe SMS) for an eCommerce brand. They probably don't write code, so explain what you're doing in plain English and don't make them read JSON unless they ask.

## Klaviyo

Use `bin/klaviyo` for Klaviyo data and actions (campaigns, segments, flows, metrics, profiles). It's [klaviyo-cli](https://github.com/BeauchampAndrew/klaviyo-cli), run through `uvx` with the API key loaded from `.env`. Pass `--json` before the command for machine-readable output (e.g. `bin/klaviyo --json list-campaigns --days 30`). Run `bin/klaviyo --help` for the full command list and `bin/klaviyo <command> --help` for options.

- Always call it as `bin/klaviyo` from this folder, exactly like that. That's what the read-only permissions in `.claude/settings.json` match on.
- Use `bin/klaviyo api <METHOD> <path>` only when no typed command covers what you need.
- Deleting profiles, lists, segments, campaigns, and flows is blocked inside the CLI. You can't do it, even through `api`.

## First-run setup

When someone says "set this up", "get me started", or anything like it on a fresh clone, walk them through this. Do each step, check it worked, then move on.

1. **uv.** Run `command -v uvx`. If it's missing, tell them it's the tool that runs klaviyo-cli without installing Python packages, and offer to install it: `curl -LsSf https://astral.sh/uv/install.sh | sh`. After it installs, they may need to open a new terminal.
2. **The API key.** Check `test -f .env`. If `.env` doesn't exist, run `cp .env.example .env`, then tell them to open `.env` and paste their key over `pk_your_key_here`.
   - They create the key in Klaviyo under **Settings > API Keys > Create Private API Key**. Recommend **Read-only** for the first week. Add write scopes (Campaigns, Segments, Lists, Profiles, Subscriptions) when they want you to build and schedule things.
   - **Never ask them to paste the key into chat, and never read, print, or echo `.env`.** They edit the file themselves.
3. **Test the connection.** Run `bin/klaviyo account-health`. You should see profiles, lists, and metrics.
   - "No Klaviyo API key yet" means `.env` is missing or still has the placeholder.
   - A 401 means the key is wrong or got cut off when pasting. A 403 on a specific command means that key doesn't have that scope.
4. **Hand off.** Tell them it's working and in one sentence what's in the account (e.g. "You've got 41 lists, 63 segments, and Shopify order data."). Then point them to `prompts.md` and suggest one Level 1 question to try first.

## How to work

- **Read before you write.** Pull the current state (`get-campaign`, `get-segment`, `flow-detail`) before changing anything, and verify it again after.
- **Confirm every write.** Before any command that changes the account (`create-segment`, `create-flow`, `patch-campaign`, `patch-message`, `clone-campaign`, `set-campaign-html`, `schedule`, `upload-sms`, `upload-image`, `push-event`, `suppress`, `unsuppress`, `unsubscribe`, or `api` with POST/PATCH), show exactly what will change and get a yes.
- **Nothing sends without a human.** Build drafts freely once approved. Scheduling needs an explicit yes on a final table that shows date, time with timezone, subject, preview, and audiences.
- **Show segment definitions in plain English before creating them.** The user approves the logic, not the JSON.
- **Metric IDs are different in every account.** Resolve them with `bin/klaviyo list-metrics --search "<name>"`. Accounts often have duplicates (e.g. a Shopify "Placed Order" and an API "Placed Order"). Say which one you used and why.
- **Say when you're inferring.** The data can be right while the conclusion is wrong. Example: a flow showing $0 this year may have been merged into another flow, not retired. Ask about business context (a flow merge, a platform switch, a pricing change) before you call something broken or dead.
- **Save working files in `work/`** (gitignored). Exports can contain customer emails and order data, so they must never land anywhere that gets committed.
- **Rate limits are real.** Reporting endpoints (`campaign-performance`, `flow-series`, `flow-performance`) allow about 2 calls a minute. Space those calls out and honor "retry after N seconds". `segment-count` is 1 call per second.

Read `docs/klaviyo-gotchas.md` before scheduling, cloning, suppressing, or editing flows. Every item there is a mistake someone already made.

## Skills in this folder

| Skill | Use it for |
|---|---|
| `building-segments` | Building an audience by describing it, then iterating on it |
| `scheduling-campaigns` | Reconciling a send plan against drafts, then patching and scheduling |
| `flow-qa` | Checking who actually went through a flow, and whether its filters work |
| `performance-diagnosis` | Promo rankings, engagement tiers, and "why is email down when the business is up?" |
| `deploying-sunset-reactivation` | The sunset + re-activation suppression loop (advanced) |

`examples/segments/` has starter segment definitions with `<METRIC_ID>` placeholders. `prompts.md` has the prompts from the walkthrough video, organized by level.
