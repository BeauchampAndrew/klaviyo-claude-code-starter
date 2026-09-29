# Klaviyo Gotchas

Things that look fine and aren't. Each of these cost a real send, a real cleanup, or a real afternoon. Claude reads this before scheduling, cloning, suppressing, or touching flows.

## Campaigns

**Clones carry everything forward, not just the design.** A duplicated campaign keeps the source's subject line, preview text, from address, and audiences. Usually only the name gets updated. Before scheduling any draft, check all four: `bin/klaviyo get-campaign <id> --names` shows subject, preview, and audiences, and `bin/klaviyo api GET /api/campaigns/<id>/campaign-messages/` shows the from address.
- A wrong subject is easy to spot once you look: it reads like a different email, it's a placeholder ("Subject line", "TBD"), or it matches last week's send.
- A stale from address is worse. If it's on an old domain that doesn't match the account's authenticated sending domain, Gmail rejects it outright (550 errors, bounce rate spikes). Confirm the from address matches the brand's current sending domain.
- If the draft's audiences don't match the plan, assume the clone brought them along. Fix the draft to match the plan and say what you changed.

**`patch-campaign` replaces the whole audience, it doesn't merge.** Pass `--include` and `--exclude` together, every time, with the complete lists you want.
- `--exclude` alone errors out ("At least one audience must be included").
- `--include` alone silently wipes the existing exclusions. There's no error.
- `--exclude ""` clears exclusions on purpose.
- Re-check with `get-campaign --names` after every patch.

**Cancelling a scheduled campaign is permanent.** A cancelled campaign can't be edited or rescheduled. To change a scheduled send, clone it into a fresh draft, fix that, and schedule the new one. The cancelled one is harmless and stays in the list.

**"Audience not found" usually means the segment is deactivated.** If `patch-campaign` says an ID doesn't exist but `bin/klaviyo api GET /api/segments/<id>/` returns it, the segment is archived. Ask the user to reactivate it in Klaviyo. Don't work around it.

**Klaviyo's UI shows API-scheduled times in UTC.** 10am Eastern shows as 2pm. The send goes out at the right local time; only the display is off.

**Editing a cloned campaign's HTML: use `set-campaign-html`.** Cloned campaigns get a campaign-scoped template that rejects direct edits, and assigning a template snapshots its HTML. `set-campaign-html --expect "new text" --forbid "old text"` handles both problems and confirms the live creative actually changed.

**A code in the email doesn't prove the code works.** Before scheduling a promo, confirm the discount code is active in Shopify and valid through the last send that uses it. The strongest proof is a real redemption in recent Placed Order events (`bin/klaviyo events --metric <placed_order_id> --properties`). A `/discount/CODE` link that redirects doesn't prove anything; Shopify redirects invalid codes too.

**Want results by audience group? Use one campaign, not several.** Build mutually exclusive segments (each person lands in exactly one), include all of them on one campaign, and use Klaviyo's audience breakdown report. Splitting into one campaign per group multiplies the work and fragments the send.

## Things that look broken but aren't

- `#insertUrlLink` and `manage.kmail-lists.com/subscriptions/placeholder` in the footer are placeholders Klaviyo fills in at send time.
- `ctrk.klclick1.com/...` links are Klaviyo's click tracking. To see where one goes: `curl -s -o /dev/null -w "%{redirect_url}" <url>`.
- The plaintext version showing raw destination URLs is normal. Klaviyo generates it and unwraps the trackers.

## Segments

**New segments lag.** Membership can take minutes to hours to settle after creation or a bulk change. A segment created seconds after its data landed can miss those profiles until a new event re-evaluates them. Wait, then count.

**Metric IDs are per-account, and duplicates are common.** Many stores have two "Placed Order" metrics (Shopify and API), two "Added to Cart" metrics, and so on. For engagement, include every instance. For revenue, pick one on purpose and say which.

**Check what the order metric actually contains.** A Shopify Placed Order metric can include marketplace orders (Amazon via a connector, wholesale, POS). Pull a few raw events with `--properties` and look at fields like `source_name` before treating the metric as "website orders".

**Order metrics change when a store changes platforms.** If a store moved from WooCommerce or a custom API to Shopify, older revenue lives on a different metric. `campaign-performance` picks one automatically and will show $0 for campaigns sent before the switch.

**`get-segment` summarizes consent conditions loosely.** For consent logic, trust the raw JSON: `bin/klaviyo api GET "/api/segments/<id>/?fields[segment]=name,definition"`.

## Flows

**Turning a flow off stops everyone mid-flow.** People already in it don't finish the series. To replace a flow: add an impossible condition to the old flow's trigger filter so nobody new enters, launch the new flow, and turn the old one off only after the longest path has finished.

**The API can't edit a live flow's trigger or filters.** The user changes those in the Klaviyo UI. You can read them before and after (`flow-detail`) to confirm the change is right.

**A flow triggered on one order metric misses orders on the other.** If orders come in through both a Shopify metric and an API metric, a flow triggered on the Shopify metric never sees the API-only orders. Check which metric the trigger uses before trusting coverage.

**The flow looking fine in the UI doesn't mean it's working.** Check who actually went through it (see the `flow-qa` skill). SKU lists are the classic miss: an exclusion that lists the subscription SKUs (`-AS`, `-SUB`) but not the one-time versions of the same product.

## Suppression and consent

**Suppression is reversible; unsubscribing isn't.** `suppress` stops marketing email and `unsuppress` undoes it. `unsubscribe` revokes consent, and `unsuppress` can't bring it back. Both need `--yes` or an interactive confirmation. Never run either on real profiles as an experiment. Propose the exact command and wait.

**`suppress --file` doesn't split big files.** Klaviyo caps suppression jobs at 100 profiles. A larger file fails and nothing gets suppressed. Split it first and loop:

```bash
mkdir -p work/chunks work/done
split -l 100 work/emails.txt work/chunks/c
for f in work/chunks/*; do b=$(basename "$f"); [ -f "work/done/$b" ] && continue
  bin/klaviyo suppress --file "$f" --yes | grep -q submitted && touch "work/done/$b"
  sleep 0.2
done
```

Verify by spot-checking profiles (`bin/klaviyo get-profile <email> --subscriptions`), not by segment counts, which lag for hours.

**Unsuppress only clears manual suppressions.** It can't undo hard bounces or spam complaints, and it never resubscribes anyone.

## SMS

- Never put "Reply STOP to opt out" in the body. Klaviyo appends it automatically.
- Send between 8am and 9pm in the recipient's timezone. 1 to 3pm Eastern is safe across the US.
- Use SMS audiences (e.g. "Accepts SMS Marketing"). Email segments won't work for SMS.
- Pass multi-line bodies as a file (`--body @work/sms.txt`) so the line breaks survive.

## Template math

Klaviyo's template language has `add` and `{% widthratio a b c %}` (which computes `round(a*c/b)`, rounding rather than truncating). There's no subtract or multiply. Tricks that work:
- Subtract a variable: negate it first with `{% widthratio X 1 -1 as neg_x %}`, then `{{ P|add:neg_x }}`.
- Floor division `floor(P/N)`: `round((2P-(N-1))/2N)`. For N=500: `{% widthratio P|add:P|add:'-499' 1000 1 %}`.
- Coerce a profile property to a number: `person|lookup:'Prop'|default:0|add:0`.
