---
name: performance-diagnosis
description: Use when the user asks a performance question the Klaviyo dashboard can't answer directly, e.g. "which promo drove the most revenue last year?", "what sale should we run for Black Friday?", "break the list into engagement tiers and show share of sends vs share of revenue", "why is email revenue down when the business is up?", "why did click rate drop?", "is our campaign audience too broad?". Pulls campaign, flow, and event data over long windows, decomposes the change, tests its own theories, and writes up the answer.
---

# Performance Diagnosis

Answer "what worked?" and "why did this change?" with numbers, not a feeling. The value here is breaking a headline number into its parts and checking each part against the raw data. That's also where it's easy to be confidently wrong, so every theory gets tested before it goes in the write-up.

Write findings to `work/reports/YYYY-MM-DD-<topic>.md`. Keep the pulled data next to it so the numbers can be rechecked.

## Data you'll use

```bash
bin/klaviyo campaign-performance --since YYYY-MM-DD --until YYYY-MM-DD --detail --sort revenue
bin/klaviyo flow-performance --days 90
bin/klaviyo flow-series <flow_id> --interval monthly --since YYYY-MM-DD
bin/klaviyo metric-aggregate <metric_id> --measurements count,unique,sum_value --interval month --since YYYY-MM-DD
bin/klaviyo metric-aggregate <metric_id> --by <field> ...   # group by an event field; a rejected field makes the API list the valid ones
bin/klaviyo export-events --metric <metric_id> --since ... --until ... --out work/<name>.ndjson
```

- **Rate limits.** The campaign and flow reporting endpoints allow about 2 calls a minute. For long windows, pull month by month in a background loop with ~32 seconds between calls, and cache each month to a file so a rerun only fetches what's missing.
- **Which order metric?** `campaign-performance` auto-picks Placed Order. If the account has more than one (Shopify + API), or changed platforms, older revenue may live on another metric. Check with `list-metrics --search "placed order"` and say which you used.

## Recipes

### Best promos (e.g. for Black Friday planning)
1. Pull 12 months of campaigns with `--detail`.
2. Group campaigns into promos (same sale, several sends) by name, dates, and subject lines. Show the grouping and ask if it's right. Only the user knows which sends belonged to which sale.
3. For each promo: total revenue, number of sends, recipients, **revenue per recipient**, and the offer (pull it from the creative with `get-creative <id> --grep "%"` or `--grep code`).
4. Rank by revenue per recipient as well as total revenue. A big number from mailing everyone five times isn't the same as a strong offer.

### Engagement tiers: share of sends vs share of revenue
1. Define tiers by days since the last open or click (e.g. 0-30, 31-60, 61-90, 91-180, 180+). If the account's open data is inflated by Apple Mail Privacy Protection, weigh clicks and site activity more heavily, and say so.
2. Export Received Email, Opened Email, Clicked Email, and the order metric for the window. Join them by profile.
3. For each tier: share of sends, open and click rate, and share of attributed revenue.
4. The headline is usually "the X+ day tail is N% of sends and M% of revenue." Make sure the tier definitions are stated right next to it.

### "Why is email revenue down when the business is up?"
Break email revenue into **orders x average order value** before looking at anything else.
1. **Same-period comparison** (e.g. same quarter, year over year): email-attributed orders, email average order value, email revenue, total store orders, total store revenue.
2. **If orders are up and order value is down,** email isn't losing customers. Look at what changed per order: cheaper entry products (trial kits, samples), add-ons, discount depth, the product mix.
3. **Check what "the business" includes.** Pull raw order events with `--properties` and look at `source_name`, channel, or store fields. Marketplace orders (Amazon through a connector), wholesale, and POS can flow into the same order metric and make "the business" look like it's growing when the website isn't. Recompute email's share of **website** revenue.
4. **Check for revenue email can't be credited for.** Subscription renewals, auto-ship, and anything outside the attribution window grow the business without email orders. If the brand changed its default (e.g. more new customers starting on subscription), measure the before and after.
5. **Test each theory against the data and drop the ones that don't hold.** Example: "customers moved to Amazon." Check how many website customers actually show up in marketplace orders. If it's a handful, say so and move on.

### Click rate or engagement dropped
1. Monthly campaign aggregates for 12 to 24 months: recipients, open rate, click rate, click-to-open.
2. Check audience size first. If recipients per send grew (a broader audience, a list import), rates fall even with identical content. Plot rate against recipients.
3. Then content: tag a sample of the best and worst campaigns by theme, offer, and format (render them with `get-creative --html`), and compare click rates by tag.

## Rules for the write-up

- **Lead with the answer**, then the table that proves it, then how you got there.
- **Every number gets its window and metric.** "Q3 (Jul 1 to Sep 24), Shopify Placed Order, email-attributed."
- **Separate what the data shows from what you're inferring.** When a conclusion depends on business context you don't have ("this flow went to $0, so it was retired"), ask before writing it down. Merges, platform changes, and pricing changes all look like failures in the data.
- **Say what you ruled out**, with the number that ruled it out.
- When the user pushes back with context, re-run with their version of the question and update the write-up. Don't defend the first answer.
