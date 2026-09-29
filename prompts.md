# Prompts from the video

Swap in your own products, cities, and dates. You never need to name a command. Claude reads `CLAUDE.md`, checks `bin/klaviyo --help`, and works out the rest.

Levels 1 and 2 only read data, so they work with a read-only key. Levels 3 and 4 need write access. Claude shows you what it's about to do and waits for your yes.

## Level 1: Questions Klaviyo wasn't built to answer

```
Who are my top 50 customers by lifetime spend within about 50 miles of Dallas? I need people I could invite to film a commercial.
```
```
Show me the definition of my main campaign segment in plain English.
```
```
Do my three biggest segments overlap? How many people are in more than one?
```
```
What flows are live right now, and what did each one make in the last 90 days?
```

## Level 2: The pulls the dashboard won't give you

```
Across the last year, which promo drove the most revenue? Break it down by campaign, revenue, and revenue per recipient so I can decide what kind of sale to run for Black Friday.
```
```
Break the list into engagement tiers, then for each tier give me share of send volume and share of revenue.
```
```
This flow made money last year and nothing this year. What happened to it?
```
When it gives you an answer, check it against what you know. If you merged that flow into another one, say so and have it re-run.

## Level 3: Build an audience by talking to it

```
Build me two audiences. One is everyone who has ever bought [your main product], not accessories, just the product. The other is people who have bought other things but never [your main product].
```
Read the definitions it shows you before you approve. Then change your mind:
```
Actually, make another one that includes all [broader category] buyers, not just that one brand.
```
```
Make me three audiences that don't overlap: buyers, engaged non-buyers from the last 90 days, and everyone else. I want to layer them on one campaign and see results by group.
```

## Level 4: Ship the campaign

```
Clone last month's best-performing campaign into a new draft.
```
```
Rewrite the subject line five ways, all under 40 characters.
```
```
Point the new draft at the [audience name] audience and schedule it for Tuesday at 10am Eastern.
```
Then open Klaviyo and look at the draft before it goes. Claude builds it; you send it.

## Level 5: Check who actually went through a flow

```
Pull the last few days of orders and tell me who went through the post-purchase flow. Did any [product you exclude] buyers get in?
```
```
Check the post-purchase trigger filter. Are all the [product] SKUs in there, including the one-time and subscription versions?
```

## Level 6: The hard question

```
Our email revenue keeps missing year over year, but the business is almost always up. Figure out why.
```
Push back when something doesn't match what you know about the business. It re-runs with your context.

## Before you schedule anything

```
Check this week's drafts before they go out: subject lines, from address, audiences, and whether the discount codes are live.
```
