---
name: manual-sync
description: Fetch new LeetCode comp posts and manually parse/normalize them (replacing the LLM pipeline). Invoke with /manual-sync.
disable-model-invocation: true
---

# LeetComp Manual Sync

You replace the LLM steps of `leetcomp-sync`. The fetch script grabs raw posts, then YOU parse, normalize, and rebuild final data.

## Step 1: Fetch Raw Posts

```bash
cd {{workspace}} && uv run python -c "
import asyncio
from leetcomp.utils import last_fetched_info
from leetcomp.fetch import fetch_posts_in_bulk
from leetcomp import POSTS_FILE
till_id, till_ts = last_fetched_info(POSTS_FILE)
print(f'OLD_FIRST_ID={till_id}')
asyncio.run(fetch_posts_in_bulk(till_id=till_id, till_timestamp=till_ts))
print('FETCH_DONE')
"
```

Note the `OLD_FIRST_ID` printed. All lines in `data/posts.jsonl` before that id's line are new posts. If no new posts were fetched, stop and report that.

## Step 2: Parse New Posts

Read the new posts from `data/posts.jsonl`. For each post:

- **Skip** if downvotes > upvotes → `{"id": ..., "created_at": ..., "skip": true}`
- **Skip** if not a compensation post → `{"id": ..., "created_at": ..., "skip": true}`
- **Parse** India-based offers only (Lakhs/LPA/INR/₹ = India signal)

Parsed record JSON line:
```json
{"id": <int>, "created_at": "<str>", "skip": false, "compensation-post": true,
 "offer-type": "full-time", "company": "<raw>", "company-normalized": "<lowercase>",
 "role": "<raw>", "role-normalized": "<lowercase short form>", "yoe": <float>,
 "base": <float, LPA>, "total": <float LPA explicit>, "total-calculated": <float LPA computed>,
 "location": "<city>", "currency": "INR"}
```

Key rules:
- LPA values: "30 lacs"→30, "₹10,00,000"→10, "1.5 Cr"→150
- `total` = user-stated total; `total-calculated` = annual compensation computed from recurring annual components only (base + annual bonus/target bonus + annualized stocks/equity), and only if no explicit total is stated
- Exclude one-time or non-recurring items from `total-calculated`: joining/sign-on bonus, relocation bonus, retention bonus, PF, gratuity, retirals, benefits, reimbursements, clawback notes, etc.
- If the user explicitly states a first-year/total compensation number, prefer `total` exactly as stated instead of recomputing
- Omit missing fields entirely (never N/A)
- One post can yield multiple parsed records
- company-normalized: lowercase normalized ("Samsung R&D" → "samsung")
- role-normalized: lowercase normalized key matching existing `data/role_map.json` conventions when possible; prefer specific normalized titles over overly generic ones when the role is explicitly stated (e.g. `application engineer l3` is better than `l3`)

**Critical accuracy rules:**
- Never infer unstated fields. Only include fields explicitly stated in the post or directly derivable from explicit compensation text
- If role is not stated, omit `role` and `role-normalized`
- If location for the offer is not stated, omit `location` even if the author mentions current city, preferred city, NCR, office preference, or a company office elsewhere
- Do not invent a generic role like `sde`/`sse` just because the YOE/company suggests it
- For multi-offer posts, keep each offer isolated; do not leak role/location/YOE/comp components from one offer into another unless clearly shared
- Preserve raw `role` text from the post when present, and normalize separately in `role-normalized`

**Mandatory verification before writing:**
- Re-read every newly parsed record against the raw post text
- Check that every non-derived field (`company`, `role`, `location`, `yoe`, explicit `total`) appears in the post text
- Check that every derived field is traceable and formula-correct
- Check `role-normalized` against existing `data/role_map.json` patterns before creating a new style of normalization
- Check that `total-calculated` excludes one-time components unless the post explicitly gives a total, in which case use `total`
- Only after this verification, prepend all new parsed lines to `data/parsed_posts.jsonl` (newest first)

## Step 3: Update Entity Maps

For each of `company_map.json`, `role_map.json`, `location_map.json`:

1. Load existing map (keys are lowercase)
2. Collect new entity values from your parsed records (`company-normalized`, `role-normalized`, `location`)
3. For any value not already in the map, add: `"lowercase_variant": "Canonical Name"`
4. Save sorted by key

**Canonical naming:**
- Company: proper casing (Google, Amazon, JPMorgan Chase, PhonePe, IBM)
- Role: UPPERCASE abbrevs (SDE, SSE, MLE), Title Case full names (Data Scientist). Preserve levels (SDE 2)
- Location: official name (Bengaluru, Gurugram, Hyderabad)

## Step 4: Rebuild final_data.json

```bash
cd {{workspace}} && uv run python -c "from leetcomp.transform import create_final_data; create_final_data()"
```

## Step 5: Report

Print summary: new posts fetched, parsed offers vs skipped, new mappings added per entity type, total final records.

```bash
cd {{workspace}} && python3 -c "import json; print(f'Final records: {len(json.load(open(\"data/final_data.json\")))}')"
```
