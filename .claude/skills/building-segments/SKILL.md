---
name: building-segments
description: Use when the user wants to build, change, or understand a Klaviyo segment or audience by describing it, e.g. "build me an audience of everyone who bought X but not Y", "make a segment of engaged non-buyers", "change that to include all grill buyers", "explain this segment in plain English", "do these segments overlap?". Covers resolving metric IDs, drafting the definition, getting approval, creating it, counting it, and iterating.
---

# Building Segments

The user describes an audience in English. You turn it into a Klaviyo segment definition, explain it back in plain English, get a yes, create it, and report the count. Then they'll change their mind, and that's the point. Make the second and third versions as fast as the first.

## 1. Understand the ask

Pin down, from what they said (ask only if it's genuinely ambiguous):
- **Who's in:** the behavior or property (bought X, opened in last 30 days, lives in Texas).
- **Who's out:** exclusions (already bought, unsubscribed, recently emailed).
- **Time windows:** "ever" vs "last N days". Default to what the words imply and state it.
- **What it's for:** a campaign audience, a flow trigger, or a one-off count. This changes whether it needs to be mutually exclusive with other segments.

## 2. Check what already exists

```bash
bin/klaviyo search-segments "<keyword>"
bin/klaviyo get-segment <id>          # definition + count, metric IDs resolved
```

If something close exists, show it and ask whether to reuse, copy, or build new. `create-segment` blocks duplicate names unless you pass `--force`.

## 3. Resolve metrics and product data

```bash
bin/klaviyo list-metrics --search "placed order"
bin/klaviyo list-metrics --search "ordered product"
```

- Accounts often have duplicate metrics (Shopify and API). For "ever bought", use the one that holds the real order history. For engagement, include every instance in an OR group.
- For product-level conditions ("bought an actual smoker, not accessories"), look at real event payloads first to see which property holds what you need (`ProductName`, `Categories`, `SKU`, `Brand`, `ItemNames`):
  ```bash
  bin/klaviyo events --metric <ordered_product_id> --properties --limit 20
  ```
  Pick the property that actually separates what they asked for. Categories are usually cleaner than product-name matching. Show them the values you plan to match on.

## 4. Draft the definition

Write it to `work/segments/<slug>.json`. Shape:

```json
{
  "condition_groups": [
    {"conditions": [ ... ]},
    {"conditions": [ ... ]}
  ]
}
```

- **Groups are AND'd. Conditions inside one group are OR'd.**
- Common condition types: `profile-metric` (did an event N times in a window, optionally filtered by event properties), `profile-property`, `profile-marketing-consent`, `profile-region`, `profile-predictive-analytics`.
- Product filter on a metric condition:
  ```json
  "metric_filters": [{"property": "Categories", "filter": {"type": "list", "operator": "contains-any", "value": ["Smokers"]}}]
  ```
- Starter definitions with `<METRIC_ID>` placeholders are in `examples/segments/`. Copy one, then swap in real IDs.

## 5. Explain it back, then get a yes

Plain English, one line per group, no JSON unless they ask:

> **Smoker buyers (all time)**
> - Placed an order containing a product in the "Smokers" category, ever (Shopify Placed Order)
> - Can receive email marketing

Say what you assumed (e.g. "I used the Shopify order metric; the API one only goes back to March"). Wait for approval.

## 6. Create and count

```bash
bin/klaviyo create-segment --name "<Name>" --body @work/segments/<slug>.json
bin/klaviyo segment-count <new_id>
```

The count can read low for a few minutes while Klaviyo builds the segment. If it looks wrong, wait and recount before changing the definition. Report the count and a sanity check: is it plausible against the list size or the number of buyers?

## 7. Iterate

When they change their mind ("actually, include all grill buyers"), copy the JSON to a new file, change only what they asked, show the diff in plain English, confirm, create, count. Keep earlier versions unless they ask you to change one in place.

## Common follow-ups

- **"Do these overlap?"** Pull members of each (`bin/klaviyo --json segment-members <id> --limit N`, saved into `work/`), intersect by profile ID, and report counts: only A, only B, both. For big segments, say how many profiles you sampled.
- **"Make these mutually exclusive"** (for per-group reporting on one campaign): layer them in priority order. Each later segment excludes the earlier ones: Buyers, then Engaged non-buyers (not in Buyers), then Everyone else (in neither).
- **"Who are my top customers near <city>?"** Use `profile-region` or location properties if they're populated; otherwise pull profiles and filter on `location` fields. Say how many profiles had no location data.

## Rules

- Never create without explicit approval of the plain-English definition.
- Never edit or overwrite an existing segment that campaigns or flows use without asking. Build a new one instead.
- State which metric ID you used whenever there's more than one candidate.
- Names: follow the account's existing naming pattern if it has one (check `list-audiences`).
