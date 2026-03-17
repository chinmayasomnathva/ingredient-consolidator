---
name: ingredient-list-consolidator
description: Build or update an ingredient list consolidator web app that parses multiple ingredient lists, matches items against a Google Sheets master list, consolidates quantities, and exports a shopping list. Use this skill whenever the user wants to build, fix, or improve an ingredient consolidator, shopping list tool, recipe scaler, or any app that matches food/ingredient text against a master product list. Also trigger when the user reports matching bugs like wrong items being matched, missing items, or incorrect column outputs.
---

# Ingredient List Consolidator

A single-file HTML/React app that:
- Accepts up to N paste-in ingredient lists (start with 3, user can add more)
- Parses quantities, units, and ingredient names
- Matches against a Google Sheets master list
- Consolidates duplicates across lists
- Outputs a clean shopping list with Copy to Google Sheets and Submit (OAuth) actions

---

## Architecture

Single `index.html` with inline React (via Babel CDN). No build step. Key functions:

| Function | Purpose |
|---|---|
| `parseIngredientLine` | Extract quantity, unit, name from raw text |
| `normalizeIngredientName` | Clean name for matching (strip noise words, parens) |
| `fuzzyMatchWithCategory` | Score master list candidates for a given input |
| `calculateSimilarity` | Levenshtein + word-coverage hybrid similarity |
| `consolidateIngredients` | Orchestrate parse → match → merge pipeline |
| `finalizeConsolidation` | Build output rows after user resolves ambiguous matches |
| `copyToGoogleSheets` | TSV clipboard export |
| `submitToGoogleSheets` | OAuth write to Sheets tab |

---

## Output Columns

The output table and clipboard export include **only these columns**:
- Items
- Quantity
- Units
- Default Unit
- Preferred Store

**Never include** Preferred Brand, Category, or Alt Store in either the table or the `copyToGoogleSheets` TSV — these were intentionally removed. Any new column logic must stay consistent between the table render and `copyToGoogleSheets`. The `submitToGoogleSheets` function must also use these same 5 columns (`numCols = 5`).

---

## Quantity Formatting

```js
function formatQty(qty) {
    if (Number.isInteger(qty)) return String(qty);
    if (qty < 1) return qty.toFixed(2);       // e.g. 0.33 → "0.33"
    return parseFloat(qty.toPrecision(6)).toString(); // strip trailing zeros
}
```
Only show decimal places when quantity < 1. Whole numbers stay as integers.

---

## Quantity Parsing — Critical Regex Rule

All quantity patterns must use `\d*\.?\d+` (NOT `\d+\.?\d*`) to correctly handle decimal-only inputs like `.5`:

```javascript
// CORRECT - handles .5, 0.5, 1.5, 10, etc.
/(\d*\.?\d+)\s*(lbs?|pounds?)/gi

// WRONG - misses .5 (requires at least one leading digit)
/(\d+\.?\d*)\s*(lbs?|pounds?)/gi
```

This applies to ALL patterns in `parseQuantity`, the unit-stripping pattern in `parseIngredientLine`, and the range/fallback number patterns.

- **Last Number Rule**: Finds the LAST number+unit combination in each line
  - Example: "777 sambhar powder 500 gms – 1 packet" → uses "1 packet" as quantity
- **Range Handling**: Processes ranges and uses upper limit
  - Example: "5-6 tomatoes" → uses 6 tomatoes
- **Pantry Items**: Marks items as "pantry" when no quantity or unit specified

---

## Ingredient Name Normalization (`normalizeIngredientName`)

### Parenthetical handling — KEEP content, remove brackets
```js
normalized = normalized.replace(/\(([^)]+)\)/g, (match, inner) => {
    const innerLower = inner.toLowerCase().trim();
    // Strip only clearly meaningless annotations
    if (/^(brand|store|costco|trader joe|restaurant depot|organic|optional|adjust to taste)$/i.test(innerLower)) {
        return '';
    }
    return ' ' + innerLower; // Keep the content inline
});
```
**Why:** "Black Pepper (Red)" must become "black pepper red" — not "black pepper". Stripping all parentheticals loses meaningful descriptors that differentiate items.

### Colour words
Only strip colour words (red, green, black, etc.) when the item is NOT a pepper, chili, onion, tomato, bell pepper, peas, pumpkin, salt, cardamom, mustard, grape, or lentil/dal variant. Colours are meaningful for those categories.

### Word removal list
Strip: `twin pack, costco, restaurant depot, trader joe, patel, patel brothers, whole foods, kroger, walmart, organic, extra virgin, virgin, unsalted, salted, fresh, frozen, canned, plain, simple, regular, classic, shredded, diced, chopped, sliced, whole, peeled, flat leaf, baby, puree, pantry, big, large, small, medium, packet, pack, bundle, mixed, english, long, mini, seedless, boneless, skinless`

Also strip prepositions/stop words including `from`: `from, or, and, of, the, a, an, with, in, on, for, to`

**Why `plain`, `from`, and store/brand names were added:**
- `"plain roti from patel"` must normalize to `"roti"` so it can match "Fresh Rotis" via synonym.
- Without stripping `patel` and `from`, unmatched tokens tank the similarity score.
- `plain` is a descriptor (like `fresh`) that adds no ingredient identity.

**Protect exceptions:**
- Keep `mixed` for bell peppers
- Keep `english` AND `long` for cucumbers (master key is often "cucumbers (english - long)")
- Push `peas` to remove list for bean/chickpea inputs

---

## Matching Pipeline (`consolidateIngredients`)

> ⚠️ **Stale closure fix**: After calling `setLists(numberedLists)`, always iterate `numberedLists` (not `lists`) in the same function body. React state updates are async, so `lists` still holds the old value.

Three-tier lookup — run in order, stop at first hit:

### Tier 1 — Direct dict lookup
```js
let masterMatch = masterData.lookup[name]; // exact lowercase key
```

### Tier 2 — Base-key phrase scan (catches adjective-qualified items)
```js
for (const [masterKey, masterVal] of Object.entries(masterData.lookup)) {
    // CRITICAL: strip parens OFF entirely — never expand them into the matching string.
    // "Kanda Lasun Masala (Onion Garlic)" base key = "kanda lasun masala"
    // Input "garlic" must NOT match this, even though the expanded text ends with "garlic".
    const baseKey = masterKey.replace(/\s*\([^)]*\)/g, '').replace(/\s+/g, ' ').trim().toLowerCase();
    if (baseKey === name || masterKey.toLowerCase() === name) {
        masterMatch = masterVal; break;
    }
    // Only additional case: hyphens/dashes in master key treated as spaces
    const baseKeyDashNorm = baseKey.replace(/-/g, ' ').replace(/\s+/g, ' ').trim();
    if (baseKeyDashNorm === name) {
        masterMatch = masterVal; break;
    }
}
```
**Why:** "baby corn" matches "Baby Corn (frozen)" via its base key "baby corn". Input "garlic" does NOT match "Kanda Lasun Masala (Onion Garlic)" because the base key is "kanda lasun masala".

⚠️ **Never use paren-expanded strings in Tier 2.** Expanding parens was the root cause of garlic silently matching Kanda Lasun Masala.

⚠️ **Never use startsWith/endsWith/includes in Tier 2.** These cause silent false matches:
- `"tomato"` startsWith-matches `"Tomato Ketchup"` → wrong auto-match
- `"garlic"` startsWith-matches `"Garlic Paste"`, `"Garlic Pods"` → wrong auto-match
- `"garlic"` endsWith-matches `"Minced Garlic"` → wrong auto-match

**Tier 2 must only match when `baseKey === name` exactly** (plus the dash-normalised variant). Everything else must fall through to fuzzy → resolver dialog.

### Edit-Distance Fallback Discount
The edit-distance fallback score is multiplied by 0.95 to prevent borderline cases (raw score exactly 0.50) from squeezing into the resolver:
```js
return ((longer.length - editDistance) / longer.length) * 0.95;
```
**Why:** `"whole milk"` vs `"coconut milk"` has edit distance 6 over length 12 → raw score 0.500, exactly at the `>= 0.50` resolver threshold. With the 0.95 discount → 0.475, safely below threshold → not shown as a resolver candidate.

---

### Tier 3 — Fuzzy match with category boost
Auto-accept at >95% similarity. Show selection modal at 50–95%.

---

## Fuzzy Match Guards (in `fuzzyMatchWithCategory`)

All guards use `continue` to skip a master key entirely before scoring.

### Guard 1 — Compound masala/blend blocklist

> ⚠️ **`powder` is intentionally NOT in this pattern.** Single-ingredient powders like "Turmeric Powder" and "Coriander Powder" must remain reachable when a user types "turmeric" or "coriander". The form-mismatch guard in `calculateSimilarity` already handles cases where the user explicitly types a conflicting form (e.g. `"turmeric seeds"` vs `"turmeric powder"`). Guard 1 must only block true compound blend names.

```js
const isCompoundMasala = /\b(masala|blend|spice mix|seasoning)\b/i.test(key);
const inputIsSimple = searchTerm.split(/\s+/).filter(w => w.length > 2).length <= 2;
if (isCompoundMasala && inputIsSimple) {
    if (!/(masala|spice|seasoning|blend)/i.test(searchTerm)) continue;
}
```

### Guard 2 — Multi-ingredient parenthetical block
```js
// Prevents "garlic pods" → "Kanda Lasun Masala (Onion Garlic)"
const multiIngredientParen = /\(([^)]+)\)/g;
while ((parenMatch = multiIngredientParen.exec(key)) !== null) {
    const parenWords = parenMatch[1].toLowerCase().split(/[\s,\/]+/).filter(w => w.length > 2);
    if (parenWords.length >= 2 && !parenWords.every(pw => searchTerm.includes(pw))) {
        skipDueToMultiIngredient = true; break;
    }
}
if (skipDueToMultiIngredient) continue;
```
**Why:** "Kanda Lasun Masala (Onion Garlic)" has 2 paren-words: "onion" and "garlic". Input "garlic pods" only contains "garlic", not both — so it's skipped.

### Guard 3 — Brand number token guard
```js
// Prevents "777 Sambhar Powder" → "Amchoor Powder 777"
const brandNumberMatch = searchTerm.match(/\b(\d{2,})\b/);
if (brandNumberMatch) {
    const brandNum = brandNumberMatch[1];
    if (!key.includes(brandNum) && !keyForMatch.includes(brandNum)) continue;
    const inputWithoutNum = searchTerm.replace(/\b\d+\b/g, '').trim();
    const keyWithoutNum = keyForMatch.replace(/\b\d+\b/g, '').trim();
    if (calculateSimilarity(inputWithoutNum, keyWithoutNum) < 0.4) continue;
}
```

---

## Synonym Table (in `fuzzyMatchWithCategory`)

Synonyms redirect the `searchTerm` before all scoring. Longest-key-wins to prevent partial clobbers.

```js
const synonyms = {
    // Cumin variants
    'jira': 'cumin',
    'jeera': 'cumin',
    'zeera': 'cumin',
    // Coriander — bare 'coriander' intentionally NOT mapped to cilantro;
    // "coriander powder" and "coriander seeds" are distinct spices, not leaves.
    'coriander leaf': 'cilantro',
    'coriander leaves': 'cilantro',
    'fresh coriander': 'cilantro',
    'curry leaf': 'curry leaves',
    'tomato': 'tomatoes',
    // Legumes
    'chickpea': 'garbanzo beans',
    'chickpeas': 'garbanzo beans',
    'garbanzo': 'garbanzo beans',
    'garbanzos': 'garbanzo beans',
    'lobia': 'black eyed peas',       // Hindi/Urdu regional name
    'lobiya': 'black eyed peas',      // alternate spelling
    'black eyed bean': 'black eyed peas',
    'black eyed beans': 'black eyed peas',
    // Cucumbers
    'english cucumber': 'cucumbers english long',
    'cucumber': 'cucumbers',
    // Pasta
    'penne pasta': 'pasta penne rigatoni elbow',
    'penne': 'pasta penne',
    'rigatoni': 'pasta penne',
    'elbow pasta': 'pasta penne rigatoni elbow',
    // Peppers
    'mixed bell pepper': 'bell peppers mixed',
    'bell pepper': 'bell peppers',
    'capsicum': 'bell peppers',
    // Butter
    'butter sticks': 'unsalted butter sticks',
    'butter': 'unsalted butter sticks',
    'peanut butter': 'peanut butter',
    'almond butter': 'almond butter',
    'nut butter': 'nut butter',
    // Alliums
    'spring onion': 'green onions',
    'spring onions': 'green onions',
    'scallion': 'green onions',
    'scallions': 'green onions',
    // Rotis / flatbreads
    'roti': 'rotis',
    'rotis': 'rotis',
    'chapati': 'rotis',
    'chapatis': 'rotis',
    // Coconut brand variants — "Daily Delight" is a brand, not a descriptor
    'coconut daily delight': 'coconut shredded',
    'daily delight coconut': 'coconut shredded',
};
```

**Rules for adding synonyms:**
- Use when a regional/Hindi/Urdu name has zero lexical overlap with the master key (e.g. `lobia` → `black eyed peas`).
- Use when a brand name is embedded in the product description (e.g. `coconut daily delight` → `coconut shredded`).
- Do NOT add a synonym just because similarity is low — fix normalization or scoring instead.
- Always add the longest variant before shorter ones if there is overlap risk (e.g. `'english cucumber'` before `'cucumber'`).

---

## `copyToGoogleSheets` — Must Stay in Sync with Table

```js
function copyToGoogleSheets() {
    const headers = ['Items', 'Quantity', 'Units', 'Default Unit', 'Preferred Store'];
    const rows = consolidated.map(item => {
        let qty = '', unit = '';
        if (item.count === 'pantry') {
            qty = 'pantry';
        } else {
            qty = item.count.split(',').map(p => p.trim().replace(/^([\d.]+)\s+.*$/, '$1')).join(', ');
            unit = item.count.split(',').map(p => p.trim().replace(/^[\d.]+\s+(.+)$/, '$1')).join(', ');
        }
        return [item.name, qty, unit, item.defaultUnit, item.preferredStore];
    });
    ...
}
```
**Rule:** When columns are added or removed from the output table, always update `copyToGoogleSheets` headers AND row mapping AND `submitToGoogleSheets` headers AND `numCols` to match.

---

## Row Colour Coding

| Colour | Meaning |
|---|---|
| Green (bg-green-100) | Consolidated (appeared in 2+ lists) |
| Amber light (bg-amber-50) | Pantry item |
| Amber dark (bg-amber-200) | Pantry item that appeared in 2+ lists |
| Red (bg-red-100) | Not found in master list |

---

## Sorting

- Non-pantry items sorted by: **Preferred Store** (alphabetically) → **Item name** (alphabetically)
- Pantry items always at the bottom, also sorted alphabetically by name

---

## Google Sheets Integration

### Master List (Read)
- **Sheet ID**: `1yJyhd23X77zH_ITtQu1AtNTI-Ak9KnorymsPJgEJKMc`
- **Tab**: `Master_Items` (range `Master_Items!A:F`)
- **URL construction**:
```js
const range = 'Master_Items!A:F'; // Always this literal — NOT ${SHEET_NAME}!A:F
const url = `https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/${encodeURIComponent(range)}?key=${apiKey}`;
```
> ⚠️ `SHEET_NAME` (`'Master List_Annapurna'`) is for display only. Never interpolate it into the API URL — the actual tab name is `Master_Items`. Always `encodeURIComponent` the range.
- **Columns**: Items, Default Unit, Preferred Brand, Category, Preferred Store, Alt Store
- **Auth**: Public API key (`const API_KEY`)
- Loads automatically on mount if API key is set

```javascript
masterData = {
  lookup: {
    "red onion": {
      item: "Red Onion",
      defaultUnit: "lbs",
      preferredBrand: "",
      category: "Produce",
      preferredStore: "Lotte",
      altStore: "RD"
    }
  },
  categoryIndex: {
    "produce": [ { key: "red onion", data: {...} }, ... ]
  }
}
```

### Shopping Sheet (Write via OAuth)
- **Sheet ID**: `1UbBBupMUuF8m4Ax8B8Vfid2ojqa0nt4bbOxG4JpmyI4`
- **Tab name format**: `MM/DD/YYYY` (e.g., `02/28/2025`)
- **Columns written**: Items, Quantity, Units, Default Unit, Preferred Store (5 columns, `numCols = 5`)
- **Auth**: OAuth2 implicit flow via popup + postMessage

### Submit Flow (4 steps)
```
Step 1: GET spreadsheet metadata → find unique tab name (never overwrite)
Step 2: POST spreadsheets:batchUpdate → addSheet (create new tab)
Step 3: PUT spreadsheets/values/{range} → write header + data rows (5 columns)
Step 4: GET metadata → find sheetId → POST batchUpdate → formatting + freeze + auto-resize
```

---

## Master Load Status UI

Always show three distinct states — never let errors silently disappear into a generic warning:

```jsx
{masterData && (
    <div className="bg-green-50 ...">✓ Loaded ({Object.keys(masterData.lookup).length} items)</div>
)}
{loadingMaster && (
    <div className="bg-blue-50 ...">⏳ Loading master list…</div>
)}
{!masterData && !loadingMaster && masterError && (
    <div className="bg-red-50 ...">✗ Failed to load master list: {masterError}</div>
)}
{!masterData && !loadingMaster && !masterError && (
    <div className="bg-yellow-50 ...">⚠ Master data not loaded. Please configure API key in the code.</div>
)}
```

---

## Google Sheet Formatting on Submit

After data is written to the new tab, a second API call applies formatting.

**Header row (row 1):**
- Bold text, white foreground
- Dark gray background: `{ red: 0.26, green: 0.26, blue: 0.26 }`
- `verticalAlignment: MIDDLE`, frozen (`frozenRowCount: 1`)

**Data row colors — match on-screen table:**
| Row Type | Tailwind Class | RGB Values |
|----------|---------------|------------|
| Not found | bg-red-100 | r:1.0, g:0.80, b:0.80 |
| Pantry (single) | bg-amber-50 | r:1.0, g:0.98, b:0.94 |
| Pantry (combined) | bg-amber-200 | r:1.0, g:0.91, b:0.71 |
| Consolidated | bg-green-100 | r:0.88, g:0.96, b:0.88 |
| Normal | white | r:1.0, g:1.0, b:1.0 |

**Row color logic:**
```javascript
function getRowColor(item) {
    if (item.matchStatus === 'not-found') return colorMap['not-found'];
    if (item.isPantry && item.wasCombined) return colorMap['pantry-combined'];
    if (item.isPantry) return colorMap['pantry-single'];
    if (item.wasCombined) return colorMap['consolidated'];
    return colorMap['normal'];
}
```

**Column auto-resize (5 columns):**
```javascript
{ autoResizeDimensions: { dimensions: { sheetId, dimension: 'COLUMNS', startIndex: 0, endIndex: 5 } } }
```

> ⚠️ Always look up `sheetId` dynamically after tab creation — never hardcode it.

---

## Google OAuth Authentication

### Architecture
Manual OAuth2 implicit flow with a popup window and `postMessage`. Two files required:
1. **`index.html`** — main app
2. **`oauth2callback.html`** — OAuth redirect target (same directory)

Both must be served from the same origin (e.g. `http://localhost:8000` or GitHub Pages).

### OAuth Flow
```
User clicks "Sign in with Google"
    → Opens popup: accounts.google.com/o/oauth2/v2/auth
    → User approves
    → Google redirects to oauth2callback.html#access_token=...
    → oauth2callback.html parses token from URL hash
    → postMessage({ type: 'oauth2_token', access_token }) to opener
    → Popup closes
    → Main app receives token, stores in state
```

### oauth2callback.html
```javascript
var hash = window.location.hash.substring(1);
var params = {};
hash.split('&').forEach(function(part) {
    var kv = part.split('=');
    params[decodeURIComponent(kv[0])] = decodeURIComponent(kv[1] || '');
});
if (params.access_token) {
    window.opener.postMessage({
        type: 'oauth2_token',
        access_token: params.access_token,
        email: 'Signed in'
    }, window.location.origin);
    window.close();
}
```

### Constants in Code
```javascript
const OAUTH_CLIENT_ID = '905820431932-rpieibglpo732pk1ia56bo9mub4b3qtk.apps.googleusercontent.com';
const SUBMIT_SHEET_ID = '1UbBBupMUuF8m4Ax8B8Vfid2ojqa0nt4bbOxG4JpmyI4';
```

### OAuth Scope
```
https://www.googleapis.com/auth/spreadsheets
```

### Common OAuth Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `redirect_uri=storagerelay://` | Using deprecated GSI/gapi library | Use manual OAuth2 popup with oauth2callback.html |
| `Error 403: access_denied` | App in Testing mode, account not a test user | Add email to Test Users in OAuth consent screen |
| `Error 400: invalid_request` | Opening file:// directly | Serve via `python3 -m http.server 8000` |
| `Error 500` on consent screen | OAuth consent screen not fully configured | Fill in App name, support email, developer contact, and scopes |

### Deployment
- **Local**: Run `python3 -m http.server 8000`, open `http://localhost:8000/index.html`
- **GitHub Pages**: Push both files to repo root, enable Pages from main branch

---

## Matching Algorithm: Similarity Calculation

### Descriptor Recognition
Items differing only by descriptors get **80% similarity** (triggers resolution dialog).

Descriptor list:
```javascript
const descriptors = ['frozen', 'fresh', 'canned', 'dried', 'raw', 'cooked',
    'baby', 'red', 'white', 'green', 'yellow', 'black',
    'organic', 'large', 'small', 'whole', 'chopped', 'diced',
    'sliced', 'ground', 'big', 'packet', 'pack', 'bundle',
    'loose', 'extra', 'mixed', 'english', 'long', 'mini',
    'seedless', 'boneless', 'skinless', 'shelled', 'pitted'];
```

Examples:
- `"spinach"` → `"baby spinach"` = 0.80 (fresh/frozen/baby are descriptors)
- `"turmeric"` → `"fresh turmeric"` = 0.80, `"turmeric powder"` = 0.55 — **both shown in resolver**
- `"potato"` → `"potato chips"` = 0.55 (chips is not a descriptor)

### Noun-Based Matching Priority
```javascript
Input: "black chana"
Main noun: "chana" (last word)
Adjectives: ["black"]

Sort order: Noun match → Similarity → Category match
```

Noun match boost: **+20%**. Category match boost: **+30%**.

### Thresholds
| Similarity | Action |
|---|---|
| > 95% | Auto-match |
| 50–95% | Show resolution dialog (up to 8 matches) |
| < 50% | Mark as not found |

---

## User Selection Modal

- Shows up to 8 matches when similarity is between 50–95%
- **Blue highlight** for same-category matches with "Category Match" badge
- Displays: Item name, similarity %, category, preferred store
- User can select best match or skip to keep original name

---

## New Guards Added to `calculateSimilarity`

### Form/Preparation Mismatch Guard
Runs BEFORE similarity scoring. If **both** strings contain a "form" word and those forms differ, return 0.40 — below the resolver threshold.

```js
const formWords = new Set(['seeds','seed','oil','powder','paste','leaf','leaves',
    'sauce','juice','extract','flakes','chips','ketchup','starch','flour','vinegar','butter','cream']);
const str1Forms = str1Words.filter(w => formWords.has(w.toLowerCase()));
const str2Forms = str2Words.filter(w => formWords.has(w.toLowerCase()));
if (str1Forms.length > 0 && str2Forms.length > 0) {
    const formsMatch = str1Forms.some(f => str2Forms.includes(f));
    if (!formsMatch) return 0.40; // Different forms = different products
}
```
Examples blocked: `mustard seeds` vs `mustard oil`, `lemon juice` vs `lemon extract`, `coconut milk` vs `coconut cream`.

> **Turmeric note:** `"turmeric"` (no form word) vs `"turmeric powder"` (one form word) → guard does NOT fire (requires both sides to have a form word) → resolver correctly shows both "Fresh Turmeric" (0.80) and "Turmeric Powder" (0.55).

### Generic Noun Boost Skip
In `fuzzyMatchWithCategory`, noun boost (×1.2) is skipped when the input noun is a generic category word shared by many items:

```js
const genericNouns = new Set(['seed','seeds','powder','sauce','oil','paste','water',
    'juice','flour','extract','flakes','leaves','leaf','root','peel',
    'milk','cream','butter','vinegar','syrup','stock','broth']);
if (!isGenericNoun && keyForMatch.includes(inputNoun)) {
    similarity = Math.min(1.0, similarity * 1.2);
}
```
**Why:** Without this, `melon seeds` → `cumin seeds` because both share noun `seed`, and cumin gets boosted above melon seeds in the resolver ranking.

---

## Additional Normalization Rules

### Protect `whole` for milk/dairy items
```js
const isMilk = normalized.includes('milk') || normalized.includes('cream') ||
               normalized.includes('dairy') || normalized.includes('yogurt') || normalized.includes('curd');
if (isMilk) {
    const idx = removeWords.indexOf('whole');
    if (idx > -1) removeWords.splice(idx, 1);
}
```
**Why:** `whole milk` → `whole` stripped → `milk` → matches `coconut milk` falsely.

---

## Regex Bug: `bunches?` Does NOT Match `bunch`

`bunches?` = matches `bunche` or `bunches` (the `?` makes `s` optional, but `e` is required).
`bunch` (no `e`) is NOT matched.

Fix: change to `bunch(?:es)?` in both `parseQuantity` unitList and `parseIngredientLine` unitPattern.

Same bug exists for `boxes?` → fixed to `box(?:es)?`.

---

## Scoring Summary for Resolver Ranking

| Input vs Master | Score | Goes to resolver? |
|---|---|---|
| `spinach` vs `baby spinach` (descriptor-only extra) | 0.80 | Yes |
| `turmeric` vs `fresh turmeric` (fresh is descriptor) | 0.80 | Yes |
| `turmeric` vs `turmeric powder` (powder not a descriptor) | 0.55 | Yes, ranks below fresh turmeric |
| `potato` vs `potato chips` (substantive extra) | 0.55 | Yes, but ranks low |
| `mustard seeds` vs `mustard oil` (different forms) | 0.40 | No — blocked |
| `black peppercorns` vs `white peppercorns` (different color) | 0.50 | Yes, at boundary |
| `melon seeds` vs `cumin seeds` (different noun, generic seed) | edit dist ~0.45 | No |
| `whole milk` vs `coconut milk` (different noun, whole protected) | edit dist ~0.45 | No |

---

## Common Bugs & Fixes

| Bug | Root Cause | Fix |
|---|---|---|
| Items re-processed from old list state | Stale closure: `lists.forEach` after `setLists(numberedLists)` | Iterate `numberedLists` (local variable), not `lists` (React state) |
| "baby corn" stripped to "corn", wrong match | `normalizeIngredientName` strips "baby" before Tier 2 lookup | Tier 2 full-phrase scan catches "baby corn" before fuzzy |
| "cucumber english long" loses "long" | `'long'` in removeWords, cucumber protection only rescued `'english'` | Splice `'long'` out of removeWords for cucumber items too |
| "garlic pods" → "Kanda Lasun Masala (Onion Garlic)" | Masala blocklist insufficient | Guard 2: block if paren has 2+ ingredients and input doesn't match all |
| "777 Sambhar Powder" → "Amchoor Powder 777" | Shared "777" and "powder" tokens inflated similarity | Guard 3: brand number requires both brand token AND base ingredient similarity ≥0.4 |
| "Black Pepper (Red)" disappears | `(Red)` stripped before matching | Keep paren content inline; only strip known-meaningless annotations |
| Submit writes 8 columns, table shows 5 | `submitToGoogleSheets` using different headers | Align to 5-column spec; set `numCols = 5` |
| Copy to Sheets includes extra columns | `copyToGoogleSheets` not updated when columns changed | Always update both table and copy function together |
| ".5 lb" parsed as "5 count" | Regex `\d+\.?\d*` requires leading digit | Change to `\d*\.?\d+` in all quantity patterns |
| "Green chilly" — "green" stripped incorrectly | `isChili` checked `chili`/`chilli` but not `chilly` | Add `chilly` to the `isChili` condition |
| "Black pepper corns" stays as-is | "corns" left on name | Add normalization: `pepper corns` → `peppercorns` after word removal |
| "Coriander powder" matched to cilantro | Synonym `'coriander': 'cilantro'` fires on any "coriander" string | Remove bare `'coriander'` synonym; only map leaf/fresh variants |
| "Thandai (45)" treated as pantry ingredient | `(45)` kept as inline content → "thandai 45" | Section-header guard: skip lines matching `^[A-Za-z][A-Za-z\s]*\s*\(\d+\)\s*$` with no dash |
| "Capsicum" not mapped to bell peppers | Missing from synonym table | Add `'capsicum': 'bell peppers'` |
| "Spring Onion" not mapped | Missing from synonym table | Add `'spring onion': 'green onions'`, `'scallion': 'green onions'` |
| "garlic" → silently auto-matched to Kanda Lasun Masala | Tier 2 expanded parens, base key endsWith "garlic" | Tier 2 strips parens OFF — only compares base name |
| "black peppercorns" auto-matched to "white peppercorns" | Color boost pushed score >0.95 | Color/qualifier guard: mismatched colors cap at 0.50 |
| "onion" auto-matched to "frozen small onions" | Boost chain inflated to 1.0 | Boost cap: rawSimilarity < 0.90 → clamp to 0.94 |
| Master load URL broken / dead variable | `range` variable computed from `SHEET_NAME` but not used | Always use `'Master_Items!A:F'` literal, wrap in `encodeURIComponent` |
| API error silently swallowed | `masterError` state set but never displayed | Show 3 distinct states: loading (blue), error (red), unconfigured (yellow) |
| "turmeric" can't reach "Turmeric Powder" in resolver | Guard 1 pattern included `powder` → blocked all `*Powder` master keys | Remove `powder` from Guard 1; it only needs to block masala/blend/seasoning |
| "lobia beans dry" → no match | `lobia` has zero lexical overlap with "Black Eyed Peas" | Add `'lobia': 'black eyed peas'` (and `'lobiya'`) to synonyms |
| "plain roti from patel" → no match | `plain`, `from`, `patel` left as unmatched tokens | Add `plain`, `from`, store/brand names to removeWords |
| "coconut daily delight" → no match | "Daily Delight" is a brand, zero lexical overlap with "shredded" | Add `'coconut daily delight': 'coconut shredded'` to synonyms |
| "roti" / "chapati" → no match for "Fresh Rotis" | No synonym for roti variants | Add `'roti': 'rotis'`, `'chapati': 'rotis'` to synonyms |

---

## Checklist for New Bug Fixes

- [ ] "spinach" → Shows: baby spinach, frozen spinach
- [ ] "spinach big packet" → normalizes to "spinach" → same result
- [ ] "small pack curry leaf" → normalizes to "curry leaf" → synonym → "curry leaves" → shows in dialog
- [ ] "onion" → Shows: red, white, yellow onion (NOT kanda lasun masala)
- [ ] "garlic pods" → Does NOT show: Kanda Lasun Masala
- [ ] "onion garlic masala" → SHOULD show: kanda lasun masala
- [ ] "potato" → Should NOT show: potato chips
- [ ] "black chana" → Shows: kala chana (first), chana dal (second)
- [ ] "lemon" → Should NOT auto-match: lemon juice
- [ ] "ginger .5 lb" → Parsed as 0.5 lbs (NOT 5 count)
- [ ] Same unit quantities sum: 5 lbs + 3 lbs = 8 lbs
- [ ] Different unit quantities display: 5 lbs, 3 count
- [ ] Submit writes 5 columns matching on-screen table
- [ ] Copy to Sheets TSV has 5 columns matching on-screen table
- [ ] English long cucumber normalizes correctly
- [ ] "Green chilly - 100 gms" → name = "green chilly", qty = 100 gms (green preserved)
- [ ] "Black pepper corns - 100 gm" → normalizes to "black peppercorns"
- [ ] "Black pepper corns" from 3 lists → consolidates quantity correctly
- [ ] "Coriander powder" → does NOT match cilantro items
- [ ] "Thandai (45)" → skipped as section header, not treated as ingredient
- [ ] "Capsicum - 5 lb" → maps to bell peppers via synonym
- [ ] "Spring Onion - 5 packets" → maps to green onions via synonym
- [ ] "Fennel seeds" from lists 1 and 3 → quantities summed (360 gms + 500 gms = 860 gms)
- [ ] "Garlic" across lists with different units (lb vs gms) → shown as "1 lbs, 250 gms"
- [ ] "turmeric" → resolver shows BOTH "Fresh Turmeric" (0.80) AND "Turmeric Powder" (0.55)
- [ ] "turmeric powder" → resolver shows "Turmeric Powder" at top (form guard blocks "Fresh Turmeric" since both have a form word? No — "turmeric powder" has powder, "fresh turmeric" has no form word → form guard does NOT fire → "Fresh Turmeric" still shows at 0.80, "Turmeric Powder" at 0.90+ → let user decide)
- [ ] "lobia beans dry" → resolver shows "Black Eyed Peas"
- [ ] "lobiya" → resolver shows "Black Eyed Peas"
- [ ] "plain roti from patel" → normalizes to "roti" → resolver shows "Fresh Rotis"
- [ ] "chapati" → resolver shows "Fresh Rotis" via synonym
- [ ] "coconut daily delight" → resolver shows "Coconut Shredded (Frozen)"
- [ ] Master load shows blue spinner while fetching
- [ ] Master load shows red error with actual message if API key fails
- [ ] Master load URL uses `encodeURIComponent('Master_Items!A:F')` — not SHEET_NAME

---

## Configuration Constants

```javascript
// Master list (read-only, API key)
const SHEET_ID = '1yJyhd23X77zH_ITtQu1AtNTI-Ak9KnorymsPJgEJKMc';
const SHEET_NAME = 'Master List_Annapurna'; // display label only — NOT used in API URL
const API_KEY = 'AIzaSyAxFgijOuH0PeE-wmK3DBG2XqmHNnnabyE';

// Shopping sheet (write, OAuth)
const OAUTH_CLIENT_ID = '905820431932-rpieibglpo732pk1ia56bo9mub4b3qtk.apps.googleusercontent.com';
const SUBMIT_SHEET_ID = '1UbBBupMUuF8m4Ax8B8Vfid2ojqa0nt4bbOxG4JpmyI4';
```
