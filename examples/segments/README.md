# Example segment definitions

Starting points for `bin/klaviyo create-segment --name "..." --body @file.json`. Each one has `<PLACEHOLDER>` metric IDs, because metric IDs are different in every Klaviyo account. Claude looks them up with `bin/klaviyo list-metrics` and fills them in, so you don't need to edit these by hand.

How to read them: **condition groups are AND'd, conditions inside a group are OR'd.**

| File | Who's in it |
|---|---|
| `engaged-30d.json` | Opened, clicked, or was active on site in the last 30 days, and can receive email |
| `bought-category-ever.json` | Ever ordered a product in `<CATEGORY_NAME>` (e.g. only the smokers, not the accessories) |
| `bought-but-never-category.json` | Has ordered something, but never from `<CATEGORY_NAME>`. The cross-sell audience |
| `sunset-unengaged-never-bought.json` | 10+ emails received, no opens, clicks, or site visits in 270 days, never ordered, profile at least 270 days old |
| `suppressed-reengaging-30d.json` | Was suppressed more than 30 days ago, is still suppressed, and has ordered, carted, checked out, or viewed a product in the last 30 days. Used by the re-activation flow |

Notes:
- If the account has several instances of a metric (e.g. two "Added to Cart"), add one condition per instance to the OR group.
- The category filter assumes the Ordered Product event has a `Categories` property. Check what your events actually carry first: `bin/klaviyo events --metric <id> --properties --limit 5`. Swap in `ProductName`, `SKU`, or `Brand` if that separates things better.
