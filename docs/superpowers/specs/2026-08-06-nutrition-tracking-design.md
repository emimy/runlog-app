# Nutrition tracking — design

Date: 2026-08-06
Status: in progress
Revised 2026-08-06: photo/vision scope removed at Etienne's request — foods are
entered manually. `runlog-auth` is untouched; no API key, no deploy, no cost.
The manual form was already the single write path, so this removes a layer
rather than changing the architecture.

## Problem

RunLog logs training but not food. Etienne wants to log what he eats against a
daily target, with as little repeated typing as possible: enter a product's
macros once, then log grams thereafter. For unpackaged foods ("3 eggs") the app
should already know the macros.

Targets are fixed: **2,500 kcal and 170 g protein per day**, every day. These
came from analysing 17 months of WHOOP data (mean TDEE 2,321 kcal + ~200 kcal
NEAT correction) for a 33 y/o male, 75 kg, 1.65 m, training ~5×/week.

## Scope

In scope: pantry of foods entered by hand, a built-in table of common foods,
per-meal logging, daily totals against target, sync to `runlog-data`.

Out of scope: **photo-to-macros / any vision endpoint (dropped 2026-08-06)**,
barcode scanning, plate photos with portion estimation, recipe composition,
calorie cycling by training day (explicitly declined), micronutrients.

## Decisions and rationale

| Decision | Rationale |
|---|---|
| Pantry model — enter a food once, reuse forever | Typing macros is a one-time cost per product; the daily flow is then pick-and-weigh |
| Manual form is the single write path | One validation path, one place to fix a wrong number later |
| Fixed 2,500 / 170 target | Chosen over strain-tiered and live-WHOOP targets — a target that drifts during the day can't be planned against |
| Built-in food table over external API | Instant, offline, free, deterministic; covers ~95% of what he eats |
| Two sync files (`pantry.json`, `nutrition.json`) | Mirrors the existing `vo2max.json` pattern; pantry rarely changes so meal logging doesn't rewrite it |
| Code lives in `index.html` | No service worker; self-update works by regex-matching `APP_VERSION` out of the served `index.html` (`index.html:7536`). A separate `.js` file would sit outside that check and could go stale against new HTML |
| ~~Claude Haiku 4.5 for label reading~~ | **Dropped 2026-08-06.** Removed the only reason to touch `runlog-auth`, the only API key, and the only per-use cost. The pantry form it would have pre-filled already existed |

## Architecture

```
runlog-app/index.html                              runlog-data/
─────────────────────                              ────────────
pantry + nutrition stores
food table (~80 items)                             pantry.json
pantry form, log form, Today view                  nutrition.json
GitHub sync ─────────────────────────────────────► (whole-file push)
```

Entirely contained in the PWA. `runlog-auth` is not touched — no new endpoint,
no new secret. There is no server component and no network dependency beyond
the GitHub sync that already exists.

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
  "source": "manual",
  "createdAt": "2026-08-06T10:00:00Z",
  "updatedAt": "2026-08-06T10:00:00Z",
  "archived": false
}
```

- `unit` — `null` for weighed foods; `{ "label": "egg", "grams": 55 }` for
  countable ones, enabling "3 eggs".
- `cookedFactor` — grams cooked per gram as-sold. Editable per item.
- `source` — `manual` | `builtin`.
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

## Manual entry (was: the vision endpoint)

Foods are added through the pantry form: name, category, per-100 g kcal /
protein / carbs / fat, optional cooked-conversion factor, optional unit weight.
Roughly 20 seconds per product, done once per product.

`runlog-auth` is **not modified**. No `ANTHROPIC_API_KEY`, no new endpoint, no
deploy, no per-use cost.

## Error handling

With no network call in the add-food path, the failure modes are typing
mistakes. The form rejects a missing name or any missing macro outright, and
warns (but still allows saving) when the numbers don't cohere.

### Macro-panel sanity check

When saving a food, validate the typed panel to catch a slip or a
misread column on the packet:

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

**Known blind spot:** a uniformly scaled panel — per-serving values typed in
where per-100 g was wanted — stays internally consistent and passes. No
arithmetic check can catch that; reading the right column off the packet is the
only defence.

On failure the form warns and asks for confirmation rather than refusing the
save — some legitimate foods sit outside the check, and the user is the
authority on what's on the packet in front of them.

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
   grouped by meal, tap an entry to delete.
2. **Log** — pick food (search), enter amount, choose dry/cooked/units, pick
   meal, save.
3. **Pantry** — list, filter, add, edit, delete.

Six tabs is tight on a phone; shorten "Program" to "Prog" if it doesn't fit.

## Testing

**This project has no test framework** — no `package.json` in `runlog-app`.
Rather than introduce build tooling unasked:

- `resolveGrams()`, `computeMacros()`, `validateMacroPanel()`, `sumMacros()`,
  and `buildEntry()` are written as pure functions with no DOM or IndexedDB
  access.
- A `?selftest=1` URL flag runs the assertions and reports results in-page.
  Zero dependencies, consistent with the project's style. Storage round-trips
  and the built-in food table are covered by async assertions in the same
  harness — every one of the 80 built-in panels is validated on each run, so a
  typo in that table fails the suite rather than silently corrupting logs.

Manual verification before shipping: add a food, log a dry-weight meal, log a
cooked-weight meal, log a unit-based food (eggs), confirm totals, confirm both
JSON files land in `runlog-data`, reload and confirm persistence, and confirm
existing sessions/VO2/program data survived the v4→v5 upgrade.

## Risks

- **~1,100 lines added to a 7,662-line file.** Accepted for update-mechanism
  consistency. If the file becomes unmanageable, extract CSS first — it's 1,575
  lines and has no coupling to the update check.
- **Label formats vary by country** (per-serving vs per-100 g, kJ vs kcal).
  Reading the wrong column is the most likely real-world error and the sanity
  check cannot detect it. If kJ is all that's printed, divide by 4.184.
- **`ALLOWED_ORIGIN` in `runlog-auth` defaults to `*`.** Pre-existing, unrelated
  to this feature, still worth pinning to `https://emimy.github.io`.
