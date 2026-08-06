# Nutrition tracking — design

Date: 2026-08-06
Status: approved, not yet implemented

## Problem

RunLog logs training but not food. Etienne wants to log what he eats against a
daily target, with as little typing as possible: photograph a product's
nutrition label once, then log grams thereafter. For unpackaged foods ("3
eggs") the app should already know the macros.

Targets are fixed: **2,500 kcal and 170 g protein per day**, every day. These
came from analysing 17 months of WHOOP data (mean TDEE 2,321 kcal + ~200 kcal
NEAT correction) for a 33 y/o male, 75 kg, 1.65 m, training ~5×/week.

## Scope

In scope: pantry of foods, photo-to-macros for packaged products, a built-in
table of common foods, per-meal logging, daily totals against target, sync to
`runlog-data`.

Out of scope: barcode scanning, plate photos with portion estimation, recipe
composition, calorie cycling by training day (explicitly declined), micronutrients.

## Decisions and rationale

| Decision | Rationale |
|---|---|
| Pantry model (photo once, reuse) | Label photos are one-time per product; per-meal plate photos carry ±30–50% portion error, which swamps a 170 g protein target |
| Manual form is the single write path; photo **pre-fills** it | One validation path; an edit form is needed regardless; vision failure degrades to typing instead of breaking |
| Fixed 2,500 / 170 target | Chosen over strain-tiered and live-WHOOP targets — a target that drifts during the day can't be planned against |
| Built-in food table over external API | Instant, offline, free, deterministic; covers ~95% of what he eats |
| Two sync files (`pantry.json`, `nutrition.json`) | Mirrors the existing `vo2max.json` pattern; pantry rarely changes so meal logging doesn't rewrite it |
| Code lives in `index.html` | No service worker; self-update works by regex-matching `APP_VERSION` out of the served `index.html` (`index.html:7536`). A separate `.js` file would sit outside that check and could go stale against new HTML |
| Claude Haiku 4.5 for label reading | ~$0.0026 per label (~$0.13/yr at expected volume); reading a macro panel is OCR + arithmetic, not reasoning; supports structured outputs |

## Architecture

```
runlog-app/index.html          runlog-auth/api/            runlog-data/
─────────────────────          ────────────────            ────────────
pantry + nutrition stores      nutrition-label.js  ──────► (nothing)
food table (~80 items)           ANTHROPIC_API_KEY
log form, Today view                 │                     pantry.json
GitHub sync ─────────────────────────┼───────────────────► nutrition.json
                                     ▼
                              api.anthropic.com
                              claude-haiku-4-5
```

`runlog-auth` gains one endpoint and one secret. It stays a credential broker —
no nutrition logic, no storage, no totals. All computation is client-side.

## Data model

### IndexedDB — `runlog-db`, version 4 → 5

Two new stores; existing four untouched. The existing `onupgradeneeded` uses
`if (!contains)` guards, so the migration is additive and safe.

```
pantry     keyPath 'id'     index on 'name'
nutrition  keyPath 'id'     index on 'date'
```

### Pantry item

```json
{
  "id": "pnt_k3f9x2",
  "name": "Barilla Penne Rigate",
  "category": "pasta",
  "per100g": { "kcal": 352, "protein": 12.5, "carbs": 71.2, "fat": 1.5 },
  "unit": null,
  "cookedFactor": 2.4,
  "source": "photo",
  "createdAt": "2026-08-06T10:00:00Z",
  "updatedAt": "2026-08-06T10:00:00Z",
  "archived": false
}
```

- `unit` — `null` for weighed foods; `{ "label": "egg", "grams": 55 }` for
  countable ones, enabling "3 eggs".
- `cookedFactor` — grams cooked per gram as-sold. Editable per item.
- `source` — `photo` | `manual` | `builtin`.
- `archived` — hides an item from the picker without breaking history.

### Log entry

```json
{
  "id": "ent_9m2q",
  "date": "2026-08-06",
  "meal": "lunch",
  "foodId": "pnt_k3f9x2",
  "foodName": "Barilla Penne Rigate",
  "amount": 120,
  "mode": "dry",
  "grams": 120,
  "macros": { "kcal": 422, "protein": 15, "carbs": 85, "fat": 1.8 },
  "createdAt": "2026-08-06T12:30:00Z"
}
```

There is deliberately **no per-entry `syncState`**. Sessions carry one because
each session is its own file in `runlog-data/sessions/`. Nutrition uses the
`vo2max.json` model — a whole-file push of complete local state — where a
per-entry flag would be meaningless. See "Sync failures" below.

`foodName` and `macros` are **snapshots, not references**. Correcting or
archiving a pantry item must not retroactively change logged history. This
mirrors how sessions embed `_whoopData` inline.

### Built-in foods

~80 items shipped as a `const BUILTIN_FOODS` array, seeded into the `pantry`
store on first load and guarded by a `builtinSeedVersion` key in the existing
`config` store. One store, one code path, and built-ins stay editable. Bumping
the seed version adds new items without clobbering user edits.

Coverage: eggs, chicken breast, beef mince, salmon, tuna, whey, milk, Greek
yoghurt, cheese, rice, pasta, couscous, oats, bread, potato, olive oil, butter,
nuts, common fruit and vegetables.

## The dry/cooked conversion

One formula, both directions:

| `mode` | as-sold grams |
|---|---|
| `dry` | `amount` |
| `cooked` | `amount / cookedFactor` |
| `units` | `amount × unit.grams` |

Then `macros = per100g × grams / 100`.

Category defaults: pasta 2.4, rice 3.0, oats 3.0, dried legumes 2.4, couscous
3.0, chicken breast 0.75, beef mince 0.70.

Foods that *lose* water use a factor below 1 and the same formula still holds:
100 g raw chicken → 75 g cooked, and `75 / 0.75 = 100` recovers the raw weight.

## The vision endpoint

`POST /api/nutrition-label` in `runlog-auth`, alongside the existing four
handlers. Follows their conventions exactly: CORS preflight, `ALLOWED_ORIGIN`,
method guard, `server_not_configured` when the key is missing.

Request: `{ "image": "<base64>", "media_type": "image/jpeg" }`
New env var: `ANTHROPIC_API_KEY`

Model `claude-haiku-4-5`, with a forced JSON schema via
`output_config.format` so the response is guaranteed parseable rather than
prose requiring extraction:

```json
{
  "readable": true,
  "reason": null,
  "name": "Barilla Penne Rigate",
  "per100g": { "kcal": 352, "protein": 12.5, "carbs": 71.2, "fat": 1.5 },
  "category": "pasta"
}
```

The client resizes images to ≤1568 px on the long edge before upload — this
matches Haiku 4.5's image tier and caps token cost at ~1,600 tokens per image.

### Cost

~1,850 input + ~150 output tokens per label at $1/$5 per million =
**$0.0026 per label**, roughly $0.13/year at an expected ~50 new products.
Billed to the Anthropic API organisation, which is separate from any Claude
subscription.

## Error handling

**Every vision failure path ends at the manual form.** No dead ends, and no
fabricated numbers.

| Failure | Behavior |
|---|---|
| `ANTHROPIC_API_KEY` unset | 500 `server_not_configured` |
| Image > 5 MB | 413 server-side; client resizes first so this is a backstop |
| Label unreadable | `readable: false` + `reason`; show it, open a **blank** form |
| Network, 429, timeout | Show the error; open a blank form |
| Malformed model output | Treated as unreadable |

### Macro-panel sanity check

Before pre-filling, validate the returned panel to catch a misread or
hallucinated value:

- all values numeric and ≥ 0
- `kcal ≤ 900` per 100 g (pure fat is 900)
- `kcal` roughly agrees with Atwater: `4×protein + 4×carbs + 9×fat`

**The Atwater tolerance is asymmetric** — revised during implementation after
the symmetric ±10% rule flagged 10 of the 80 built-in foods, all fruit and
vegetables. Labels count dietary fibre inside total carbohydrate, but fibre
yields ~2 kcal/g rather than 4, so `4×carbs` over-estimates energy for
fibre-rich foods: broccoli derives 42.8 kcal against a true 34, a 26%
overshoot that is entirely normal. Nothing makes the derived value come out
*below* the stated kcal, so that direction stays tight:

| Direction | Tolerance |
|---|---|
| derived > kcal (fibre effect) | 30%, floor 10 kcal |
| derived < kcal (real error) | 10%, floor 10 kcal |

Verified: 0 of 80 built-in foods flagged, while still catching a nonsense
panel, a 352→400 kcal misread, and a protein misread.

**Known blind spot:** a uniformly scaled panel — per-serving values mistaken
for per-100 g doubles everything — remains internally consistent and passes.
No arithmetic check can catch that. Mitigations are the prompt instruction to
convert per-serving values, and the user confirming every panel before saving.

On failure the fields still pre-fill but are flagged for review rather than
silently accepted.

### Sync failures

Follows the `vo2max.json` model, not the sessions model. `syncAllUnsynced()`
(`index.html:2708`) calls `getAllSessions()` and therefore does **not** cover
these stores — it must not be relied on here.

Each mutation triggers a whole-file push of complete local state, reusing
`ghGetFileSha()` / `ghGetFileShaViaTree()` and the 409/422 SHA-conflict retry
already implemented in `vo2SyncToGh()` (`index.html:7130`).

Because every push contains complete state, a failed push is self-healing — the
next successful push carries everything. The one gap in the existing vo2 code is
that a failure with no subsequent mutation leaves GitHub stale indefinitely
(failures are logged via `.catch(e => console.warn(...))` and never retried).
Nutrition closes that gap with a `nutritionDirty` boolean in the `config` store:
set on any failed push, checked on app load, and cleared on a successful push.
IndexedDB remains the source of truth throughout, so no data is at risk — only
the GitHub copy lags.

## UI

A sixth tab, **Food**, containing three views:

1. **Today** (default) — totals vs 2,500 kcal / 170 g with progress bars,
   grouped by meal, tap an entry to edit or delete.
2. **Log** — pick food (search, recent-first), enter amount, choose
   dry/cooked/units, pick meal, save.
3. **Pantry** — list, search, add (photo or manual), edit, archive.

Six tabs is tight on a phone; shorten "Program" to "Prog" if it doesn't fit.

## Testing

**This project has no test framework** — no `package.json` in `runlog-app`, no
tests in `runlog-auth`. Rather than introduce build tooling unasked:

- `resolveGrams()`, `computeMacros()`, and `validateMacroPanel()` are written as
  pure functions with no DOM or IndexedDB access.
- A `?selftest=1` URL flag runs ~20 assertions against them and reports results
  in-page. Zero dependencies, consistent with the project's style.
- Endpoint verified by `curl` against real label photos plus the unreadable,
  oversized, and missing-key cases.

Manual verification before shipping: add a food by photo, add one manually, log
a dry-weight meal, log a cooked-weight meal, log a unit-based food, confirm
totals, confirm both JSON files land in `runlog-data`, reload and confirm
persistence.

## Risks

- **~1,100 lines added to a 7,662-line file.** Accepted for update-mechanism
  consistency. If the file becomes unmanageable, extract CSS first — it's 1,575
  lines and has no coupling to the update check.
- **`ALLOWED_ORIGIN` currently defaults to `*`.** Should be pinned to
  `https://emimy.github.io` before adding another endpoint. Tracked separately.
- **Label formats vary by country** (per-serving vs per-100g, kJ vs kcal). The
  prompt must instruct normalisation to per-100 g kcal, and the sanity check is
  the backstop when it goes wrong.
