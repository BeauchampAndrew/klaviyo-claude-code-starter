---
name: deploying-sunset-reactivation
description: Use when the user wants to set up a sunset flow, a suppression loop, or automatic re-activation of suppressed profiles in Klaviyo, e.g. "deploy a sunset flow", "set up the suppression loop", "auto-suppress people who never engage", "unsuppress people who come back to the site". Builds the two-flow pair (sunset suppresses chronically unengaged never-buyers via webhook; re-activation unsuppresses anyone who later shows activity) plus its segments and audit list.
---

# Sunset + Re-Activation Loop

Two flows that keep the list clean without losing people for good:

- **Sunset flow:** long-term unengaged people who never bought get one last-chance email, then get suppressed by a webhook.
- **Re-activation flow:** if a suppressed person later browses, adds to cart, or orders, a webhook unsuppresses them.

It works because **flow webhooks still fire for suppressed profiles, even though flow emails don't.** Site and Shopify tracking keep running while someone is suppressed, so the re-engaging segment can see them come back.

This is an advanced setup that changes who can receive email. Walk the user through every step and confirm before anything goes live.

```
Sunset segment (unengaged N days, never bought)
  -> Sunset flow: email -> wait 10d -> add to audit list -> webhook: SUPPRESS

... person later browses / carts / orders ...

Re-engaging segment (suppressed more than 30d ago, active in the last 30d)
  -> Re-activation flow: webhook: UNSUPPRESS
```

## Defaults (adjust per account)

| Setting | Default | Why |
|---|---|---|
| Unengaged window | 270 days | no opens, clicks, or site activity |
| Minimum emails received | 10, all time | they had a fair shot |
| Minimum profile age | 270 days | matches the window |
| Wait after last-chance email | 10 days | time to engage and exit |
| Re-activation cooldown | 30 days | prevents flip-flopping |
| Re-engagement window | 30 days | orders, carts, checkouts, product views |

Buyers never get sunset. They belong in a winback flow, not suppression.

## Prerequisites

1. **A dedicated API key with `subscriptions:write` only.** The key sits in plain text in the webhook config, so give it nothing else. The user creates it in Klaviyo: Settings > API Keys > Create Private API Key > Custom, with Subscriptions set to Full Access and everything else set to No Access. It must be Subscriptions, not Profiles; suppression lives under subscriptions, and a Profiles key gets a 403. This key goes only into the flow webhook headers, never into `.env` and never into chat.
2. **Metric IDs for this account.**
   ```bash
   bin/klaviyo list-metrics
   ```
   You need: Received Email, Opened Email, Clicked Email, Active on Site, Placed Order, every Added to Cart, Checkout Started, Viewed Product, Unsubscribed from Email Marketing, and Manually Suppressed from Email Marketing. That last one only exists once something has been suppressed in the account. If it's missing, deploy sunset first; the re-engaging segment can be built after the first suppression.

## Step 1: Sunset segment

Copy `examples/segments/sunset-unengaged-never-bought.json` to `work/segments/`, fill in the metric IDs, explain it in plain English, get a yes, then:
```bash
bin/klaviyo create-segment --name "Sunset - Unengaged 270d Never Bought" --body @work/segments/sunset.json
bin/klaviyo segment-count <id>
```

## Step 2: Re-engaging segment

Copy `examples/segments/suppressed-reengaging-30d.json`, fill in the IDs. How it reads:
1. Has been manually suppressed at some point,
2. but not in the last 30 days (the cooldown),
3. and **currently** can't receive email marketing (without this you'd match people already reactivated),
4. and showed activity in the last 30 days: ordered, added to cart, started checkout, or viewed a product (these are OR'd; include every instance of each metric),
5. and has never unsubscribed. **Unsubscribes are consent, not suppression. Never touch them.**

```bash
bin/klaviyo create-segment --name "Suppressed - Re-Engaging 30d" --body @work/segments/reengaging.json
```

## Step 3: Audit list

A list named `ADDED TO SUPPRESSION LIST` that the sunset flow adds people to right before suppressing them. It's the record of who the flow suppressed.
```bash
bin/klaviyo api POST /api/lists/ --body '{"data":{"type":"list","attributes":{"name":"ADDED TO SUPPRESSION LIST"}}}'
```

## Step 4: Flows

Flows can be created through the API, webhooks included. They're created as **drafts**. When creating, actions use `temporary_id`, `entry_action_id` points at one, and `links.next` chains the steps.

**Re-activation flow** (webhook only). Write the body to `work/reactivation-flow.json`, with the user pasting their subscriptions key in place of `<SUBSCRIPTIONS_KEY>` themselves:

```json
{"data":{"type":"flow","attributes":{"name":"Re-Activation Flow","definition":{
  "triggers":[{"type":"segment","id":"<REENGAGING_SEGMENT_ID>"}],
  "profile_filter":null,
  "entry_action_id":"1",
  "reentry_criteria":{"duration":0,"unit":"alltime"},
  "actions":[{"temporary_id":"1","type":"send-webhook","data":{"message":{
    "url":"https://a.klaviyo.com/api/profile-suppression-bulk-delete-jobs",
    "headers":{"Authorization":"Klaviyo-API-Key <SUBSCRIPTIONS_KEY>","Content-Type":"application/json","revision":"2026-01-15"},
    "body":"{\"data\":{\"type\":\"profile-suppression-bulk-delete-job\",\"attributes\":{\"profiles\":{\"data\":[{\"type\":\"profile\",\"attributes\":{\"email\":\"{{ person.email }}\"}}]}}}}",
    "name":"Re-Activation Webhook"},
    "status":"draft"},
    "links":{"next":null}}]
}}}}
```
```bash
bin/klaviyo api POST /api/flows/ --body @work/reactivation-flow.json
```
Delete `work/reactivation-flow.json` afterwards; it holds a key.

**Sunset flow.** Its email step needs an existing template, so building it in the Klaviyo UI is often just as fast:
- **Trigger:** the sunset segment. Re-entry: once ever.
- **Flow filter** (so anyone who engages mid-flow drops out before suppression): Opened Email = 0 since starting this flow, AND Clicked Email = 0 since starting this flow, AND Active on Site = 0 since starting this flow, AND Placed Order = 0 over all time.
- **Steps:**
  1. Email: the last-chance send ("Should we stop emailing you?"). Smart Sending off.
  2. Time delay: 10 days.
  3. Update list: add to `ADDED TO SUPPRESSION LIST`.
  4. Webhook: the **suppress** version (below).

**Re-activation flow notes:** re-entry allowed (someone can be sunset, come back, and be sunset again later). **The webhook must be the first step.** Never put it after an email; suppressed people never receive the email, so they'd never reach the webhook.

### Webhook settings

| | Suppress (sunset) | Unsuppress (re-activation) |
|---|---|---|
| URL | `https://a.klaviyo.com/api/profile-suppression-bulk-create-jobs` | `https://a.klaviyo.com/api/profile-suppression-bulk-delete-jobs` |
| Body `type` | `profile-suppression-bulk-create-job` | `profile-suppression-bulk-delete-job` |

Headers (both): `Authorization: Klaviyo-API-Key <subscriptions key>`, `Content-Type: application/json`, `revision: 2026-01-15`.

Body (swap the `type`):
```json
{"data":{"type":"profile-suppression-bulk-create-job","attributes":{"profiles":{"data":[
  {"type":"profile","attributes":{"email":"{{ person.email }}"}}
]}}}}
```

## Step 5: Verify, then go live

1. **Key test.** Have the user send the exact webhook body with their key to the endpoint for a test address they own (a curl they run themselves). `202 Accepted` means the scope is right. `403 insufficient_permissions` means wrong scope, not a bad body.
2. **Counts.** `segment-count` on both segments. Sunset should be sizable; re-engaging starts near zero.
3. **Order matters.** Turn on **re-activation first**, then sunset. Never run sunset without the way back.
4. **Afterwards**, see who got unsuppressed:
   ```bash
   bin/klaviyo list-metrics --search suppress
   bin/klaviyo events --metric <manually_unsuppressed_id> --limit 20
   bin/klaviyo suppression-jobs
   ```

## Gotchas

- Unsuppress only clears manual/API suppressions. It can't clear hard bounces or spam complaints, and it never resubscribes anyone.
- Keep the 30-day cooldown. Without it the two flows can flip a profile back and forth.
- A segment created seconds after its data landed can miss those profiles until a new event re-evaluates them. When testing with seeded data: seed, wait a few minutes, then create the segment.
- `get-segment` summarizes consent conditions loosely. Check consent logic in the raw JSON: `bin/klaviyo api GET "/api/segments/<id>/?fields[segment]=name,definition"`.
