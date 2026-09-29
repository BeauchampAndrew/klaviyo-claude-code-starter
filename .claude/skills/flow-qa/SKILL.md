---
name: flow-qa
description: Use when the user wants to check whether a Klaviyo flow is working as intended, e.g. "who went through the post-purchase flow?", "did any trial buyers get into this flow?", "is my exclusion filter working?", "why did this flow's revenue drop?", "check the trigger filter has all the SKUs now", "did anything in this flow get switched off?". Compares who should have entered against who actually got the emails, and audits triggers, filters, and message status.
---

# Flow QA

A flow can look perfect in the builder and still be wrong. The only real test is who actually went through it. This skill compares **who should have entered** with **who actually received the emails**, then traces any difference back to the trigger, the filters, or a message that got switched off.

## 1. Read the flow

```bash
bin/klaviyo flows --help                  # search/sort options
bin/klaviyo flows                         # find the flow ID
bin/klaviyo get-flow <flow_id> --definition   # branch tree: splits, messages, delays
bin/klaviyo flow-detail <flow_id>         # trigger, trigger filters, profile filters, emails
bin/klaviyo flow-actions <flow_id>        # each action's live/draft status + last updated
```

Write down in plain English:
- **Trigger:** which metric, and **which instance** (a Shopify "Placed Order" vs an API "Placed Order" are different metrics; a flow on one never sees orders that only land on the other).
- **Trigger filters:** event-level conditions (e.g. "Items does not contain SKU-A, SKU-B"). These decide who gets in.
- **Flow (profile) filters:** checked at entry and before every step. They can drop people mid-flow.
- **Status:** any message in draft or manual mode? Any recent `updated` timestamps that line up with a drop?

## 2. Decide what "correct" means

Get the rule from the user in one sentence, e.g. "Everyone who places an order gets this, except trial kit buyers, who get their own flow." Then list the specifics the rule depends on. For SKU rules, pull the real list:
```bash
bin/klaviyo list-metrics --search "ordered product"
bin/klaviyo events --metric <ordered_product_id> --properties --since YYYY-MM-DD --limit 200
```
Look at every variant of the product: one-time vs subscription (`-AS`, `-SUB`), bundle, sample, regional. **Missing variants are the most common bug.**

## 3. Who should have entered

Export the trigger events for a window that's long enough to see a pattern (3 to 10 days is usually plenty):
```bash
bin/klaviyo export-events --metric <trigger_metric_id> --since YYYY-MM-DD --out work/trigger.ndjson --fields all
```
Apply the intended rule to each event (in Python or jq) and get two sets of profiles: **should enter** and **should be kept out**.

## 4. Who actually entered

Flow emails log a "Received Email" event with the flow and message identifiers in the payload. Check one event first to confirm the field names (flow sends usually carry `$flow` and `$message`; campaign sends carry `$campaign`):
```bash
bin/klaviyo list-metrics --search "received email"
bin/klaviyo events --metric <received_email_id> --properties --limit 5
bin/klaviyo export-events --metric <received_email_id> --since YYYY-MM-DD --out work/received.ndjson --fields all
```
Filter to this flow's ID. The first message is the entry point; use it for the "who entered" set.

Account for the flow's delay. If the first email goes out 3 days after the trigger, the last 3 days of triggers haven't received anything yet.

## 5. Compare and classify

| | Received the flow | Didn't receive it |
|---|---|---|
| **Should enter** | Working | Missed (trigger metric? filter too tight? message off? unsubscribed or suppressed?) |
| **Should be kept out** | **Leak** | Working |

For each leak, show the event that let them in (the SKUs, the property values) and the filter condition it slipped past. Group leaks by cause; there's usually one.

**Before calling anything a failure, check it against the rule the user gave you.** People who were supposed to be excluded and were excluded are the filter working. Don't count them as broken. Also rule out ordinary reasons someone didn't get an email: they were unsubscribed, suppressed, already in the flow, or blocked by smart sending.

## 6. Report

- The one-line answer: "Yes, 62 trial buyers got in over 3 days."
- The cause: "The exclusion lists the 6 subscription SKUs (`-AS`) but not the 6 one-time versions."
- The size: rate per day, and share of trigger volume.
- The fix, in exact UI terms.

**The API can't edit a live flow's trigger or filters.** The user makes the change in Klaviyo. Give them the exact values to add (copy-pasteable). Once they've done it, re-run `flow-detail` and confirm the filter matches, e.g. "All 12 SKUs are in there now."

## Other checks

- **Revenue dropped on a flow.** Plot it before theorizing:
  ```bash
  bin/klaviyo flow-series <flow_id> --interval weekly --since YYYY-MM-DD --per-message
  ```
  A message going to zero on a specific week, with `flow-actions` showing an update that day, is a switched-off branch. A flow reading $0 this year may have been merged into another flow. **Ask before calling it dead.**
- **Replacing a flow.** Turning a flow off kills everyone's remaining emails mid-sequence. Block new entries with an impossible trigger condition, launch the replacement, and turn the old one off after its longest path has elapsed.

## Rules

- Read-only by default. Don't change flows through the API without asking; say what should change and let the user do it in the UI.
- Show your window, your metric IDs, and how you handled the delay, so the numbers can be checked.
- Keep exports in `work/`; they contain customer emails.
