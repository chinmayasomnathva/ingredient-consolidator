# Ingredient List Consolidator - Skill Documentation

## Overview
This skill helps users create and debug an ingredient list consolidator tool that combines multiple shopping lists into a single, organized list with automatic store assignments, quantity consolidation, and master list integration for standardized ingredient names with intelligent noun-based and descriptor-aware matching.

## When to Use This Skill
- User wants to consolidate multiple ingredient/shopping lists
- User needs to combine quantities across different recipe lists
- User wants to standardize input ingredient names with Items column from master list Google Sheet
- User wants to assign columns (Default Unit, Preferred Brand, Category, Preferred Store, Alt Store) based on matching items from the master list
- User wants intelligent matching that recognizes descriptors (colors, adjectives) vs substantive differences
- User wants noun-based matching priority (e.g., "black chana" shows "Kala Chana" and "Chana Dal" prioritizing items with "chana")
- User wants to consolidate items AFTER matching process is complete, with quantities summed if same unit, otherwise displayed as comma-separated string
- User needs visual indicators for unfound items in the output

## Key Features

### 1. Master List Integration

#### Google Sheets Connection
- Connects to master ingredient list via Google Sheets API
- Sheet ID: `1yJyhd23X77zH_ITtQu1AtNTI-Ak9KnorymsPJgEJKMc`
- Sheet Name: `Master List_Annapurna` (configurable)
- Expected columns: Items, Default Unit, Preferred Brand, Category, Preferred Store, Alt Store
- API key configured in code (const API_KEY)
- Auto-loads on application start if API key is set
- Simple status display: "✓ Loaded Master Data (X items)"

#### Master Data Structure
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
    "produce": [
      { key: "red onion", data: {...} },
      { key: "yellow onion", data: {...} }
    ],
    "spice": [
      { key: "kanda lasun onion garlic masala", data: {...} }
    ]
  }
}
```

### 2. Intelligent Matching Algorithm

#### Matching Priority & Thresholds
1. **Exact Match**: Direct match with master list item name → Auto-match
2. **Very High Similarity (>95%)**: Auto-match without user confirmation
3. **High Similarity (50-95%)**: Show resolution dialog for user selection
4. **Low Similarity (<50%)**: Mark as not found

#### Similarity Calculation Rules

**Descriptor Recognition:**
Items differing only by descriptors get **80% similarity** (triggers resolution).

Full descriptor list (words treated as non-substantive modifiers):
- Colors: red, white, green, yellow, black
- Processing: frozen, fresh, canned, dried, raw, cooked
- Size/Type: baby, organic, large, small, whole, big
- Packaging: packet, pack, bundle, loose, extra
- Prep: chopped, diced, sliced, ground

Examples:
- "spinach" → "baby spinach" = 80%
- "spinach" → "frozen spinach" = 80%
- "spinach big packet" → normalizes to "spinach" → matches "Baby Spinach", "Frozen Spinach" = 80%
- "curry leaves" → "curry leaves large packet" = 80% (large + packet are descriptors)
- "onion" → "red onion" = 80%
- "onion" → "white onion" = 80%

**Substantive Differences:**
Items with non-descriptor differences get lower similarity:
- "potato" → "potato chips" = 60% (chips is not a descriptor)
- "lemon" → "lemon juice" = 60% (juice is not a descriptor)

**No Common Words:**
Items with no word overlap get very low similarity:
- "potatoes" → "potato chips" = 30% (no true overlap after stemming check)

#### Compound Masala / Spice Blend Blocklist
Compound masala and spice blend items (those containing "masala", "blend", "spice mix", or "seasoning" in the master list key) are **never suggested** when the input is a simple 1–2 word ingredient that does not itself reference masala/spice.

- "onion" → will NOT show "Kanda Lasun Masala (Onion Garlic)"
- "garlic" → will NOT show "Kanda Lasun Masala (Onion Garlic)"
- "onion garlic masala" → WILL show "Kanda Lasun Masala (Onion Garlic)" (input contains masala)

**Implementation:**
```javascript
const compoundMasalaPattern = /\b(masala|blend|spice mix|seasoning)\b/i;
const isCompoundMasala = compoundMasalaPattern.test(key);
const inputIsSimple = searchTerm.split(/\s+/).length <= 2;
if (isCompoundMasala && inputIsSimple) {
    if (!/(masala|spice|seasoning|blend)/i.test(searchTerm)) {
        continue; // Skip this match entirely
    }
}
```

#### Noun-Based Matching Priority
The algorithm extracts the **main noun** (last word) and prioritizes matches containing it:

```javascript
Input: "black chana"
Main noun: "chana"
Adjectives: ["black"]

Matching process:
1. Find all items containing "chana"
2. Boost similarity by 20% for noun matches
3. Boost similarity by 30% for same category
4. Sort by: Noun match → Similarity → Category match

Result order:
1. "Kala Chana" (noun: ✓, similarity: high, category: ✓)
2. "Chana Dal" (noun: ✓, similarity: good, category: ✓)
3. Other items ranked lower
```

#### Category-Aware Filtering
- Identifies likely category from highest initial match
- Boosts same-category items by 30%
- Shows category matches + any high similarity items (>70%)
- Filters out very different categories only when there's strong category signal

Example: Input "onion"
- Shows: "Red Onion" (produce), "White Onion" (produce), "Yellow Onion" (produce)
- Filters: "Kanda Lasun Onion Garlic Masala" (blocked by compound masala rule)

### 3. User Selection Interface

#### Resolution Dialog Features
- Shows up to 8 matches when similarity is between 50-95%
- **Blue highlight** for same-category matches with "Category Match" badge
- **Bold** for items with noun match
- Displays: Item name, similarity %, category, preferred store
- Sorted by: Noun match first → Similarity → Category match
- User can select best match or skip to keep original name

#### What Triggers Resolution Dialog
- "spinach" → Shows: baby spinach, frozen spinach, etc.
- "spinach big packet" → normalizes to "spinach" → Shows: baby spinach, frozen spinach
- "small pack curry leaf" → normalizes to "curry leaves" → Shows: curry leaves large packet
- "onion" → Shows: red onion, white onion, yellow onion (NOT kanda lasun masala)
- "black chana" → Shows: kala chana, chana dal (noun-sorted)
- "potato" → Shows: potatoes (NOT potato chips - filtered out)

### 4. Workflow: Match → Consolidate

#### Processing Steps
1. **Parse**: Extract ingredients from all input lists
2. **Match**: Match each ingredient with master list
   - Exact match → Use master data immediately
   - Very high similarity (>95%) → Auto-match
   - Medium similarity (50-95%) → User selection dialog
   - Low similarity (<50%) → Mark as not found
3. **User Selection**: Modal appears if partial matches need review
4. **Consolidate**: Combine items with same standardized name
   - Same unit: Sum quantities (5 lbs + 3 lbs = 8 lbs)
   - Different units: Display as comma-separated (5 lbs, 3 count)
5. **Display**: Show final consolidated results

#### Example Flow
```
Input Lists:
- List 1: "red onions 5 lbs"
- List 2: "onion 3 lbs"

Step 1 - Parse:
- "red onions" → normalized to "red onion"
- "onion" → normalized to "onion"

Step 2 - Match:
- "red onion" → Exact match → "Red Onion" (master)
- "onion" → 80% match → User selects "Red Onion" from dialog

Step 3 - Consolidate:
- "Red Onion": 5 lbs + 3 lbs = 8 lbs

Output:
- Red Onion | 8 lbs | lbs | [brand] | Produce | Lotte | RD
```

### 5. Ingredient Parsing & Normalization

#### Quantity Extraction
- **Last Number Rule**: Finds the LAST number+unit combination in each line
  - Example: "777 sambhar powder 500 gms – 1 packet" → uses "1 packet" as quantity
- **Range Handling**: Processes ranges and uses upper limit
  - Example: "5-6 tomatoes" → uses 6 tomatoes
- **Leading Decimal Support**: Quantities without a leading zero are parsed correctly
  - Example: "ginger .5 lb" → 0.5 lbs (NOT 5 count)
  - All quantity regexes use `\d*\.?\d+` pattern (zero or more digits before decimal)
- **Supported Units**: From Default Unit column in master sheet, plus standard units (lbs, oz, cups, gallons, packs, jars, bottles, boxes, bunches, cans, bags, grams, count)
- **Default Unit**: Defaults to "count" when number exists but no unit specified
- **Pantry Items**: Marks items as "pantry" when no quantity or unit specified

#### Quantity Regex Pattern (Critical)
All quantity patterns must use `\d*\.?\d+` (NOT `\d+\.?\d*`) to correctly handle decimal-only inputs like `.5`:

```javascript
// CORRECT - handles .5, 0.5, 1.5, 10, etc.
/(\d*\.?\d+)\s*(lbs?|pounds?)/gi

// WRONG - misses .5 (requires at least one leading digit)
/(\d+\.?\d*)\s*(lbs?|pounds?)/gi
```

This applies to ALL patterns in `parseQuantity`, the unit-stripping pattern in `parseIngredientLine`, and the range/fallback number patterns.

#### Name Normalization Rules
**Words removed during normalization** (so "spinach big packet" → "spinach"):
- Brand/store references: twin pack, costco, restaurant depot, trader joe
- Processing state: organic, extra virgin, virgin, unsalted, salted, fresh, frozen, canned
- Prep style: shredded, diced, chopped, sliced, whole, peeled, flat leaf, baby, purée, puree, pantry
- Size/packaging: **big, large, small, packet, pack, bundle** ← added in fix

**Special Handling:**
- Handles alternative names with "/": "ash guard/white pumpkin" preserved as-is
- Normalizes dashes and spaces for consolidation
- Removes bullet points (•, -, *, numbered lists) from input
- Removes blank lines when adding line numbers after consolidation
- Handles pluralization: tomatoes → tomato, onions → onion
- Protects parentheses content: "cheese(7.5 oz) 5" keeps "(7.5 oz)" intact
- Colors (red, green, yellow, white, black, purple) are only removed when item is NOT one of: pepper, bread, rice, chili, apple, onion, tomato, bell pepper, peas, pumpkin

### 6. Unit Conversions

**Cups to Pounds:**
- Automatically converts cups to lbs (1 cup = 0.5 lbs)
- Displays note: "Note: Cups have been converted to lbs (1 cup = 0.5 lbs)"
- Only shows conversion note when cups are actually detected in input

### 7. Visual Indicators

**Color Coding:**
- **Blue highlight in modal**: Category match (same category as likely match)
- **Green** (bg-green-100): Items consolidated from multiple lists
- **Light Amber** (bg-amber-50): Single pantry items
- **Dark Amber** (bg-amber-200): Pantry items combined from multiple lists
- **Red** (bg-red-100): Items not found in master list

**Output Table Columns:**
1. **Items**: Standardized ingredient name from master list
2. **Units**: Consolidated quantity and unit (e.g., "5 lbs", "3 count", "5 lbs, 2 count", "pantry")
3. **Default Unit**: From master list
4. **Preferred Brand**: From master list
5. **Category**: From master list
6. **Preferred Store**: From master list
7. **Alt Store**: From master list

**Sorting:**
- Non-pantry items sorted by: Preferred Store (alphabetically) → Item name (alphabetically)
- Pantry items always at the bottom, also sorted alphabetically

### 8. Line Numbering

**After Consolidation:**
- Input lists automatically get line numbers (1., 2., 3., etc.)
- Blank lines are removed
- Only actual ingredient lines are numbered
- Helps track which items have been processed

### 9. Export Features

**Copy to Google Sheets:**
- Tab-separated format (TSV)
- Includes all columns: Items, Units, Default Unit, Preferred Brand, Category, Preferred Store, Alt Store
- One-click copy to clipboard
- Confirmation message: "✓ Copied! Paste into Google Sheets"

## Technical Implementation

### Key Functions

**calculateSimilarity(str1, str2)**
- Checks for common word overlap (minimum requirement)
- Recognizes descriptor-only differences (80% similarity)
- Descriptor list: frozen, fresh, canned, dried, raw, cooked, baby, red, white, green, yellow, black, organic, large, small, whole, big, packet, pack, bundle, loose, extra, chopped, diced, sliced, ground
- Penalizes substantive differences (60% or lower)
- Returns 30% if no common words

**fuzzyMatchWithCategory(input, masterData, threshold)**
- Extracts main noun (last word) for priority matching
- Applies compound masala blocklist before scoring
- Two-pass algorithm: identify category → boost same-category items
- Noun match boost: 20%
- Category match boost: 30%
- Returns up to 8 matches sorted by: noun match → similarity → category

**consolidateIngredients()**
- Step 1: Parse all ingredients
- Step 2: Match with master list (store in temp)
- Step 3: Show modal for user selections if needed (50-95% similarity)
- Step 4: Call finalizeConsolidation() after all matches resolved

**finalizeConsolidation(matchedIngredients, stats)**
- Consolidates items with same standardized name
- Merges quantities: same unit = sum, different units = comma-separated
- Combines source lists
- Creates final sorted output

### State Management
- `masterData`: { lookup: {...}, categoryIndex: {...} }
- `pendingMatches`: { ingredientName: { item, matches } }
- `matchSuggestions`: Array of ingredient names needing selection
- `showMatchModal`: Boolean for modal visibility
- `window.tempMatchedIngredients`: Temporary storage during matching
- `window.tempListStats`: Statistics for finalization

## Common Debugging Scenarios

### Issue: Descriptor Items Not Showing (e.g., "spinach" not matching "baby spinach")
**Cause**: Descriptor list incomplete or similarity too low
**Solution**: 
- Verify descriptor list includes: frozen, fresh, canned, dried, raw, cooked, baby, red, white, green, yellow, black, organic, large, small, whole, big, packet, pack, bundle, loose, extra
- Ensure descriptor-only differences return 80% similarity
- Check that 80% is above threshold (50%) and below auto-match (95%)

### Issue: Input like "spinach big packet" not matching "Baby Spinach"
**Cause**: "big" and "packet" not in removeWords list in normalizeIngredientName, so they remain in the normalized name and break matching
**Solution**:
- Ensure removeWords includes: 'big', 'large', 'small', 'packet', 'pack', 'bundle'
- After normalization, "spinach big packet" → "spinach" → then matches via descriptor logic

### Issue: "small pack curry leaf" not matching "Curry Leaves Large Packet"
**Cause**: (a) "small" and "pack" not removed during input normalization, or (b) "large" and "packet" not in descriptor list for master side
**Solution**:
- removeWords must include 'small', 'pack' so input normalizes to "curry leaf" → synonym → "curry leaves"
- Descriptor list in calculateSimilarity must include 'large', 'packet' so "curry leaves large packet" is treated as descriptor variant of "curry leaves" → 80% similarity

### Issue: Compound masala showing for simple inputs (e.g., "Kanda Lasun Masala" for "onion")
**Cause**: No blocklist for masala/spice blend items
**Solution**:
- Add compound masala blocklist in fuzzyMatchWithCategory second pass
- Skip any key matching `/\b(masala|blend|spice mix|seasoning)\b/i` when input is ≤2 words and doesn't itself contain masala/spice

### Issue: Decimal quantity like ".5 lb" parsed as "5 count"
**Cause**: Regex pattern `\d+\.?\d*` requires at least one leading digit, so `.5` fails to match unit patterns and falls through to bare-number fallback which captures `5` only
**Solution**:
- Change ALL quantity regexes from `\d+\.?\d*` to `\d*\.?\d+`
- This applies to: all 19 patterns in `parseQuantity`, the unitPattern in `parseIngredientLine`, and the range/fallback number matchers in both functions

### Issue: Too Many Irrelevant Matches
**Cause**: No word overlap check or threshold too low
**Solution**:
- Verify common word check returns 30% for no overlap
- Ensure threshold is set to 0.5 (50%)
- Check category filtering is active for strong category signals (>80% top match)

### Issue: Noun-Based Sorting Not Working
**Cause**: Main noun not extracted correctly
**Solution**:
- Verify noun extraction: last word of input (e.g., "chana" from "black chana")
- Check 20% noun match boost is applied
- Ensure sorting prioritizes nounMatch flag first

### Issue: Quantities Not Displaying
**Cause**: Quantities not copied during consolidation initialization
**Solution**:
- Check `quantities: { ...matchInfo.quantities }` in first occurrence
- Verify quantities are summed for same units in merge logic
- Ensure quantityStrings are joined with ", " in output

## Best Practices

### Master List Management
1. Keep category names consistent across items
2. Use specific categories (e.g., "Produce", "Spice", "Dairy")
3. Ensure all items have category assigned for better matching
4. Include both base items and descriptor variants (e.g., "Spinach" and "Baby Spinach")
5. Test descriptor matching with color/processing/size variations

### Matching Threshold Tuning
- **>95%**: Auto-match (very high confidence)
- **80%**: Descriptor-only differences (show in dialog)
- **60-70%**: Substantive but related items (show in dialog)
- **50-60%**: Marginal matches (show if no better options)
- **<50%**: Not found (too dissimilar)

### Testing Scenarios
Test with these cases:
- [ ] "spinach" → Should show: baby spinach, frozen spinach
- [ ] "spinach big packet" → Should normalize to "spinach" → show: baby spinach, frozen spinach
- [ ] "small pack curry leaf" → Should normalize to "curry leaves" → show: curry leaves large packet
- [ ] "onion" → Should show: red onion, white onion, yellow onion; should NOT show kanda lasun masala
- [ ] "garlic" → Should NOT show: kanda lasun masala
- [ ] "onion garlic masala" → SHOULD show: kanda lasun masala
- [ ] "potato" → Should NOT show: potato chips
- [ ] "black chana" → Should show: kala chana (first), chana dal (second)
- [ ] "lemon" → Should NOT auto-match: lemon juice
- [ ] "ginger .5 lb" → Should parse as 0.5 lbs (NOT 5 count)
- [ ] "ginger 0.5 lb" → Should parse as 0.5 lbs
- [ ] Quantities display correctly in Units column
- [ ] Same unit quantities sum: 5 lbs + 3 lbs = 8 lbs
- [ ] Different unit quantities display: 5 lbs, 3 count

## Configuration

### In Code Setup
```javascript
const SHEET_ID = '1yJyhd23X77zH_ITtQu1AtNTI-Ak9KnorymsPJgEJKMc';
const SHEET_NAME = 'Master List_Annapurna'; // Your sheet tab name
const API_KEY = 'YOUR_GOOGLE_SHEETS_API_KEY_HERE';
```

### Descriptor List (for 80% similarity in calculateSimilarity)
```javascript
const descriptors = ['frozen', 'fresh', 'canned', 'dried', 'raw', 'cooked', 
                    'baby', 'red', 'white', 'green', 'yellow', 'black', 
                    'organic', 'large', 'small', 'whole', 'chopped', 'diced',
                    'sliced', 'ground', 'big', 'packet', 'pack', 'bundle',
                    'loose', 'extra'];
```

### Remove Words List (for normalizeIngredientName)
```javascript
const removeWords = [
    'twin pack', 'costco', 'restaurant depot', 'trader joe',
    'organic', 'extra virgin', 'virgin', 'unsalted', 'salted',
    'fresh', 'frozen', 'canned',
    'shredded', 'diced', 'chopped', 'sliced', 'whole', 'peeled',
    'flat leaf', 'baby', 'purée', 'puree', 'pantry',
    'big', 'large', 'small', 'packet', 'pack', 'bundle'  // ← size/packaging words
];
```

## Example Usage Scenarios

### Scenario 1: Descriptor Matching
**Input:** "spinach"
**Master List:** Baby Spinach, Frozen Spinach, Fresh Spinach
**Result:** Shows all three with 80% similarity, user selects appropriate one

### Scenario 2: Descriptor Matching with Size/Pack Words in Input
**Input:** "spinach big packet"
**Normalization:** "big" and "packet" removed → "spinach"
**Master List:** Baby Spinach, Frozen Spinach
**Result:** Shows both with 80% similarity, user selects appropriate one

### Scenario 3: Descriptor Matching with Size/Pack Words in Master
**Input:** "small pack curry leaf"
**Normalization:** "small", "pack" removed → "curry leaf" → synonym → "curry leaves"
**Master List:** Curry Leaves Large Packet
**Result:** "large" and "packet" treated as descriptors → 80% similarity → shows in dialog

### Scenario 4: Color Variants
**Input:** "onion"
**Master List:** Red Onion, White Onion, Yellow Onion
**Result:** Shows all three with 80% similarity, sorted alphabetically within category

### Scenario 5: Compound Masala Blocked
**Input:** "onion"
**Master List:** Red Onion, Kanda Lasun Masala (Onion Garlic)
**Result:** Shows Red Onion; Kanda Lasun Masala is blocked by compound masala rule

### Scenario 6: Noun-Based Priority
**Input:** "black chana"
**Master List:** Kala Chana, Chana Dal, Bengal Gram
**Process:**
1. Extract noun: "chana"
2. Boost items containing "chana": Kala Chana, Chana Dal
3. Sort by noun match
**Result:** Shows Kala Chana first, then Chana Dal

### Scenario 7: Avoiding False Matches
**Input:** "potato"
**Master List:** Potatoes, Potato Chips, Potato Starch
**Result:** Shows Potatoes (80%), may show Potato Starch (60%), excludes Potato Chips (30%)

### Scenario 8: Quantity Consolidation
**Input:**
- List 1: "Red Onion 5 lbs"
- List 2: "onion 3 lbs" → user selects "Red Onion"
- List 3: "Red Onion 2 count"

**Output:**
- Red Onion | 8 lbs, 2 count | lbs | [brand] | Produce | Lotte | RD

### Scenario 9: Leading Decimal Quantity
**Input:** "ginger .5 lb"
**Result:** Parsed as 0.5 lbs (regex `\d*\.?\d+` captures ".5" correctly)
**Wrong result to avoid:** 5 count (happens if regex is `\d+\.?\d*` which misses the dot)

## Performance Considerations
- Noun extraction: O(1) per item
- Two-pass fuzzy matching: O(n) + O(n) = O(n)
- Category indexing at load time (one-time cost)
- Consolidation after matching: O(n) single pass
- Up to 8 matches per item (configurable)
- Efficient for typical use cases (3-10 lists, 20-100 items each)

## Changelog

### Bug Fixes Applied
1. **Descriptor matching for size/pack words in input names** (e.g., "spinach big packet"):
   - Added 'big', 'large', 'small', 'packet', 'pack', 'bundle' to `removeWords` in `normalizeIngredientName`
   - These words are stripped before matching so the core ingredient name is cleanly extracted

2. **Descriptor matching for size/pack words in master list names** (e.g., "Curry Leaves Large Packet"):
   - Added 'large', 'small', 'big', 'packet', 'pack', 'bundle', 'loose', 'extra' to `descriptors` list in `calculateSimilarity`
   - Master items with only these extra words now score 80% (resolution dialog) instead of failing to match

3. **Compound masala/spice blend suppression** (e.g., "Kanda Lasun Masala" not shown for "onion"):
   - Added blocklist check in `fuzzyMatchWithCategory` second pass
   - Any master item matching `/\b(masala|blend|spice mix|seasoning)\b/i` is skipped when input is ≤2 words and doesn't contain masala/spice keywords

4. **Leading decimal quantity parsing** (e.g., "ginger .5 lb" → 0.5 lbs not 5 count):
   - Changed all quantity regex patterns from `\d+\.?\d*` to `\d*\.?\d+`
   - Applied to all 19 unit patterns in `parseQuantity`, the unitPattern in `parseIngredientLine`, and the range/fallback number regexes in both functions


5. **Submit to Google Sheets (Friday/Sunday shopping sheet)**:
   - Added Submit button alongside Copy to Google Sheets button
   - Friday/Sunday dropdown lets user select target shopping day
   - Button label updates live: "Submit — Friday MM/DD/YYYY" or "Submit — Sunday MM/DD/YYYY"
   - Upcoming day calculated dynamically (if today IS that day, targets next occurrence)
   - Creates a new tab named MM/DD/YYYY in the shopping sheet, then writes all 7 columns
   - If tab already exists, overwrites gracefully
   - Requires OAuth sign-in (see OAuth section below)

6. **Google OAuth 2.0 sign-in for write access**:
   - Read (master list) continues using API key — no auth needed
   - Write (submit) requires OAuth token — uses popup-based implicit flow
   - Sign-in panel displayed below master data status bar
   - Shows "Signed in ✓" when authenticated, "Sign Out" button to revoke
   - Token stored in React state (not localStorage) — must sign in each session
   - Auth flow uses manual OAuth2 popup + `oauth2callback.html` postMessage pattern
   - Scope: `https://www.googleapis.com/auth/drive.file` (restricted to files the app accesses)

---

## 10. Submit to Shopping Sheet

### Overview
After consolidation, the user can submit the shopping list directly to a designated Google Sheet. A new tab is created for the selected shopping day's date.

### Target Sheet
- **Sheet ID**: `1UbBBupMUuF8m4Ax8B8Vfid2ojqa0nt4bbOxG4JpmyI4`
- **Tab name format**: `MM/DD/YYYY` (e.g., `02/28/2025`)
- **Columns written**: Items, Units, Default Unit, Preferred Brand, Category, Preferred Store, Alt Store

### Day Selection Dropdown
- Options: **Friday** or **Sunday**
- Defaults to Friday
- Upcoming date calculated from current date:
  ```javascript
  function getUpcomingDay(targetDay) {
      // targetDay: 0=Sunday, 5=Friday
      const today = new Date();
      const day = today.getDay();
      const daysUntil = (targetDay - day + 7) % 7 || 7; // if today IS that day, go to next occurrence
      const result = new Date(today);
      result.setDate(today.getDate() + daysUntil);
      return result;
  }
  ```

### Submit Flow
1. User selects Friday or Sunday from dropdown
2. User clicks Submit button (shows upcoming date)
3. If not signed in → shows error "Please sign in with Google first"
4. Creates new tab named with the date via `spreadsheets:batchUpdate`
5. Writes header + all rows via `spreadsheets/values/{range}` PUT
6. Shows status: `✓ Submitted to tab "02/28/2025"`

### New State Variables
- `submitDay`: `'friday'` or `'sunday'` — controls target day
- `submitStatus`: Status message string for submit feedback
- `oauthToken`: OAuth access token (null when signed out)
- `oauthUser`: Display string shown in sign-in panel

---

## 11. Google OAuth Authentication

### Architecture
The app uses a **manual OAuth2 implicit flow** with a popup window and `postMessage` to avoid deprecated libraries and `storagerelay://` errors.

### Two Files Required
1. **`index.html`** — main app
2. **`oauth2callback.html`** — OAuth redirect target (must be in same directory)

Both files must be served from the same origin (e.g. `http://localhost:8000` or GitHub Pages).

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
Minimal redirect handler — parses `access_token` from URL hash and posts to opener:
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

### OAuth Scope
```
https://www.googleapis.com/auth/drive.file
```
- Only accesses files created by or explicitly opened by the app
- Cannot read/modify other spreadsheets in the user's Drive
- Considered "sensitive" by Google — requires test user setup OR app verification

### Google Cloud Console Setup (One-Time)
1. Create project at console.cloud.google.com
2. Enable **Google Sheets API**
3. **APIs & Services → Credentials → Create OAuth 2.0 Client ID**
   - Type: Web application
4. **Authorised JavaScript Origins**:
   - `http://localhost`
   - `http://localhost:8000`
   - `https://chinmayasomnathva.github.io`
5. **Authorised Redirect URIs**:
   - `http://localhost:8000/oauth2callback.html`
   - `https://chinmayasomnathva.github.io/ingredient-consolidator/oauth2callback.html`
6. **OAuth consent screen**:
   - Fill in App name, support email, developer contact email
   - Add scope: `https://www.googleapis.com/auth/drive.file`
   - Publishing status: **Testing**
   - Add your Google account email as a **Test User**
   - Do NOT need to publish/verify for personal use

### Constants in Code
```javascript
const OAUTH_CLIENT_ID = '905820431932-rpieibglpo732pk1ia56bo9mub4b3qtk.apps.googleusercontent.com';
const SUBMIT_SHEET_ID = '1UbBBupMUuF8m4Ax8B8Vfid2ojqa0nt4bbOxG4JpmyI4';
```

### Common OAuth Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `redirect_uri=storagerelay://` | Using deprecated GSI/gapi library | Use manual OAuth2 popup with oauth2callback.html |
| `Error 403: access_denied` | App in Testing mode, account not a test user | Add email to Test Users in OAuth consent screen |
| `Error 400: invalid_request` | Opening file:// directly | Serve via `python3 -m http.server 8000` |
| `Error 500` on consent screen | OAuth consent screen not fully configured | Fill in App name, support email, developer contact, and scopes |
| `access_type=online ... FedCM` | Using gapi.auth2 (deprecated) | Switch to manual OAuth2 popup flow |

### Deployment: Local vs GitHub Pages
- **Local**: Run `python3 -m http.server 8000` in the folder, open `http://localhost:8000/index.html`
- **GitHub Pages**: Push both `index.html` and `oauth2callback.html` to repo root, enable Pages from main branch
- GitHub Pages URL: `https://chinmayasomnathva.github.io/ingredient-consolidator/`


7. **Legend text fix** — "Not found in master" shortened to "Not found"

8. **Google Sheet formatting on Submit**:
   - After writing data, a formatting batchUpdate pass runs automatically
   - Header row: bold white text on dark gray background, frozen (stays visible when scrolling)
   - Row colors match the on-screen table exactly (green/amber/red/white)
   - All 7 columns auto-resized to fit content
   - Sheet ID fetched dynamically from spreadsheet metadata to correctly target the new tab

---

## 12. Google Sheet Formatting on Submit

### Overview
After data is written to the new tab, a second API call applies formatting so the Google Sheet visually matches the on-screen consolidated table.

### Formatting Steps (in order)
1. **Write data** — PUT values to the new tab
2. **Fetch sheet metadata** — GET spreadsheet properties to find the new tab's `sheetId`
3. **batchUpdate formatting** — apply all formatting in a single API call
4. **Status update** — show "✓ Submitted to tab MM/DD/YYYY"

### Why Fetch Sheet Metadata
The Sheets API formatting calls require a numeric `sheetId`, not the tab name string. After creating the tab, the sheetId must be looked up:
```javascript
const metaRes = await fetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${SUBMIT_SHEET_ID}?fields=sheets.properties`,
    { headers: { 'Authorization': `Bearer ${freshToken}` } }
);
const meta = await metaRes.json();
const sheetObj = meta.sheets.find(s => s.properties.title === tabName);
const sheetId = sheetObj ? sheetObj.properties.sheetId : 0;
```

### Formatting Applied

**Header row (row 1):**
- Bold text, white foreground
- Dark gray background: `{ red: 0.26, green: 0.26, blue: 0.26 }`
- verticalAlignment: MIDDLE
- Frozen (frozenRowCount: 1)

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

**Column auto-resize:**
```javascript
{
    autoResizeDimensions: {
        dimensions: { sheetId, dimension: 'COLUMNS', startIndex: 0, endIndex: 7 }
    }
}
```

### Full Submit Flow (4 steps)
```
Step 1: POST spreadsheets:batchUpdate → addSheet (create tab named MM/DD/YYYY)
Step 2: PUT spreadsheets/values/{range} → write header + data rows
Step 3: GET spreadsheets?fields=sheets.properties → find numeric sheetId of new tab
Step 4: POST spreadsheets:batchUpdate → apply header format + row colors + freeze + auto-resize
```

### Debugging Formatting Issues

**Issue: Wrong row gets colored**
- Check `consolidated` array index matches row index (data starts at row index 1, not 0)
- `startRowIndex: i + 1, endRowIndex: i + 2` for data row i

**Issue: Colors look different from on-screen table**
- Google Sheets RGB values are 0.0–1.0 floats, not 0–255
- Verify color conversion: Tailwind amber-200 (#fde68a) = r:0.992, g:0.906, b:0.541

**Issue: sheetId is wrong (formatting goes to wrong tab)**
- Always look up sheetId dynamically after tab creation — never hardcode it
- If tab already existed (duplicate submit), the lookup still returns the correct sheetId

