# Nutrition Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add food logging to the RunLog PWA — photograph a product label once to add it to a pantry, then log grams per meal against fixed daily targets of 2,500 kcal and 170 g protein.

**Architecture:** All storage and computation is client-side in `runlog-app/index.html` (IndexedDB + a new "Food" tab), synced to `runlog-data` as two JSON files following the existing `vo2max.json` whole-file pattern. `runlog-auth` gains exactly one endpoint that holds `ANTHROPIC_API_KEY` and turns a label photo into per-100 g macros; it stores nothing and computes nothing else.

**Tech Stack:** Vanilla JS (no framework, no build step), IndexedDB, GitHub Contents API, Vercel serverless functions (Node 20, ESM), Claude Haiku 4.5 with structured outputs.

**Spec:** `docs/superpowers/specs/2026-08-06-nutrition-tracking-design.md`

## Global Constraints

- **No build step, no new runtime dependencies.** `runlog-app` is a single hand-edited `index.html` served from GitHub Pages; `runlog-auth` has zero dependencies in `package.json`.
- **All new PWA code goes in `runlog-app/index.html`.** A separate `.js` file would sit outside the `APP_VERSION` self-update check at `index.html:7536` and could go stale against new HTML.
- **Follow existing section style:** `// ====` banner comments, `uid()` for IDs, `toast(msg, isError)` for user feedback, `getConfig()`/`setConfig()` for the config store, `.field` / `.settings-group` / `.empty-state` / `button.primary` CSS classes.
- **Bump `APP_VERSION`** (`index.html:2027`) in the final task only — it triggers the update banner for a running client.
- **Daily targets are fixed:** `DAILY_KCAL_TARGET = 2500`, `DAILY_PROTEIN_TARGET = 170`. Do not make them strain-dependent.
- **Never fabricate macros.** Every vision failure path ends at a blank manual form.
- **Model ID is exactly `claude-haiku-4-5`.** No date suffix.
- Branch: `feat/nutrition-tracking` (already created; spec already committed there).

---

## File Structure

| File | Responsibility | Change |
|---|---|---|
| `runlog-app/index.html` | Everything client-side: stores, math, food table, UI, sync | Modify (~1,100 lines added) |
| `runlog-auth/api/nutrition-label.js` | Label photo → per-100 g macros | Create (~95 lines) |
| `runlog-auth/README.md` | Document the new endpoint + env var | Modify |
| `runlog-app/docs/superpowers/plans/…` | This plan | Already created |

Within `index.html`, new code is added as five banner-delimited sections placed **after** the VO2 max section (which ends at `index.html:7168`) so the diff stays contiguous:

1. `// Nutrition — pure math` (Task 1)
2. `// Nutrition — storage` (Task 2)
3. `// Nutrition — built-in foods` (Task 3)
4. `// Nutrition — UI` (Tasks 4–6)
5. `// Nutrition — GitHub sync` (Task 7)

---

### Task 1: Self-test harness and pure math functions

The project has no test framework. This task builds the minimal harness every later task uses, plus the three pure functions that carry all the arithmetic risk.

**Files:**
- Modify: `runlog-app/index.html` — add a new section after line 7168 (end of the VO2 sync block)

**Interfaces:**
- Consumes: nothing
- Produces:
  - `resolveGrams(item, amount, mode) -> number|null` where `mode` is `'dry'|'cooked'|'units'`
  - `computeMacros(item, grams) -> {kcal, protein, carbs, fat}|null`
  - `validateMacroPanel(per100g) -> {ok: boolean, issues: string[]}`
  - `round1(n) -> number`
  - `assert(cond, msg)` and `runSelfTests()` — the harness, invoked by `?selftest=1`

- [ ] **Step 1: Write the failing tests**

Insert after `index.html:7168`:

```javascript
// ============================================================
// Nutrition — pure math
// Pure functions only: no DOM, no IndexedDB, no network. These carry
// all the arithmetic risk in the nutrition feature and are the only
// part covered by ?selftest=1.
// ============================================================

// --- self-test harness ---------------------------------------------------
// The project has no test framework and no build step. Loading the app with
// ?selftest=1 runs these assertions and renders the results in place of the
// normal UI. Cheap, dependency-free, and runs in the real browser.
let _selfTestFailures = [];

function assert(cond, msg) {
  if (!cond) _selfTestFailures.push(msg);
  return !!cond;
}

function assertClose(actual, expected, tol, msg) {
  const ok = typeof actual === 'number' && Math.abs(actual - expected) <= tol;
  if (!ok) _selfTestFailures.push(msg + ' (got ' + actual + ', want ' + expected + '±' + tol + ')');
  return ok;
}

function runSelfTests() {
  _selfTestFailures = [];
  let count = 0;
  const t = (fn) => { count++; fn(); };

  const penne = {
    per100g: { kcal: 352, protein: 12.5, carbs: 71.2, fat: 1.5 },
    cookedFactor: 2.4, unit: null,
  };
  const egg = {
    per100g: { kcal: 143, protein: 12.6, carbs: 0.7, fat: 9.5 },
    cookedFactor: null, unit: { label: 'egg', grams: 55 },
  };
  const chicken = {
    per100g: { kcal: 165, protein: 31, carbs: 0, fat: 3.6 },
    cookedFactor: 0.75, unit: null,
  };

  // resolveGrams — dry mode is identity
  t(() => assert(resolveGrams(penne, 120, 'dry') === 120, 'dry mode returns amount unchanged'));

  // resolveGrams — cooked mode divides by the factor (water gained)
  t(() => assertClose(resolveGrams(penne, 288, 'cooked'), 120, 0.001,
    'cooked pasta converts back to dry weight'));

  // resolveGrams — cooked mode also handles water LOST (factor < 1)
  t(() => assertClose(resolveGrams(chicken, 75, 'cooked'), 100, 0.001,
    'cooked chicken converts back to raw weight'));

  // resolveGrams — units mode multiplies
  t(() => assert(resolveGrams(egg, 3, 'units') === 165, '3 eggs resolves to 165g'));

  // resolveGrams — invalid inputs return null, never NaN
  t(() => assert(resolveGrams(penne, -5, 'dry') === null, 'negative amount rejected'));
  t(() => assert(resolveGrams(penne, 100, 'units') === null, 'units mode without unit rejected'));
  t(() => assert(resolveGrams(egg, 100, 'cooked') === null, 'cooked mode without factor rejected'));
  t(() => assert(resolveGrams(penne, 100, 'bogus') === null, 'unknown mode rejected'));
  t(() => assert(resolveGrams(null, 100, 'dry') === null, 'null item rejected'));

  // computeMacros — scales linearly from per-100g
  t(() => {
    const m = computeMacros(penne, 120);
    assertClose(m.kcal, 422.4, 0.05, 'penne 120g kcal');
    assertClose(m.protein, 15, 0.05, 'penne 120g protein');
    assertClose(m.carbs, 85.4, 0.05, 'penne 120g carbs');
    assertClose(m.fat, 1.8, 0.05, 'penne 120g fat');
  });
  t(() => {
    const m = computeMacros(egg, 165);
    assertClose(m.protein, 20.8, 0.05, '3 eggs protein');
  });
  t(() => assert(computeMacros(penne, 0).kcal === 0, 'zero grams gives zero kcal'));
  t(() => assert(computeMacros(penne, -1) === null, 'negative grams rejected'));

  // validateMacroPanel — real labels pass
  t(() => assert(validateMacroPanel(penne.per100g).ok, 'penne panel is self-consistent'));
  t(() => assert(validateMacroPanel({ kcal: 884, protein: 0, carbs: 0, fat: 100 }).ok,
    'olive oil panel passes (884 vs 900 derived)'));
  t(() => assert(validateMacroPanel({ kcal: 15, protein: 1.4, carbs: 2.9, fat: 0.2 }).ok,
    'low-calorie vegetable passes via absolute floor'));

  // validateMacroPanel — catches misreads
  t(() => assert(!validateMacroPanel({ kcal: 950, protein: 0, carbs: 0, fat: 100 }).ok,
    'kcal above 900/100g rejected'));
  t(() => assert(!validateMacroPanel({ kcal: 100, protein: 50, carbs: 50, fat: 50 }).ok,
    'kcal inconsistent with macros rejected'));
  t(() => assert(!validateMacroPanel({ kcal: -5, protein: 1, carbs: 1, fat: 1 }).ok,
    'negative kcal rejected'));
  t(() => assert(!validateMacroPanel({ kcal: 100, protein: 'x', carbs: 1, fat: 1 }).ok,
    'non-numeric macro rejected'));
  t(() => assert(!validateMacroPanel(null).ok, 'null panel rejected'));

  return { count, failures: _selfTestFailures };
}

// Run on ?selftest=1 and render results instead of the normal app.
if (new URLSearchParams(location.search).get('selftest') === '1') {
  window.addEventListener('DOMContentLoaded', () => {
    const r = runSelfTests();
    document.body.innerHTML =
      '<pre style="padding:16px;font:13px/1.5 ui-monospace,monospace;white-space:pre-wrap">' +
      (r.failures.length === 0
        ? '✅ ' + r.count + ' self-test groups passed'
        : '❌ ' + r.failures.length + ' failure(s) of ' + r.count + ' groups\n\n' +
          r.failures.map(f => '  • ' + f).join('\n')) +
      '</pre>';
  });
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Serve the file and open the self-test URL:

```bash
cd runlog-app && python3 -m http.server 8123 &
open 'http://localhost:8123/index.html?selftest=1'
```

Expected: the page shows a JavaScript error in the console (`resolveGrams is not defined`) — the functions do not exist yet.

- [ ] **Step 3: Write the minimal implementation**

Insert immediately below the harness, before the `?selftest=1` block:

```javascript
// --- math ----------------------------------------------------------------

// Round to one decimal. Macro panels are never more precise than this, and
// it keeps summed daily totals from accumulating float noise.
function round1(n) {
  return Math.round(n * 10) / 10;
}

// Resolve a user-entered amount to grams "as sold" — the basis every
// per-100g label uses.
//   dry    : the amount already is as-sold grams (weighed before cooking)
//   cooked : divide by cookedFactor. Works in both directions — pasta gains
//            water (factor 2.4) and chicken loses it (factor 0.75), and
//            amount/factor recovers the as-sold weight either way.
//   units  : multiply by the item's per-unit gram weight (e.g. 1 egg = 55g)
// Returns null rather than NaN on any invalid input so callers can show a
// validation message instead of writing garbage to the log.
function resolveGrams(item, amount, mode) {
  if (!item) return null;
  if (typeof amount !== 'number' || !isFinite(amount) || amount < 0) return null;
  if (mode === 'dry') return amount;
  if (mode === 'cooked') {
    const f = item.cookedFactor;
    if (typeof f !== 'number' || !isFinite(f) || f <= 0) return null;
    return amount / f;
  }
  if (mode === 'units') {
    const g = item.unit && item.unit.grams;
    if (typeof g !== 'number' || !isFinite(g) || g <= 0) return null;
    return amount * g;
  }
  return null;
}

// Scale a pantry item's per-100g panel to an actual gram weight.
function computeMacros(item, grams) {
  if (!item || !item.per100g) return null;
  if (typeof grams !== 'number' || !isFinite(grams) || grams < 0) return null;
  const r = grams / 100;
  const p = item.per100g;
  return {
    kcal:    round1(p.kcal * r),
    protein: round1(p.protein * r),
    carbs:   round1(p.carbs * r),
    fat:     round1(p.fat * r),
  };
}

// Sanity-check a per-100g panel before trusting it — catches a misread
// label or a hallucinated value from the vision endpoint.
//   - every field numeric and >= 0
//   - kcal <= 900 (pure fat is 900 kcal/100g; nothing exceeds it)
//   - kcal within 10% of 4*protein + 4*carbs + 9*fat
// The tolerance has an absolute floor of 5 kcal so near-zero foods
// (lettuce, black coffee) aren't flagged by a percentage that tight.
function validateMacroPanel(p) {
  if (!p || typeof p !== 'object') return { ok: false, issues: ['missing'] };
  const issues = [];
  for (const k of ['kcal', 'protein', 'carbs', 'fat']) {
    const v = p[k];
    if (typeof v !== 'number' || !isFinite(v) || v < 0) issues.push(k + '_invalid');
  }
  if (issues.length) return { ok: false, issues };
  if (p.kcal > 900) issues.push('kcal_too_high');
  const derived = 4 * p.protein + 4 * p.carbs + 9 * p.fat;
  const tol = Math.max(Math.max(derived, p.kcal) * 0.10, 5);
  if (Math.abs(derived - p.kcal) > tol) issues.push('kcal_mismatch');
  return { ok: issues.length === 0, issues };
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Reload `http://localhost:8123/index.html?selftest=1`

Expected: `✅ 21 self-test groups passed`

If any fail, the page lists each failing assertion by name. Fix and reload.

- [ ] **Step 5: Commit**

```bash
git add runlog-app/index.html
git commit -m "feat(nutrition): pure math functions and ?selftest=1 harness"
```

---

### Task 2: IndexedDB stores for pantry and nutrition

**Files:**
- Modify: `runlog-app/index.html:2033` (`DB_VERSION`), `index.html:2044-2062` (`onupgradeneeded`), and add a storage section after the Task 1 block

**Interfaces:**
- Consumes: `uid()` (`index.html:2205`), the `db` handle
- Produces:
  - `putPantryItem(item) -> Promise<void>`
  - `getAllPantry() -> Promise<PantryItem[]>`
  - `getPantryItem(id) -> Promise<PantryItem|undefined>`
  - `deletePantryItem(id) -> Promise<void>`
  - `putNutritionEntry(entry) -> Promise<void>`
  - `getNutritionByDate(date) -> Promise<Entry[]>` where `date` is `'YYYY-MM-DD'`
  - `getAllNutrition() -> Promise<Entry[]>`
  - `deleteNutritionEntry(id) -> Promise<void>`

- [ ] **Step 1: Write the failing test**

Add to `runSelfTests()`, just before the `return { count, failures: _selfTestFailures };` line. Note this group is async — the harness awaits it via a promise the test stores.

```javascript
  // Storage round-trip. Async, so it appends to a promise the harness awaits.
  _selfTestAsync.push((async () => {
    await openDB();
    const probe = {
      id: 'pnt_selftest', name: 'Self Test Food', category: 'other',
      per100g: { kcal: 100, protein: 10, carbs: 10, fat: 2 },
      unit: null, cookedFactor: null, source: 'manual',
      createdAt: '2026-01-01T00:00:00Z', updatedAt: '2026-01-01T00:00:00Z',
      archived: false,
    };
    await putPantryItem(probe);
    const back = await getPantryItem('pnt_selftest');
    assert(back && back.name === 'Self Test Food', 'pantry item round-trips');
    await deletePantryItem('pnt_selftest');
    assert(!(await getPantryItem('pnt_selftest')), 'pantry item deletes');

    const entry = {
      id: 'ent_selftest', date: '1999-12-31', meal: 'lunch',
      foodId: 'pnt_selftest', foodName: 'Self Test Food',
      amount: 100, mode: 'dry', grams: 100,
      macros: { kcal: 100, protein: 10, carbs: 10, fat: 2 },
      createdAt: '2026-01-01T00:00:00Z',
    };
    await putNutritionEntry(entry);
    const byDate = await getNutritionByDate('1999-12-31');
    assert(byDate.length === 1 && byDate[0].id === 'ent_selftest',
      'nutrition entry retrievable by date index');
    assert((await getNutritionByDate('1999-12-30')).length === 0,
      'date index does not leak across dates');
    await deleteNutritionEntry('ent_selftest');
    assert((await getNutritionByDate('1999-12-31')).length === 0,
      'nutrition entry deletes');
  })());
```

Add near the top of the math section (with `_selfTestFailures`):

```javascript
let _selfTestAsync = [];
```

Change `runSelfTests()` to `async function runSelfTests()`, reset `_selfTestAsync = []` alongside `_selfTestFailures = []`, and before the return add:

```javascript
  count += _selfTestAsync.length;
  await Promise.all(_selfTestAsync);
```

Change the `?selftest=1` handler to `async () => { const r = await runSelfTests(); … }`.

- [ ] **Step 2: Run the test to verify it fails**

Reload `http://localhost:8123/index.html?selftest=1`

Expected: failure — `putPantryItem is not defined`.

- [ ] **Step 3: Write the minimal implementation**

**3a.** At `index.html:2033`, bump the version and add the store name constants:

```javascript
const DB_VERSION = 5;
const STORE = 'sessions';
const CONFIG_STORE = 'config';
const VO2_STORE = 'vo2max';
const PROGRAM_COMPLETION_STORE = 'program_completion';
const PANTRY_STORE = 'pantry';
const NUTRITION_STORE = 'nutrition';
```

**3b.** Inside `onupgradeneeded`, after the existing `PROGRAM_COMPLETION_STORE` block (`index.html:2057-2061`), add:

```javascript
      if (!d.objectStoreNames.contains(PANTRY_STORE)) {
        // Foods available to log. Seeded with BUILTIN_FOODS on first run,
        // then grown by photo or manual entry.
        const ps = d.createObjectStore(PANTRY_STORE, { keyPath: 'id' });
        ps.createIndex('name', 'name', { unique: false });
      }
      if (!d.objectStoreNames.contains(NUTRITION_STORE)) {
        // One record per food per meal. Macros are snapshotted at log time
        // so later pantry edits never rewrite history.
        const ns = d.createObjectStore(NUTRITION_STORE, { keyPath: 'id' });
        ns.createIndex('date', 'date', { unique: false });
      }
```

**3c.** Add the storage section after the Task 1 math block:

```javascript
// ============================================================
// Nutrition — storage
// Pantry (foods) and nutrition (log entries) stores. Same
// promise-wrapped IndexedDB style as the sessions/vo2max helpers above.
// ============================================================

function pantryTx(mode = 'readonly') {
  return db.transaction(PANTRY_STORE, mode).objectStore(PANTRY_STORE);
}
function nutTx(mode = 'readonly') {
  return db.transaction(NUTRITION_STORE, mode).objectStore(NUTRITION_STORE);
}

function _req(request) {
  return new Promise((resolve, reject) => {
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

function putPantryItem(item)  { return _req(pantryTx('readwrite').put(item)).then(() => {}); }
function getPantryItem(id)    { return _req(pantryTx().get(id)); }
function getAllPantry()       { return _req(pantryTx().getAll()); }
function deletePantryItem(id) { return _req(pantryTx('readwrite').delete(id)).then(() => {}); }

function putNutritionEntry(e)   { return _req(nutTx('readwrite').put(e)).then(() => {}); }
function getAllNutrition()      { return _req(nutTx().getAll()); }
function deleteNutritionEntry(id) { return _req(nutTx('readwrite').delete(id)).then(() => {}); }

function getNutritionByDate(date) {
  return _req(nutTx().index('date').getAll(IDBKeyRange.only(date)));
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Reload `http://localhost:8123/index.html?selftest=1`

Expected: `✅ 22 self-test groups passed`

If the DB upgrade doesn't fire, the browser is holding the old version — close all tabs on `localhost:8123` and reload, or delete the `runlog-db` database in DevTools → Application → IndexedDB.

- [ ] **Step 5: Commit**

```bash
git add runlog-app/index.html
git commit -m "feat(nutrition): add pantry and nutrition IndexedDB stores (v5)"
```

---

### Task 3: Built-in food table and first-run seeding

**Files:**
- Modify: `runlog-app/index.html` — add a section after the Task 2 storage block

**Interfaces:**
- Consumes: `putPantryItem()`, `getAllPantry()`, `getConfig()`, `setConfig()`, `uid()`
- Produces:
  - `BUILTIN_FOODS` — array of partial pantry items (no `id`/timestamps)
  - `BUILTIN_SEED_VERSION` — integer, bump to add foods later
  - `COOKED_FACTORS` — `{category: number}` defaults
  - `seedBuiltinFoods() -> Promise<number>` returning how many were inserted

- [ ] **Step 1: Write the failing test**

Add to `_selfTestAsync` in `runSelfTests()`:

```javascript
  _selfTestAsync.push((async () => {
    await openDB();
    assert(BUILTIN_FOODS.length >= 60, 'built-in table has at least 60 foods');

    // Every built-in food must have a self-consistent macro panel — a typo
    // in this table would silently corrupt every log entry using that food.
    let bad = [];
    for (const f of BUILTIN_FOODS) {
      if (!validateMacroPanel(f.per100g).ok) bad.push(f.name);
      if (!f.name || !f.category) bad.push(f.name + ' (missing name/category)');
    }
    assert(bad.length === 0, 'all built-in panels validate: ' + bad.join(', '));

    // Names must be unique or the picker shows duplicates.
    const names = BUILTIN_FOODS.map(f => f.name.toLowerCase());
    assert(new Set(names).size === names.length, 'built-in food names are unique');

    // Seeding is idempotent.
    await seedBuiltinFoods();
    const afterFirst = (await getAllPantry()).filter(p => p.source === 'builtin').length;
    await seedBuiltinFoods();
    const afterSecond = (await getAllPantry()).filter(p => p.source === 'builtin').length;
    assert(afterFirst === afterSecond, 'seeding twice does not duplicate foods');
    assert(afterFirst === BUILTIN_FOODS.length, 'all built-ins seeded');
  })());
```

- [ ] **Step 2: Run the test to verify it fails**

Reload the self-test URL. Expected: `BUILTIN_FOODS is not defined`.

- [ ] **Step 3: Write the minimal implementation**

```javascript
// ============================================================
// Nutrition — built-in foods
// Common foods that have no package to photograph. Seeded into the
// pantry store on first run so there is exactly one code path for
// looking up a food, and so these stay editable if your eggs are a
// different size than mine.
//
// All values are per 100g "as sold" (raw/dry), matching how packaged
// labels are printed. Bump BUILTIN_SEED_VERSION after adding entries.
// ============================================================

const BUILTIN_SEED_VERSION = 1;

// Grams cooked per gram as-sold. Starches gain water (>1); meat loses
// it (<1). Used as the default when adding a food in that category;
// always editable per item.
const COOKED_FACTORS = {
  pasta: 2.4, rice: 3.0, oats: 3.0, legumes: 2.4, couscous: 3.0, grain: 2.5,
  meat: 0.75, fish: 0.80,
  dairy: null, fat: null, fruit: null, veg: null, nuts: null,
  supplement: null, bread: null, other: null,
};

const BUILTIN_FOODS = [
  // --- eggs & dairy ---
  { name: 'Egg, whole', category: 'other', per100g: { kcal: 143, protein: 12.6, carbs: 0.7, fat: 9.5 }, unit: { label: 'egg', grams: 55 } },
  { name: 'Egg white', category: 'other', per100g: { kcal: 52, protein: 10.9, carbs: 0.7, fat: 0.2 }, unit: { label: 'white', grams: 33 } },
  { name: 'Milk, whole', category: 'dairy', per100g: { kcal: 61, protein: 3.2, carbs: 4.8, fat: 3.3 } },
  { name: 'Milk, semi-skimmed', category: 'dairy', per100g: { kcal: 47, protein: 3.4, carbs: 4.8, fat: 1.7 } },
  { name: 'Greek yoghurt, 0%', category: 'dairy', per100g: { kcal: 59, protein: 10.2, carbs: 3.6, fat: 0.4 } },
  { name: 'Greek yoghurt, full fat', category: 'dairy', per100g: { kcal: 97, protein: 9, carbs: 3.6, fat: 5 } },
  { name: 'Cottage cheese', category: 'dairy', per100g: { kcal: 98, protein: 11, carbs: 3.4, fat: 4.3 } },
  { name: 'Cheddar', category: 'dairy', per100g: { kcal: 403, protein: 25, carbs: 1.3, fat: 33 } },
  { name: 'Mozzarella', category: 'dairy', per100g: { kcal: 280, protein: 28, carbs: 3.1, fat: 17 } },
  { name: 'Parmesan', category: 'dairy', per100g: { kcal: 431, protein: 38, carbs: 4.1, fat: 29 } },
  { name: 'Feta', category: 'dairy', per100g: { kcal: 264, protein: 14, carbs: 4.1, fat: 21 } },
  { name: 'Butter', category: 'fat', per100g: { kcal: 717, protein: 0.9, carbs: 0.1, fat: 81 } },

  // --- meat & fish ---
  { name: 'Chicken breast, raw', category: 'meat', per100g: { kcal: 165, protein: 31, carbs: 0, fat: 3.6 } },
  { name: 'Chicken thigh, raw', category: 'meat', per100g: { kcal: 209, protein: 26, carbs: 0, fat: 11 } },
  { name: 'Beef mince 5%', category: 'meat', per100g: { kcal: 137, protein: 21, carbs: 0, fat: 5 } },
  { name: 'Beef mince 20%', category: 'meat', per100g: { kcal: 254, protein: 17, carbs: 0, fat: 20 } },
  { name: 'Beef steak, sirloin', category: 'meat', per100g: { kcal: 201, protein: 27, carbs: 0, fat: 10 } },
  { name: 'Pork loin', category: 'meat', per100g: { kcal: 143, protein: 21, carbs: 0, fat: 6 } },
  { name: 'Turkey breast', category: 'meat', per100g: { kcal: 157, protein: 29, carbs: 0, fat: 4 } },
  { name: 'Bacon', category: 'meat', per100g: { kcal: 393, protein: 27, carbs: 1.3, fat: 31 } },
  { name: 'Ham, sliced', category: 'meat', per100g: { kcal: 145, protein: 21, carbs: 1.5, fat: 6 } },
  { name: 'Salmon, raw', category: 'fish', per100g: { kcal: 208, protein: 20, carbs: 0, fat: 13 } },
  { name: 'Tuna, canned in water', category: 'fish', per100g: { kcal: 116, protein: 26, carbs: 0, fat: 1 } },
  { name: 'Cod, raw', category: 'fish', per100g: { kcal: 82, protein: 18, carbs: 0, fat: 0.7 } },
  { name: 'Prawns', category: 'fish', per100g: { kcal: 99, protein: 24, carbs: 0.2, fat: 0.3 } },
  { name: 'Sardines, canned', category: 'fish', per100g: { kcal: 208, protein: 25, carbs: 0, fat: 11 } },

  // --- starches ---
  { name: 'Pasta, dry (generic)', category: 'pasta', per100g: { kcal: 352, protein: 12.5, carbs: 71.2, fat: 1.5 } },
  { name: 'Wholewheat pasta, dry', category: 'pasta', per100g: { kcal: 348, protein: 14, carbs: 67, fat: 2.5 } },
  { name: 'White rice, dry', category: 'rice', per100g: { kcal: 360, protein: 6.6, carbs: 80, fat: 0.6 } },
  { name: 'Brown rice, dry', category: 'rice', per100g: { kcal: 367, protein: 7.9, carbs: 76, fat: 2.9 } },
  { name: 'Basmati rice, dry', category: 'rice', per100g: { kcal: 356, protein: 8.5, carbs: 78, fat: 0.9 } },
  { name: 'Couscous, dry', category: 'couscous', per100g: { kcal: 376, protein: 13, carbs: 77, fat: 0.6 } },
  { name: 'Quinoa, dry', category: 'grain', per100g: { kcal: 368, protein: 14, carbs: 64, fat: 6 } },
  { name: 'Rolled oats', category: 'oats', per100g: { kcal: 379, protein: 13, carbs: 68, fat: 6.5 } },
  { name: 'Potato, raw', category: 'veg', per100g: { kcal: 77, protein: 2, carbs: 17, fat: 0.1 } },
  { name: 'Sweet potato, raw', category: 'veg', per100g: { kcal: 86, protein: 1.6, carbs: 20, fat: 0.1 } },
  { name: 'Bread, white', category: 'bread', per100g: { kcal: 265, protein: 9, carbs: 49, fat: 3.2 }, unit: { label: 'slice', grams: 36 } },
  { name: 'Bread, wholemeal', category: 'bread', per100g: { kcal: 247, protein: 13, carbs: 41, fat: 3.4 }, unit: { label: 'slice', grams: 40 } },
  { name: 'Tortilla wrap', category: 'bread', per100g: { kcal: 306, protein: 8.2, carbs: 51, fat: 7.9 }, unit: { label: 'wrap', grams: 62 } },
  { name: 'Bagel', category: 'bread', per100g: { kcal: 250, protein: 10, carbs: 49, fat: 1.5 }, unit: { label: 'bagel', grams: 95 } },

  // --- legumes ---
  { name: 'Lentils, dry', category: 'legumes', per100g: { kcal: 352, protein: 25, carbs: 60, fat: 1.1 } },
  { name: 'Chickpeas, dry', category: 'legumes', per100g: { kcal: 378, protein: 20, carbs: 63, fat: 6 } },
  { name: 'Chickpeas, canned', category: 'legumes', per100g: { kcal: 139, protein: 7.1, carbs: 22, fat: 2.6 } },
  { name: 'Black beans, canned', category: 'legumes', per100g: { kcal: 91, protein: 6, carbs: 16, fat: 0.3 } },
  { name: 'Kidney beans, canned', category: 'legumes', per100g: { kcal: 94, protein: 6.5, carbs: 17, fat: 0.4 } },

  // --- supplements ---
  { name: 'Whey protein isolate', category: 'supplement', per100g: { kcal: 373, protein: 85, carbs: 3.5, fat: 1.5 }, unit: { label: 'scoop', grams: 30 } },
  { name: 'Whey protein concentrate', category: 'supplement', per100g: { kcal: 400, protein: 75, carbs: 8, fat: 6 }, unit: { label: 'scoop', grams: 30 } },
  { name: 'Casein protein', category: 'supplement', per100g: { kcal: 367, protein: 78, carbs: 6, fat: 2 }, unit: { label: 'scoop', grams: 30 } },
  { name: 'Creatine monohydrate', category: 'supplement', per100g: { kcal: 0, protein: 0, carbs: 0, fat: 0 }, unit: { label: 'scoop', grams: 5 } },

  // --- fats, nuts, seeds ---
  { name: 'Olive oil', category: 'fat', per100g: { kcal: 884, protein: 0, carbs: 0, fat: 100 } },
  { name: 'Peanut butter', category: 'nuts', per100g: { kcal: 588, protein: 25, carbs: 20, fat: 50 } },
  { name: 'Almonds', category: 'nuts', per100g: { kcal: 579, protein: 21, carbs: 22, fat: 50 } },
  { name: 'Walnuts', category: 'nuts', per100g: { kcal: 654, protein: 15, carbs: 14, fat: 65 } },
  { name: 'Cashews', category: 'nuts', per100g: { kcal: 553, protein: 18, carbs: 30, fat: 44 } },
  { name: 'Chia seeds', category: 'nuts', per100g: { kcal: 486, protein: 17, carbs: 42, fat: 31 } },
  { name: 'Avocado', category: 'fruit', per100g: { kcal: 160, protein: 2, carbs: 9, fat: 15 } },

  // --- fruit ---
  { name: 'Banana', category: 'fruit', per100g: { kcal: 89, protein: 1.1, carbs: 23, fat: 0.3 }, unit: { label: 'banana', grams: 118 } },
  { name: 'Apple', category: 'fruit', per100g: { kcal: 52, protein: 0.3, carbs: 14, fat: 0.2 }, unit: { label: 'apple', grams: 182 } },
  { name: 'Orange', category: 'fruit', per100g: { kcal: 47, protein: 0.9, carbs: 12, fat: 0.1 }, unit: { label: 'orange', grams: 131 } },
  { name: 'Blueberries', category: 'fruit', per100g: { kcal: 57, protein: 0.7, carbs: 14, fat: 0.3 } },
  { name: 'Strawberries', category: 'fruit', per100g: { kcal: 32, protein: 0.7, carbs: 7.7, fat: 0.3 } },
  { name: 'Grapes', category: 'fruit', per100g: { kcal: 69, protein: 0.7, carbs: 18, fat: 0.2 } },
  { name: 'Mango', category: 'fruit', per100g: { kcal: 60, protein: 0.8, carbs: 15, fat: 0.4 } },
  { name: 'Dates, dried', category: 'fruit', per100g: { kcal: 282, protein: 2.5, carbs: 75, fat: 0.4 }, unit: { label: 'date', grams: 24 } },

  // --- vegetables ---
  { name: 'Broccoli', category: 'veg', per100g: { kcal: 34, protein: 2.8, carbs: 7, fat: 0.4 } },
  { name: 'Spinach', category: 'veg', per100g: { kcal: 23, protein: 2.9, carbs: 3.6, fat: 0.4 } },
  { name: 'Tomato', category: 'veg', per100g: { kcal: 18, protein: 0.9, carbs: 3.9, fat: 0.2 } },
  { name: 'Cucumber', category: 'veg', per100g: { kcal: 15, protein: 0.7, carbs: 3.6, fat: 0.1 } },
  { name: 'Carrot', category: 'veg', per100g: { kcal: 41, protein: 0.9, carbs: 10, fat: 0.2 } },
  { name: 'Onion', category: 'veg', per100g: { kcal: 40, protein: 1.1, carbs: 9.3, fat: 0.1 } },
  { name: 'Bell pepper', category: 'veg', per100g: { kcal: 31, protein: 1, carbs: 6, fat: 0.3 } },
  { name: 'Courgette', category: 'veg', per100g: { kcal: 17, protein: 1.2, carbs: 3.1, fat: 0.3 } },
  { name: 'Mushrooms', category: 'veg', per100g: { kcal: 22, protein: 3.1, carbs: 3.3, fat: 0.3 } },
  { name: 'Lettuce', category: 'veg', per100g: { kcal: 15, protein: 1.4, carbs: 2.9, fat: 0.2 } },
  { name: 'Green beans', category: 'veg', per100g: { kcal: 31, protein: 1.8, carbs: 7, fat: 0.2 } },
  { name: 'Peas, frozen', category: 'veg', per100g: { kcal: 81, protein: 5.4, carbs: 14, fat: 0.4 } },

  // --- other ---
  { name: 'Honey', category: 'other', per100g: { kcal: 304, protein: 0.3, carbs: 82, fat: 0 } },
  { name: 'Dark chocolate 85%', category: 'other', per100g: { kcal: 592, protein: 10, carbs: 24, fat: 46 } },
  { name: 'Hummus', category: 'other', per100g: { kcal: 166, protein: 7.9, carbs: 14, fat: 9.6 } },
  { name: 'Tofu, firm', category: 'other', per100g: { kcal: 144, protein: 17, carbs: 2.8, fat: 8.7 } },
];

// Insert any built-in food not already present. Idempotent: matches on
// lowercased name, so re-running after a version bump adds only the new
// entries and never clobbers a user's edits to an existing one.
async function seedBuiltinFoods() {
  const existing = await getAllPantry();
  const have = new Set(existing.map(p => p.name.toLowerCase()));
  const now = new Date().toISOString();
  let inserted = 0;
  for (const f of BUILTIN_FOODS) {
    if (have.has(f.name.toLowerCase())) continue;
    await putPantryItem({
      id: 'pnt_' + uid(),
      name: f.name,
      category: f.category,
      per100g: f.per100g,
      unit: f.unit || null,
      cookedFactor: COOKED_FACTORS[f.category] ?? null,
      source: 'builtin',
      createdAt: now,
      updatedAt: now,
      archived: false,
    });
    inserted++;
  }
  await setConfig('builtinSeedVersion', BUILTIN_SEED_VERSION);
  return inserted;
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Reload `http://localhost:8123/index.html?selftest=1`

Expected: `✅ 23 self-test groups passed`. If a built-in panel fails validation, the failure message names the food — correct that row's numbers.

- [ ] **Step 5: Commit**

```bash
git add runlog-app/index.html
git commit -m "feat(nutrition): built-in food table with idempotent seeding"
```

---

### Task 4: Food tab and Today view

**Files:**
- Modify: `runlog-app/index.html:1595-1599` (tab bar), after `:1599` (new view section), `:11-1586` (CSS), `:2808` (tab switch handler), and add a UI section after the Task 3 block

**Interfaces:**
- Consumes: `getNutritionByDate()`, `seedBuiltinFoods()`, `getConfig()`, `toast()`
- Produces:
  - `DAILY_KCAL_TARGET`, `DAILY_PROTEIN_TARGET` constants
  - `todayISO() -> 'YYYY-MM-DD'`
  - `sumMacros(entries) -> {kcal, protein, carbs, fat}`
  - `renderFoodToday() -> Promise<void>`
  - `setFoodMode(mode)` where mode is `'today'|'log'|'pantry'`

- [ ] **Step 1: Write the failing test**

Add to `runSelfTests()` (synchronous group):

```javascript
  t(() => {
    const sum = sumMacros([
      { macros: { kcal: 100, protein: 10, carbs: 5, fat: 2 } },
      { macros: { kcal: 250.5, protein: 20.2, carbs: 30, fat: 8 } },
    ]);
    assertClose(sum.kcal, 350.5, 0.01, 'sumMacros totals kcal');
    assertClose(sum.protein, 30.2, 0.01, 'sumMacros totals protein');
  });
  t(() => {
    const z = sumMacros([]);
    assert(z.kcal === 0 && z.protein === 0, 'sumMacros of empty list is all zeros');
  });
  t(() => assert(DAILY_KCAL_TARGET === 2500 && DAILY_PROTEIN_TARGET === 170,
    'daily targets are 2500 kcal / 170g protein'));
  t(() => assert(/^\d{4}-\d{2}-\d{2}$/.test(todayISO()), 'todayISO returns YYYY-MM-DD'));
```

- [ ] **Step 2: Run the test to verify it fails**

Reload the self-test URL. Expected: `sumMacros is not defined`.

- [ ] **Step 3: Write the minimal implementation**

**3a.** Tab bar — replace `index.html:1595-1599`:

```html
  <button class="tab active" data-view="log">Log</button>
  <button class="tab" data-view="food">Food</button>
  <button class="tab" data-view="analyze">Analyze</button>
  <button class="tab" data-view="program">Prog</button>
  <button class="tab" data-view="list">History</button>
  <button class="tab" data-view="settings">Sync</button>
```

**3b.** New view section — insert after the closing `</section>` of `view-settings`:

```html
  <!-- FOOD VIEW -->
  <section class="view" id="view-food">
    <div class="log-mode-toggle">
      <button type="button" class="food-mode-btn active" data-fmode="today">Today</button>
      <button type="button" class="food-mode-btn" data-fmode="log">Log</button>
      <button type="button" class="food-mode-btn" data-fmode="pantry">Pantry</button>
    </div>
    <div id="food-today" class="food-pane"></div>
    <div id="food-log" class="food-pane" style="display:none"></div>
    <div id="food-pantry" class="food-pane" style="display:none"></div>
  </section>
```

**3c.** CSS — insert before `</style>` at `index.html:1586`:

```css
    .food-pane { padding: 0 4px; }
    .macro-ring { display: flex; gap: 12px; margin: 12px 0 18px; }
    .macro-card {
      flex: 1; background: var(--card, #1b1b1f); border-radius: 12px;
      padding: 12px; text-align: center;
    }
    .macro-card .val { font-size: 22px; font-weight: 700; }
    .macro-card .lbl { font-size: 11px; opacity: .6; text-transform: uppercase; letter-spacing: .5px; }
    .macro-card .tgt { font-size: 11px; opacity: .5; margin-top: 2px; }
    .macro-bar { height: 5px; border-radius: 3px; background: rgba(255,255,255,.12); margin-top: 8px; overflow: hidden; }
    .macro-bar > span { display: block; height: 100%; background: #4ade80; transition: width .25s; }
    .macro-bar.over > span { background: #f87171; }
    .meal-group { margin-bottom: 16px; }
    .meal-group h4 {
      font-size: 12px; text-transform: uppercase; letter-spacing: .6px;
      opacity: .55; margin: 0 0 6px 2px;
    }
    .food-entry {
      display: flex; justify-content: space-between; align-items: center;
      padding: 9px 11px; background: var(--card, #1b1b1f);
      border-radius: 9px; margin-bottom: 5px;
    }
    .food-entry .fe-name { font-size: 14px; }
    .food-entry .fe-sub { font-size: 11px; opacity: .55; margin-top: 1px; }
    .food-entry .fe-macros { font-size: 12px; text-align: right; white-space: nowrap; }
```

**3d.** Tab handler — add inside the existing handler at `index.html:2808`, alongside the other `if (tab.dataset.view === …)` lines:

```javascript
    if (tab.dataset.view === 'food') { setFoodMode('today'); }
```

**3e.** The UI section:

```javascript
// ============================================================
// Nutrition — UI
// Three panes behind one tab: Today (totals), Log (add an entry),
// Pantry (manage foods).
// ============================================================

const DAILY_KCAL_TARGET = 2500;
const DAILY_PROTEIN_TARGET = 170;

const MEALS = ['breakfast', 'lunch', 'dinner', 'snack'];

function todayISO() {
  const d = new Date();
  return d.getFullYear() + '-' +
    String(d.getMonth() + 1).padStart(2, '0') + '-' +
    String(d.getDate()).padStart(2, '0');
}

function sumMacros(entries) {
  const out = { kcal: 0, protein: 0, carbs: 0, fat: 0 };
  for (const e of entries) {
    if (!e || !e.macros) continue;
    out.kcal    += e.macros.kcal    || 0;
    out.protein += e.macros.protein || 0;
    out.carbs   += e.macros.carbs   || 0;
    out.fat     += e.macros.fat     || 0;
  }
  for (const k of Object.keys(out)) out[k] = round1(out[k]);
  return out;
}

function setFoodMode(mode) {
  document.querySelectorAll('.food-mode-btn').forEach(b =>
    b.classList.toggle('active', b.dataset.fmode === mode));
  document.getElementById('food-today').style.display  = mode === 'today'  ? '' : 'none';
  document.getElementById('food-log').style.display    = mode === 'log'    ? '' : 'none';
  document.getElementById('food-pantry').style.display = mode === 'pantry' ? '' : 'none';
  if (mode === 'today')  renderFoodToday();
  if (mode === 'log')    renderFoodLog();
  if (mode === 'pantry') renderFoodPantry();
}

document.querySelectorAll('.food-mode-btn').forEach(btn => {
  btn.addEventListener('click', () => setFoodMode(btn.dataset.fmode));
});

function macroCard(label, value, target, unit) {
  const pct = target ? Math.min(100, (value / target) * 100) : 0;
  const over = target && value > target;
  return '<div class="macro-card">' +
    '<div class="val">' + Math.round(value) + '</div>' +
    '<div class="lbl">' + label + '</div>' +
    (target ? '<div class="tgt">of ' + target + unit + '</div>' +
      '<div class="macro-bar' + (over ? ' over' : '') + '"><span style="width:' + pct + '%"></span></div>'
      : '<div class="tgt">' + unit + '</div>') +
    '</div>';
}

async function renderFoodToday() {
  const el = document.getElementById('food-today');
  const date = todayISO();
  const entries = await getNutritionByDate(date);
  const total = sumMacros(entries);

  let html = '<div class="macro-ring">' +
    macroCard('kcal', total.kcal, DAILY_KCAL_TARGET, '') +
    macroCard('protein', total.protein, DAILY_PROTEIN_TARGET, 'g') +
    '</div>' +
    '<div class="macro-ring">' +
    macroCard('carbs', total.carbs, null, 'g') +
    macroCard('fat', total.fat, null, 'g') +
    '</div>';

  if (entries.length === 0) {
    html += '<div class="empty-state"><div class="icon">🍽️</div>' +
      '<div>Nothing logged today.</div></div>';
  } else {
    for (const meal of MEALS) {
      const forMeal = entries.filter(e => e.meal === meal)
        .sort((a, b) => (a.createdAt || '').localeCompare(b.createdAt || ''));
      if (!forMeal.length) continue;
      const m = sumMacros(forMeal);
      html += '<div class="meal-group"><h4>' + meal + ' — ' +
        Math.round(m.kcal) + ' kcal, ' + Math.round(m.protein) + 'g P</h4>';
      for (const e of forMeal) {
        const amt = e.mode === 'units'
          ? e.amount + '×'
          : Math.round(e.amount) + 'g ' + e.mode;
        html += '<div class="food-entry" data-entry-id="' + e.id + '">' +
          '<div><div class="fe-name">' + escapeHtml(e.foodName) + '</div>' +
          '<div class="fe-sub">' + amt + '</div></div>' +
          '<div class="fe-macros">' + Math.round(e.macros.kcal) + ' kcal<br>' +
          '<span style="opacity:.6">' + Math.round(e.macros.protein) + 'g P</span></div>' +
          '</div>';
      }
      html += '</div>';
    }
  }
  el.innerHTML = html;

  // Tap an entry to delete it (with confirm).
  el.querySelectorAll('.food-entry').forEach(row => {
    row.addEventListener('click', async () => {
      if (!confirm('Delete this entry?')) return;
      await deleteNutritionEntry(row.dataset.entryId);
      toast('Entry deleted');
      renderFoodToday();
      nutritionSyncToGh().catch(e => console.warn('[nutrition sync]', e));
    });
  });
}

// Minimal HTML escape for user-supplied food names.
function escapeHtml(s) {
  return String(s == null ? '' : s)
    .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;').replace(/'/g, '&#39;');
}
```

**3f.** Seed on startup — in the app-init IIFE that begins at `index.html:7571`, insert immediately after the `await loadWhoopTokens();` line (`index.html:7576`) and before the VO2 migration comment:

```javascript
  if ((await getConfig('builtinSeedVersion')) !== BUILTIN_SEED_VERSION) {
    const n = await seedBuiltinFoods();
    if (n > 0) console.log('[nutrition] seeded ' + n + ' built-in foods');
  }
```

> `renderFoodLog`, `renderFoodPantry`, and `nutritionSyncToGh` are defined in Tasks 5, 6, and 7. Until then, add temporary no-op stubs directly below `setFoodMode` so the Today pane is testable in isolation:
> ```javascript
> async function renderFoodLog() {}
> async function renderFoodPantry() {}
> async function nutritionSyncToGh() { return { ok: true }; }
> ```
> Delete each stub in the task that implements it for real.

- [ ] **Step 4: Run the tests and check the view**

```bash
open 'http://localhost:8123/index.html?selftest=1'   # expect 27 groups passed
open 'http://localhost:8123/index.html'              # tap the Food tab
```

Expected: `✅ 27 self-test groups passed`, and the Food tab shows four macro cards reading 0 with an empty state below. Confirm all six tabs still fit on a narrow window (resize to 380px).

- [ ] **Step 5: Commit**

```bash
git add runlog-app/index.html
git commit -m "feat(nutrition): Food tab with Today totals view"
```

---

### Task 5: Log-entry form

**Files:**
- Modify: `runlog-app/index.html` — replace the `renderFoodLog` stub; add CSS before `</style>`

**Interfaces:**
- Consumes: `getAllPantry()`, `putNutritionEntry()`, `resolveGrams()`, `computeMacros()`, `uid()`, `toast()`, `todayISO()`, `escapeHtml()`
- Produces:
  - `renderFoodLog() -> Promise<void>`
  - `buildEntry(item, amount, mode, meal, date) -> Entry|null` — the pure part, testable

- [ ] **Step 1: Write the failing test**

Add to `runSelfTests()` (synchronous group):

```javascript
  t(() => {
    const penne = {
      id: 'pnt_x', name: 'Penne', category: 'pasta',
      per100g: { kcal: 352, protein: 12.5, carbs: 71.2, fat: 1.5 },
      cookedFactor: 2.4, unit: null,
    };
    const e = buildEntry(penne, 120, 'dry', 'lunch', '2026-08-06');
    assert(e !== null, 'buildEntry returns an entry for valid input');
    assert(e.foodId === 'pnt_x' && e.foodName === 'Penne', 'entry snapshots food identity');
    assert(e.grams === 120, 'entry records resolved grams');
    assertClose(e.macros.kcal, 422.4, 0.05, 'entry macros computed');
    assert(e.date === '2026-08-06' && e.meal === 'lunch', 'entry records date and meal');
    assert(typeof e.id === 'string' && e.id.startsWith('ent_'), 'entry has an ent_ id');
    assert(!('syncState' in e), 'entry has no syncState (whole-file sync model)');
  });
  t(() => {
    const penne = { id: 'p', name: 'Penne', per100g: { kcal: 352, protein: 12.5, carbs: 71.2, fat: 1.5 }, cookedFactor: 2.4, unit: null };
    assert(buildEntry(penne, -1, 'dry', 'lunch', '2026-08-06') === null,
      'buildEntry rejects negative amount');
    assert(buildEntry(penne, 100, 'units', 'lunch', '2026-08-06') === null,
      'buildEntry rejects units mode without a unit');
    assert(buildEntry(penne, 100, 'dry', 'brunch', '2026-08-06') === null,
      'buildEntry rejects an unknown meal');
  });
```

- [ ] **Step 2: Run the test to verify it fails**

Reload the self-test URL. Expected: `buildEntry is not defined`.

- [ ] **Step 3: Write the minimal implementation**

Replace the `renderFoodLog` stub with:

```javascript
// Build a log entry from a pantry item and user input. Pure — no storage,
// no DOM — so the resolve/compute/snapshot chain is covered by self-tests.
// Returns null on any invalid input rather than writing a bad record.
function buildEntry(item, amount, mode, meal, date) {
  if (!item || !MEALS.includes(meal)) return null;
  if (!/^\d{4}-\d{2}-\d{2}$/.test(date || '')) return null;
  const grams = resolveGrams(item, amount, mode);
  if (grams === null) return null;
  const macros = computeMacros(item, grams);
  if (macros === null) return null;
  return {
    id: 'ent_' + uid(),
    date, meal,
    foodId: item.id,
    foodName: item.name,
    amount, mode,
    grams: round1(grams),
    macros,
    createdAt: new Date().toISOString(),
  };
}

let _logSelectedFood = null;

async function renderFoodLog() {
  const el = document.getElementById('food-log');
  const pantry = (await getAllPantry())
    .filter(p => !p.archived)
    .sort((a, b) => a.name.localeCompare(b.name));

  el.innerHTML =
    '<div class="field">' +
      '<label for="fl-search">Food</label>' +
      '<input type="text" id="fl-search" placeholder="Search your pantry…" ' +
        'autocomplete="off" autocapitalize="off" autocorrect="off" spellcheck="false">' +
      '<div id="fl-results" class="fl-results"></div>' +
    '</div>' +
    '<div id="fl-form" style="display:none">' +
      '<div class="fl-chosen" id="fl-chosen"></div>' +
      '<div class="log-mode-toggle" id="fl-modes"></div>' +
      '<div class="field">' +
        '<label for="fl-amount" id="fl-amount-label">Amount (g)</label>' +
        '<input type="number" id="fl-amount" inputmode="decimal" step="any" min="0" placeholder="120">' +
      '</div>' +
      '<div class="field">' +
        '<label for="fl-meal">Meal</label>' +
        '<select id="fl-meal">' +
          MEALS.map(m => '<option value="' + m + '">' + m + '</option>').join('') +
        '</select>' +
      '</div>' +
      '<div class="fl-preview" id="fl-preview"></div>' +
      '<button class="primary" id="fl-save">Add to today</button>' +
    '</div>';

  const search  = document.getElementById('fl-search');
  const results = document.getElementById('fl-results');
  const form    = document.getElementById('fl-form');

  function renderResults(q) {
    const needle = q.trim().toLowerCase();
    const hits = needle
      ? pantry.filter(p => p.name.toLowerCase().includes(needle)).slice(0, 12)
      : pantry.slice(0, 12);
    results.innerHTML = hits.map(p =>
      '<button type="button" class="fl-hit" data-id="' + p.id + '">' +
      escapeHtml(p.name) +
      '<span class="fl-hit-sub">' + Math.round(p.per100g.kcal) + ' kcal / 100g</span>' +
      '</button>').join('') ||
      '<div class="fl-empty">No match. Add it in the Pantry tab.</div>';
    results.querySelectorAll('.fl-hit').forEach(b => {
      b.addEventListener('click', () => selectFood(pantry.find(p => p.id === b.dataset.id)));
    });
  }

  function selectFood(item) {
    _logSelectedFood = item;
    document.getElementById('fl-chosen').textContent = item.name;
    results.innerHTML = '';
    search.value = '';
    form.style.display = '';

    // Only offer modes this food actually supports.
    const modes = [{ k: 'dry', label: item.cookedFactor ? 'Dry' : 'Weight' }];
    if (item.cookedFactor) modes.push({ k: 'cooked', label: 'Cooked' });
    if (item.unit) modes.push({ k: 'units', label: item.unit.label });
    const box = document.getElementById('fl-modes');
    box.innerHTML = modes.map((m, i) =>
      '<button type="button" class="food-mode-btn' + (i === 0 ? ' active' : '') +
      '" data-lmode="' + m.k + '">' + escapeHtml(m.label) + '</button>').join('');
    box.querySelectorAll('.food-mode-btn').forEach(b => {
      b.addEventListener('click', () => {
        box.querySelectorAll('.food-mode-btn').forEach(x => x.classList.remove('active'));
        b.classList.add('active');
        document.getElementById('fl-amount-label').textContent =
          b.dataset.lmode === 'units' ? 'How many ' + item.unit.label + '?' : 'Amount (g)';
        updatePreview();
      });
    });
    document.getElementById('fl-amount').value = '';
    updatePreview();
  }

  function currentMode() {
    const on = document.querySelector('#fl-modes .food-mode-btn.active');
    return on ? on.dataset.lmode : 'dry';
  }

  function updatePreview() {
    const prev = document.getElementById('fl-preview');
    const amt = parseFloat(document.getElementById('fl-amount').value);
    if (!_logSelectedFood || isNaN(amt)) { prev.textContent = ''; return; }
    const g = resolveGrams(_logSelectedFood, amt, currentMode());
    const m = g === null ? null : computeMacros(_logSelectedFood, g);
    prev.textContent = m
      ? Math.round(m.kcal) + ' kcal · ' + m.protein + 'g P · ' +
        m.carbs + 'g C · ' + m.fat + 'g F'
      : 'Enter a valid amount';
  }

  search.addEventListener('input', () => renderResults(search.value));
  document.getElementById('fl-amount').addEventListener('input', updatePreview);

  document.getElementById('fl-save').addEventListener('click', async () => {
    const amt = parseFloat(document.getElementById('fl-amount').value);
    const entry = buildEntry(_logSelectedFood, amt, currentMode(),
      document.getElementById('fl-meal').value, todayISO());
    if (!entry) { toast('Enter a valid amount', true); return; }
    await putNutritionEntry(entry);
    toast('Logged ' + entry.foodName);
    _logSelectedFood = null;
    form.style.display = 'none';
    setFoodMode('today');
    nutritionSyncToGh().catch(e => console.warn('[nutrition sync]', e));
  });

  renderResults('');
}
```

CSS before `</style>`:

```css
    .fl-results { margin-top: 6px; }
    .fl-hit {
      display: flex; justify-content: space-between; align-items: center; width: 100%;
      text-align: left; padding: 9px 11px; margin-bottom: 4px;
      background: var(--card, #1b1b1f); border: none; border-radius: 8px;
      color: inherit; font-size: 14px; cursor: pointer;
    }
    .fl-hit-sub { font-size: 11px; opacity: .5; }
    .fl-empty { padding: 10px; font-size: 13px; opacity: .55; }
    .fl-chosen { font-size: 17px; font-weight: 600; margin: 10px 0 8px 2px; }
    .fl-preview { font-size: 13px; opacity: .7; margin: 10px 0 14px 2px; min-height: 18px; }
```

- [ ] **Step 4: Run the tests and exercise the form**

```bash
open 'http://localhost:8123/index.html?selftest=1'   # expect 29 groups passed
```

Then in the app: Food → Log → search "penne" → pick it → enter 120 → Dry → Lunch → Add. Confirm Today shows ~422 kcal and 15 g protein under Lunch. Repeat with Cooked at 288 g and confirm it produces the same numbers.

- [ ] **Step 5: Commit**

```bash
git add runlog-app/index.html
git commit -m "feat(nutrition): log-entry form with dry/cooked/units modes"
```

---

### Task 6: Pantry management — the single write path

**Files:**
- Modify: `runlog-app/index.html` — replace the `renderFoodPantry` stub; add CSS

**Interfaces:**
- Consumes: `getAllPantry()`, `putPantryItem()`, `deletePantryItem()`, `validateMacroPanel()`, `COOKED_FACTORS`, `uid()`, `toast()`, `escapeHtml()`
- Produces:
  - `renderFoodPantry() -> Promise<void>`
  - `openPantryForm(item|null, prefill|null) -> void` — `prefill` is the shape the vision endpoint returns (Task 9 passes it)
  - `readPantryForm() -> {item, issues}` — reads and validates the form

- [ ] **Step 1: Write the failing test**

```javascript
  _selfTestAsync.push((async () => {
    await openDB();
    // openPantryForm must accept a vision-endpoint prefill and populate fields.
    setFoodMode('pantry');
    await renderFoodPantry();
    openPantryForm(null, {
      name: 'Test Prefill Pasta', category: 'pasta',
      per100g: { kcal: 352, protein: 12.5, carbs: 71.2, fat: 1.5 },
    });
    assert(document.getElementById('pf-name').value === 'Test Prefill Pasta',
      'prefill populates the name field');
    assertClose(parseFloat(document.getElementById('pf-kcal').value), 352, 0.01,
      'prefill populates kcal');
    assertClose(parseFloat(document.getElementById('pf-cooked').value), 2.4, 0.01,
      'prefill applies the category cooked factor');
    const read = readPantryForm();
    assert(read.item && read.item.name === 'Test Prefill Pasta',
      'readPantryForm returns the edited item');
    assert(read.issues.length === 0, 'a valid prefill produces no issues');
    document.getElementById('pf-cancel').click();
  })());
```

- [ ] **Step 2: Run the test to verify it fails**

Reload the self-test URL. Expected: `openPantryForm is not defined`.

- [ ] **Step 3: Write the minimal implementation**

Replace the `renderFoodPantry` stub with:

```javascript
async function renderFoodPantry() {
  const el = document.getElementById('food-pantry');
  const pantry = (await getAllPantry())
    .filter(p => !p.archived)
    .sort((a, b) => a.name.localeCompare(b.name));

  el.innerHTML =
    '<div class="pantry-actions">' +
      '<button class="primary" id="pf-add-photo">📷 Add from label</button>' +
      '<button id="pf-add-manual">Add manually</button>' +
    '</div>' +
    '<input type="file" id="pf-file" accept="image/*" capture="environment" style="display:none">' +
    '<div id="pf-form-wrap" style="display:none"></div>' +
    '<div class="field"><input type="text" id="pf-filter" placeholder="Filter…" ' +
      'autocomplete="off" autocapitalize="off" spellcheck="false"></div>' +
    '<div id="pf-list"></div>';

  function drawList(q) {
    const needle = (q || '').trim().toLowerCase();
    const hits = needle ? pantry.filter(p => p.name.toLowerCase().includes(needle)) : pantry;
    document.getElementById('pf-list').innerHTML = hits.map(p =>
      '<div class="food-entry" data-pid="' + p.id + '">' +
      '<div><div class="fe-name">' + escapeHtml(p.name) + '</div>' +
      '<div class="fe-sub">' + p.category +
        (p.unit ? ' · 1 ' + escapeHtml(p.unit.label) + ' = ' + p.unit.grams + 'g' : '') +
        (p.cookedFactor ? ' · ×' + p.cookedFactor + ' cooked' : '') +
      '</div></div>' +
      '<div class="fe-macros">' + Math.round(p.per100g.kcal) + ' kcal<br>' +
      '<span style="opacity:.6">' + p.per100g.protein + 'g P</span></div>' +
      '</div>').join('') || '<div class="fl-empty">No foods match.</div>';
    document.getElementById('pf-list').querySelectorAll('.food-entry').forEach(row => {
      row.addEventListener('click', () =>
        openPantryForm(pantry.find(p => p.id === row.dataset.pid), null));
    });
  }

  document.getElementById('pf-filter').addEventListener('input', e => drawList(e.target.value));
  document.getElementById('pf-add-manual').addEventListener('click', () => openPantryForm(null, null));
  document.getElementById('pf-add-photo').addEventListener('click', () =>
    document.getElementById('pf-file').click());
  // The file input's change handler is wired in Task 9.
  drawList('');
}

// The single write path into the pantry. `item` edits an existing food;
// `prefill` seeds a new one from the vision endpoint. Both may be null
// for a blank manual entry.
function openPantryForm(item, prefill) {
  const wrap = document.getElementById('pf-form-wrap');
  const src = item || prefill || {};
  const p = (src.per100g) || {};
  const cat = src.category || 'other';
  const cooked = item ? item.cookedFactor : (COOKED_FACTORS[cat] ?? null);
  const flagged = prefill && prefill._issues && prefill._issues.length;

  wrap.style.display = '';
  wrap.innerHTML =
    (flagged ? '<div class="pf-warn">These numbers looked inconsistent on the label ' +
      '(' + prefill._issues.join(', ') + '). Please check them.</div>' : '') +
    '<div class="field"><label for="pf-name">Name</label>' +
      '<input type="text" id="pf-name" value="' + escapeHtml(src.name || '') + '"></div>' +
    '<div class="field"><label for="pf-category">Category</label><select id="pf-category">' +
      Object.keys(COOKED_FACTORS).map(c =>
        '<option value="' + c + '"' + (c === cat ? ' selected' : '') + '>' + c + '</option>').join('') +
    '</select></div>' +
    '<div class="pf-grid">' +
      pfNum('pf-kcal', 'kcal /100g', p.kcal) +
      pfNum('pf-protein', 'Protein g', p.protein) +
      pfNum('pf-carbs', 'Carbs g', p.carbs) +
      pfNum('pf-fat', 'Fat g', p.fat) +
    '</div>' +
    '<div class="pf-grid">' +
      pfNum('pf-cooked', 'Cooked factor', cooked) +
      pfNum('pf-unitgrams', 'Unit weight g', item && item.unit ? item.unit.grams : null) +
    '</div>' +
    '<div class="field"><label for="pf-unitlabel">Unit name (e.g. egg, scoop)</label>' +
      '<input type="text" id="pf-unitlabel" value="' +
      escapeHtml(item && item.unit ? item.unit.label : '') + '"></div>' +
    '<button class="primary" id="pf-save">Save food</button>' +
    '<div class="btn-spacer"></div>' +
    '<button id="pf-cancel">Cancel</button>' +
    (item ? '<div class="btn-spacer"></div><button id="pf-delete">Delete this food</button>' : '');

  wrap.dataset.editingId = item ? item.id : '';

  document.getElementById('pf-cancel').addEventListener('click', () => {
    wrap.style.display = 'none'; wrap.innerHTML = '';
  });
  document.getElementById('pf-save').addEventListener('click', async () => {
    const { item: built, issues } = readPantryForm();
    if (!built) { toast(issues[0] || 'Check the values', true); return; }
    if (issues.length && !confirm('These numbers look inconsistent (' +
        issues.join(', ') + '). Save anyway?')) return;
    await putPantryItem(built);
    toast('Saved ' + built.name);
    wrap.style.display = 'none'; wrap.innerHTML = '';
    renderFoodPantry();
    pantrySyncToGh().catch(e => console.warn('[pantry sync]', e));
  });
  const del = document.getElementById('pf-delete');
  if (del) del.addEventListener('click', async () => {
    if (!confirm('Delete this food? Past log entries keep their own copy of the numbers.')) return;
    await deletePantryItem(item.id);
    toast('Deleted');
    wrap.style.display = 'none'; wrap.innerHTML = '';
    renderFoodPantry();
    pantrySyncToGh().catch(e => console.warn('[pantry sync]', e));
  });
}

function pfNum(id, label, val) {
  return '<div class="field"><label for="' + id + '">' + label + '</label>' +
    '<input type="number" id="' + id + '" inputmode="decimal" step="any" min="0" value="' +
    (val === null || val === undefined ? '' : val) + '"></div>';
}

// Read and validate the pantry form. Returns {item, issues}; item is null
// when the input is unusable, non-empty issues means "suspicious but saveable".
function readPantryForm() {
  const wrap = document.getElementById('pf-form-wrap');
  const num = id => {
    const v = parseFloat(document.getElementById(id).value);
    return isNaN(v) ? null : v;
  };
  const name = document.getElementById('pf-name').value.trim();
  const per100g = {
    kcal: num('pf-kcal'), protein: num('pf-protein'),
    carbs: num('pf-carbs'), fat: num('pf-fat'),
  };
  if (!name) return { item: null, issues: ['Name is required'] };
  for (const k of ['kcal', 'protein', 'carbs', 'fat']) {
    if (per100g[k] === null) return { item: null, issues: [k + ' is required'] };
  }
  const unitLabel  = document.getElementById('pf-unitlabel').value.trim();
  const unitGrams  = num('pf-unitgrams');
  const editingId  = wrap.dataset.editingId;
  const now = new Date().toISOString();
  const v = validateMacroPanel(per100g);

  return {
    item: {
      id: editingId || ('pnt_' + uid()),
      name,
      category: document.getElementById('pf-category').value,
      per100g,
      unit: (unitLabel && unitGrams) ? { label: unitLabel, grams: unitGrams } : null,
      cookedFactor: num('pf-cooked'),
      source: editingId ? 'manual' : 'manual',
      createdAt: now,
      updatedAt: now,
      archived: false,
    },
    issues: v.ok ? [] : v.issues,
  };
}
```

CSS before `</style>`:

```css
    .pantry-actions { display: flex; gap: 8px; margin-bottom: 14px; }
    .pantry-actions button { flex: 1; }
    .pf-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
    .pf-warn {
      background: rgba(248,113,113,.14); border: 1px solid rgba(248,113,113,.35);
      border-radius: 8px; padding: 10px 12px; font-size: 13px; margin-bottom: 12px;
    }
```

Also add the temporary stub near the others (removed in Task 7):

```javascript
async function pantrySyncToGh() { return { ok: true }; }
```

- [ ] **Step 4: Run the tests and exercise the form**

```bash
open 'http://localhost:8123/index.html?selftest=1'   # expect 30 groups passed
```

Then: Food → Pantry → Add manually → enter a food with deliberately wrong kcal (e.g. 100 kcal / 50 P / 50 C / 50 F) → Save → confirm the inconsistency prompt appears. Then edit a built-in food and confirm the change persists after reload.

- [ ] **Step 5: Commit**

```bash
git add runlog-app/index.html
git commit -m "feat(nutrition): pantry list and add/edit form"
```

---

### Task 7: GitHub sync for pantry.json and nutrition.json

**Files:**
- Modify: `runlog-app/index.html` — replace both sync stubs; add a section after the UI block

**Interfaces:**
- Consumes: `ghConfig`, `ghHeaders()`, `ghGetFileSha()`, `ghGetFileShaViaTree()` (all existing, `index.html:2368-2470`), `getAllPantry()`, `getAllNutrition()`, `getConfig()`, `setConfig()`
- Produces:
  - `pantrySyncToGh() -> Promise<{ok, error?}>`
  - `nutritionSyncToGh() -> Promise<{ok, error?}>`
  - `retryNutritionSyncIfDirty() -> Promise<void>` — called on app load

- [ ] **Step 1: Write the failing test**

```javascript
  _selfTestAsync.push((async () => {
    await openDB();
    // Without GitHub configured, sync must fail cleanly rather than throw —
    // the UI calls these fire-and-forget on every mutation.
    const savedCfg = ghConfig;
    ghConfig = null;
    const r1 = await pantrySyncToGh();
    const r2 = await nutritionSyncToGh();
    assert(r1.ok === false && r1.reason === 'no_github_config',
      'pantrySyncToGh fails cleanly with no config');
    assert(r2.ok === false && r2.reason === 'no_github_config',
      'nutritionSyncToGh fails cleanly with no config');
    ghConfig = savedCfg;
  })());
```

- [ ] **Step 2: Run the test to verify it fails**

Reload the self-test URL. Expected: the stub returns `{ok: true}`, so both assertions fail.

- [ ] **Step 3: Write the minimal implementation**

Delete the two stubs and add:

```javascript
// ============================================================
// Nutrition — GitHub sync
// Whole-file push of complete local state, mirroring vo2SyncToGh().
// Deliberately NOT the sessions model: syncAllUnsynced() walks
// getAllSessions() and would never see these stores.
//
// Because each push carries complete state, a failure is self-healing on
// the next push. The one gap is a failure with no subsequent mutation, so
// a dirty flag in the config store triggers a retry on next app load.
// ============================================================

async function _ghPutJson(path, payload, message) {
  if (!ghConfig) return { ok: false, reason: 'no_github_config' };
  try {
    const url = 'https://api.github.com/repos/' + ghConfig.owner + '/' +
      ghConfig.repo + '/contents/' + path;
    const body = {
      message,
      content: btoa(unescape(encodeURIComponent(JSON.stringify(payload, null, 2)))),
    };
    const existingSha = await ghGetFileSha(path);
    if (existingSha) body.sha = existingSha;
    let res = await fetch(url, {
      method: 'PUT',
      headers: { ...ghHeaders(), 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    // Contents API occasionally reports a stale sha; fall back to the Trees API.
    if (res.status === 409 || res.status === 422) {
      const fresh = await ghGetFileShaViaTree(path);
      if (fresh) {
        body.sha = fresh;
        res = await fetch(url, {
          method: 'PUT',
          headers: { ...ghHeaders(), 'Content-Type': 'application/json' },
          body: JSON.stringify(body),
        });
      }
    }
    if (!res.ok) {
      const err = await res.text();
      return { ok: false, error: 'PUT ' + path + ': ' + res.status + ' ' + err.slice(0, 100) };
    }
    return { ok: true };
  } catch (err) {
    return { ok: false, error: err.message };
  }
}

async function pantrySyncToGh() {
  const items = (await getAllPantry()).sort((a, b) => a.name.localeCompare(b.name));
  const r = await _ghPutJson('pantry.json', items,
    'update: pantry (' + items.length + ' foods)');
  await setConfig('pantryDirty', !r.ok);
  return r;
}

async function nutritionSyncToGh() {
  const entries = (await getAllNutrition())
    .sort((a, b) => (a.date + a.createdAt).localeCompare(b.date + b.createdAt));
  const r = await _ghPutJson('nutrition.json', entries,
    'update: nutrition (' + entries.length + ' entries)');
  await setConfig('nutritionDirty', !r.ok);
  return r;
}

// Called on app load. Retries whichever file failed to push last session.
async function retryNutritionSyncIfDirty() {
  if (!ghConfig) return;
  if (await getConfig('pantryDirty'))    await pantrySyncToGh().catch(() => {});
  if (await getConfig('nutritionDirty')) await nutritionSyncToGh().catch(() => {});
}
```

Add to the app-init IIFE immediately after the seeding call added in Task 4. It
must come after `await loadGhConfig()` (`index.html:7575`) because it reads
`ghConfig`:

```javascript
  retryNutritionSyncIfDirty().catch(e => console.warn('[nutrition retry]', e));
```

- [ ] **Step 4: Run the tests and verify a real push**

```bash
open 'http://localhost:8123/index.html?selftest=1'   # expect 31 groups passed
```

Then with GitHub sync configured in the Sync tab: log a meal, then confirm both files landed.

```bash
cd ../runlog-data && git pull -q && \
  python3 -c "import json;d=json.load(open('pantry.json'));print('pantry:',len(d),'foods')" && \
  python3 -c "import json;d=json.load(open('nutrition.json'));print('nutrition:',len(d),'entries')"
```

Expected: pantry ≈ 80 foods, nutrition ≥ 1 entry.

- [ ] **Step 5: Commit**

```bash
git add runlog-app/index.html
git commit -m "feat(nutrition): sync pantry.json and nutrition.json to runlog-data"
```

---

### Task 8: Vision endpoint in runlog-auth

**Files:**
- Create: `runlog-auth/api/nutrition-label.js`
- Modify: `runlog-auth/README.md`

**Interfaces:**
- Consumes: `ANTHROPIC_API_KEY`, `ALLOWED_ORIGIN` (Vercel env vars)
- Produces: `POST /api/nutrition-label` accepting `{image: base64, media_type: string}` and returning `{readable, reason, name, category, per100g:{kcal,protein,carbs,fat}}`

- [ ] **Step 1: Write the failing test**

There is no test runner in `runlog-auth`. The test is a curl script — create `runlog-auth/test-nutrition-label.sh`:

```bash
#!/usr/bin/env bash
# Manual verification for /api/nutrition-label.
# Usage: BASE=http://localhost:3000 ./test-nutrition-label.sh path/to/label.jpg
set -uo pipefail
BASE="${BASE:-http://localhost:3000}"
IMG="${1:?usage: $0 <label-image>}"
fail=0
check() { if [ "$2" = "$3" ]; then echo "  PASS $1"; else echo "  FAIL $1 (got $2, want $3)"; fail=1; fi; }

echo "1. OPTIONS preflight"
code=$(curl -s -o /dev/null -w '%{http_code}' -X OPTIONS "$BASE/api/nutrition-label")
check "returns 204" "$code" "204"

echo "2. GET is rejected"
code=$(curl -s -o /dev/null -w '%{http_code}' "$BASE/api/nutrition-label")
check "returns 405" "$code" "405"

echo "3. missing body"
code=$(curl -s -o /dev/null -w '%{http_code}' -X POST "$BASE/api/nutrition-label" \
  -H 'Content-Type: application/json' -d '{}')
check "returns 400" "$code" "400"

echo "4. real label photo"
b64=$(base64 < "$IMG" | tr -d '\n')
resp=$(curl -s -X POST "$BASE/api/nutrition-label" -H 'Content-Type: application/json' \
  -d "{\"image\":\"$b64\",\"media_type\":\"image/jpeg\"}")
echo "$resp" | python3 -c '
import json,sys
d=json.load(sys.stdin)
assert "readable" in d, "missing readable"
if d["readable"]:
    p=d["per100g"]
    for k in ("kcal","protein","carbs","fat"):
        assert isinstance(p[k],(int,float)), k+" not numeric"
    derived=4*p["protein"]+4*p["carbs"]+9*p["fat"]
    print("  PASS readable:", d["name"], p, "derived kcal", round(derived,1))
else:
    print("  PASS unreadable, reason:", d.get("reason"))
' || fail=1

exit $fail
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd runlog-auth && chmod +x test-nutrition-label.sh && npx vercel dev --listen 3000 &
sleep 8 && ./test-nutrition-label.sh ~/Desktop/label.jpg
```

Expected: every check FAILs with 404 — the endpoint does not exist.

- [ ] **Step 3: Write the minimal implementation**

Create `runlog-auth/api/nutrition-label.js`:

```javascript
// api/nutrition-label.js
// Reads a product's nutrition label from a photo and returns per-100g macros.
//
// Exists in this service for the same reason oauth-exchange does: it holds a
// secret (ANTHROPIC_API_KEY) that must not reach the browser. It stores
// nothing and computes nothing beyond normalising the model's answer.
//
// The response is schema-constrained via output_config.format, so the client
// gets parseable JSON rather than prose it has to extract.

const ANTHROPIC_URL = "https://api.anthropic.com/v1/messages";
const MODEL = "claude-haiku-4-5";
const MAX_IMAGE_BYTES = 5 * 1024 * 1024;
const ALLOWED_ORIGIN = process.env.ALLOWED_ORIGIN || "*";

const PANEL_SCHEMA = {
  type: "object",
  properties: {
    readable: { type: "boolean" },
    reason: { type: ["string", "null"] },
    name: { type: ["string", "null"] },
    category: {
      type: ["string", "null"],
      enum: ["pasta", "rice", "oats", "legumes", "couscous", "grain", "meat",
             "fish", "dairy", "fat", "fruit", "veg", "nuts", "supplement",
             "bread", "other", null],
    },
    per100g: {
      type: ["object", "null"],
      properties: {
        kcal:    { type: "number" },
        protein: { type: "number" },
        carbs:   { type: "number" },
        fat:     { type: "number" },
      },
      required: ["kcal", "protein", "carbs", "fat"],
      additionalProperties: false,
    },
  },
  required: ["readable", "reason", "name", "category", "per100g"],
  additionalProperties: false,
};

const PROMPT = [
  "Read the nutrition information panel in this photo.",
  "",
  "Return values PER 100 GRAMS of the product as sold. Critical rules:",
  "- If the panel lists per-serving values, convert to per-100g using the",
  "  stated serving size. If both are shown, use the per-100g column.",
  "- If energy is given in kJ only, convert: kcal = kJ / 4.184.",
  "- Use the product as sold (dry/raw), not as prepared, when both appear.",
  "- Report carbohydrate as total carbohydrate, not 'of which sugars'.",
  "",
  "Set readable=false with a short reason if the panel is blurry, cropped,",
  "absent, or you are otherwise unsure. Never guess or infer typical values",
  "for the product type - a wrong number is far worse than no number.",
  "",
  "name: the product name including brand if visible.",
  "category: best fit from the enum, or null.",
].join("\n");

function corsHeaders() {
  return {
    "Access-Control-Allow-Origin": ALLOWED_ORIGIN,
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type",
    "Access-Control-Max-Age": "86400",
  };
}

export default async function handler(req, res) {
  const setCors = () =>
    Object.entries(corsHeaders()).forEach(([k, v]) => res.setHeader(k, v));

  if (req.method === "OPTIONS") { setCors(); return res.status(204).end(); }
  if (req.method !== "POST") {
    setCors();
    return res.status(405).json({ error: "method_not_allowed" });
  }

  const apiKey = process.env.ANTHROPIC_API_KEY;
  if (!apiKey) {
    setCors();
    return res.status(500).json({ error: "server_not_configured" });
  }

  let body = req.body;
  if (typeof body === "string") {
    try { body = JSON.parse(body); } catch { body = {}; }
  }
  const { image, media_type } = body || {};
  if (!image || typeof image !== "string") {
    setCors();
    return res.status(400).json({ error: "missing_params", need: ["image", "media_type"] });
  }
  // base64 expands ~4/3; check the decoded size.
  if (image.length * 0.75 > MAX_IMAGE_BYTES) {
    setCors();
    return res.status(413).json({ error: "image_too_large", max_bytes: MAX_IMAGE_BYTES });
  }

  try {
    const upstream = await fetch(ANTHROPIC_URL, {
      method: "POST",
      headers: {
        "content-type": "application/json",
        "x-api-key": apiKey,
        "anthropic-version": "2023-06-01",
      },
      body: JSON.stringify({
        model: MODEL,
        max_tokens: 1024,
        output_config: { format: { type: "json_schema", schema: PANEL_SCHEMA } },
        messages: [{
          role: "user",
          content: [
            { type: "image", source: { type: "base64", media_type: media_type || "image/jpeg", data: image } },
            { type: "text", text: PROMPT },
          ],
        }],
      }),
    });

    const text = await upstream.text();
    if (!upstream.ok) {
      setCors();
      return res.status(502).json({
        error: "upstream_failed",
        upstream_status: upstream.status,
        detail: text.slice(0, 300),
      });
    }

    const msg = JSON.parse(text);
    // A safety refusal returns 200 with stop_reason "refusal" and no content.
    if (msg.stop_reason === "refusal") {
      setCors();
      return res.status(200).json({
        readable: false, reason: "The model declined to read this image.",
        name: null, category: null, per100g: null,
      });
    }
    const block = (msg.content || []).find(b => b.type === "text");
    if (!block) {
      setCors();
      return res.status(200).json({
        readable: false, reason: "No readable response from the model.",
        name: null, category: null, per100g: null,
      });
    }

    let parsed;
    try { parsed = JSON.parse(block.text); }
    catch {
      setCors();
      return res.status(200).json({
        readable: false, reason: "Could not parse the model's response.",
        name: null, category: null, per100g: null,
      });
    }

    setCors();
    return res.status(200).json(parsed);
  } catch (err) {
    setCors();
    return res.status(502).json({ error: "internal_error", message: err.message });
  }
}
```

Add to `runlog-auth/README.md` under the endpoints section:

```markdown
### `POST /api/nutrition-label`

Read a product's nutrition panel from a photo. Holds `ANTHROPIC_API_KEY` so the
PWA doesn't have to.

Body:
```json
{ "image": "<base64, no data: prefix>", "media_type": "image/jpeg" }
```

Returns per-100g macros, or `readable: false` with a reason when the panel
can't be read. Never guesses values.

```json
{ "readable": true, "reason": null, "name": "Barilla Penne Rigate",
  "category": "pasta",
  "per100g": { "kcal": 352, "protein": 12.5, "carbs": 71.2, "fat": 1.5 } }
```

Uses `claude-haiku-4-5`, ~$0.0026 per label. Images over 5 MB are rejected
with 413; the client resizes to ≤1568px before upload.

Requires env var `ANTHROPIC_API_KEY` in addition to the WHOOP ones.
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
cd runlog-auth
ANTHROPIC_API_KEY=sk-ant-... npx vercel dev --listen 3000 &
sleep 8 && ./test-nutrition-label.sh ~/Desktop/label.jpg
```

Expected: all four checks PASS, and check 4 prints the parsed panel with a `derived kcal` close to the reported `kcal`. Also photograph something with no nutrition panel and confirm it returns `readable: false` rather than invented numbers.

- [ ] **Step 5: Commit and deploy**

```bash
cd runlog-auth
git add api/nutrition-label.js README.md test-nutrition-label.sh
git commit -m "feat: add /api/nutrition-label endpoint for reading product labels"
vercel env add ANTHROPIC_API_KEY   # paste key, select Production
vercel --prod
```

---

### Task 9: Photo capture and pre-fill wiring

**Files:**
- Modify: `runlog-app/index.html` — add photo handling to the UI section; wire the `#pf-file` change handler left open in Task 6; bump `APP_VERSION`

**Interfaces:**
- Consumes: `openPantryForm()`, `validateMacroPanel()`, `toast()`, `getConfig()`
- Produces:
  - `resizeImageFile(file, maxEdge) -> Promise<{base64, mediaType}>`
  - `readLabelPhoto(file) -> Promise<prefill|null>`
  - `NUTRITION_API_BASE` — reuses the existing auth-broker base URL

- [ ] **Step 1: Write the failing test**

```javascript
  _selfTestAsync.push((async () => {
    // resizeImageFile must cap the long edge and return base64 without the
    // data: prefix (the endpoint expects raw base64).
    const canvas = document.createElement('canvas');
    canvas.width = 3000; canvas.height = 2000;
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = '#fff'; ctx.fillRect(0, 0, 3000, 2000);
    const blob = await new Promise(r => canvas.toBlob(r, 'image/jpeg', 0.9));
    const file = new File([blob], 'test.jpg', { type: 'image/jpeg' });

    const out = await resizeImageFile(file, 1568);
    assert(typeof out.base64 === 'string' && out.base64.length > 0,
      'resizeImageFile returns base64');
    assert(!out.base64.startsWith('data:'),
      'resizeImageFile strips the data: prefix');
    assert(out.mediaType === 'image/jpeg', 'resizeImageFile reports media type');

    // Verify the encoded image really was scaled down.
    const img = new Image();
    await new Promise((res, rej) => {
      img.onload = res; img.onerror = rej;
      img.src = 'data:image/jpeg;base64,' + out.base64;
    });
    assert(Math.max(img.width, img.height) <= 1568,
      'resized long edge is capped at 1568px (got ' + img.width + 'x' + img.height + ')');
    assertClose(img.width / img.height, 1.5, 0.02, 'aspect ratio preserved');
  })());
```

- [ ] **Step 2: Run the test to verify it fails**

Reload the self-test URL. Expected: `resizeImageFile is not defined`.

- [ ] **Step 3: Write the minimal implementation**

```javascript
// --- label photo ---------------------------------------------------------
// The broker base URL is already stored for the WHOOP OAuth flow; reuse it
// so there's one place to change if the Vercel deployment moves.
const NUTRITION_API_BASE = 'https://runlog-auth.vercel.app';

// Downscale before upload. Two reasons: Haiku 4.5's image tier tops out
// around 1568px on the long edge, and a 12MP phone photo would otherwise
// blow past the endpoint's 5MB limit for no accuracy gain.
function resizeImageFile(file, maxEdge) {
  return new Promise((resolve, reject) => {
    const url = URL.createObjectURL(file);
    const img = new Image();
    img.onload = () => {
      URL.revokeObjectURL(url);
      const scale = Math.min(1, maxEdge / Math.max(img.width, img.height));
      const w = Math.round(img.width * scale);
      const h = Math.round(img.height * scale);
      const canvas = document.createElement('canvas');
      canvas.width = w; canvas.height = h;
      canvas.getContext('2d').drawImage(img, 0, 0, w, h);
      const dataUrl = canvas.toDataURL('image/jpeg', 0.85);
      resolve({ base64: dataUrl.split(',')[1], mediaType: 'image/jpeg' });
    };
    img.onerror = () => { URL.revokeObjectURL(url); reject(new Error('Could not read that image')); };
    img.src = url;
  });
}

// Send a label photo to the broker and turn the answer into a pantry-form
// prefill. Returns null on any failure — the caller then opens a blank form,
// so a vision failure degrades to typing rather than blocking.
async function readLabelPhoto(file) {
  let payload;
  try {
    payload = await resizeImageFile(file, 1568);
  } catch (e) {
    toast(e.message, true);
    return null;
  }

  let res, data;
  try {
    res = await fetch(NUTRITION_API_BASE + '/api/nutrition-label', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ image: payload.base64, media_type: payload.mediaType }),
    });
    data = await res.json();
  } catch (e) {
    toast('Could not reach the label reader. Enter it manually.', true);
    return null;
  }

  if (!res.ok) {
    const msg = data && data.error === 'server_not_configured'
      ? 'Label reader is not configured yet.'
      : data && data.error === 'image_too_large'
        ? 'That photo is too large.'
        : 'Label reader failed. Enter it manually.';
    toast(msg, true);
    return null;
  }
  if (!data.readable) {
    toast(data.reason || 'Could not read that label. Enter it manually.', true);
    return null;
  }

  // Never trust the panel blindly — flag inconsistencies for review rather
  // than silently accepting a misread number.
  const v = validateMacroPanel(data.per100g);
  return {
    name: data.name || '',
    category: data.category || 'other',
    per100g: data.per100g,
    _issues: v.ok ? [] : v.issues,
  };
}
```

Wire the file input — replace the `// The file input's change handler is wired in Task 9.` comment in `renderFoodPantry()`:

```javascript
  document.getElementById('pf-file').addEventListener('change', async (ev) => {
    const file = ev.target.files && ev.target.files[0];
    ev.target.value = '';           // allow re-picking the same file
    if (!file) return;
    toast('Reading label…');
    const prefill = await readLabelPhoto(file);
    // Either way we land on the form — prefilled if it worked, blank if not.
    openPantryForm(null, prefill);
  });
```

Bump `index.html:2027`:

```javascript
const APP_VERSION = '2026-08-06-46';
```

- [ ] **Step 4: Run the tests and verify end to end**

```bash
open 'http://localhost:8123/index.html?selftest=1'   # expect 32 groups passed
```

Then on a phone against the deployed PWA: Food → Pantry → "Add from label" → photograph a real pasta box → confirm the form pre-fills with the right numbers → Save → Log 120 g dry → confirm Today's totals. Then photograph something with no label and confirm you get a message plus a blank form, not invented numbers.

- [ ] **Step 5: Commit**

```bash
git add runlog-app/index.html
git commit -m "feat(nutrition): photograph a label to pre-fill the pantry form"
```

---

## Verification checklist

Run after Task 9, before merging:

- [ ] `?selftest=1` reports 32 groups passed
- [ ] All six tabs fit at 380px width
- [ ] Add a food by photo; numbers match the box
- [ ] Add a food manually; deliberately inconsistent numbers trigger the warning
- [ ] Log 120 g dry pasta and 288 g cooked pasta — both produce ~422 kcal
- [ ] Log 3 eggs via units mode — ~21 g protein
- [ ] Delete an entry; totals update
- [ ] Reload; everything persists
- [ ] `runlog-data` contains `pantry.json` and `nutrition.json` with correct counts
- [ ] Photograph a non-label; get a message and a blank form, never numbers
- [ ] Turn off wifi, log a meal, turn wifi on, reload — the dirty-flag retry pushes it
- [ ] Existing features still work: log a run, log a strength session, Analyze charts render

## Known follow-ups (not in this plan)

- `ALLOWED_ORIGIN` in `runlog-auth` still defaults to `*`; pin it to `https://emimy.github.io`
- No weekly/trend view for nutrition — Today only
- `index.html` reaches ~8,800 lines; extracting CSS is the clean first split
