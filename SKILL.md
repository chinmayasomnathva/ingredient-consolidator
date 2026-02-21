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
Items differing only by descriptors get **80% similarity** (triggers resolution):
- Colors: red, white, green, yellow, black
- Processing: frozen, fresh, canned, dried, raw, cooked
- Size/Type: baby, organic, large, small, whole

Examples:
- "spinach" → "baby spinach" = 80%
- "spinach" → "frozen spinach" = 80%
- "onion" → "red onion" = 80%
- "onion" → "white onion" = 80%

**Substantive Differences:**
Items with non-descriptor differences get lower similarity:
- "potato" → "potato chips" = 60% (chips is not a descriptor)
- "lemon" → "lemon juice" = 60% (juice is not a descriptor)

**No Common Words:**
Items with no word overlap get very low similarity:
- "potatoes" → "potato chips" = 30% (no true overlap after stemming check)

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
- Filters: "Kanda Lasun Onion Garlic Masala" (spice, low similarity)

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
- "onion" → Shows: red onion, white onion, yellow onion
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
- **Supported Units**: From Default Unit column in master sheet, plus standard units (lbs, oz, cups, gallons, packs, jars, bottles, boxes, bunches, cans, bags, grams, count)
- **Default Unit**: Defaults to "count" when number exists but no unit specified
- **Pantry Items**: Marks items as "pantry" when no quantity or unit specified

#### Name Normalization Rules
**Special Handling:**
- Handles alternative names with "/": "ash guard/white pumpkin" preserved as-is
- Normalizes dashes and spaces for consolidation
- Removes bullet points (•, -, *, numbered lists) from input
- Removes blank lines when adding line numbers after consolidation
- Handles pluralization: tomatoes → tomato, onions → onion
- Protects parentheses content: "cheese(7.5 oz) 5" keeps "(7.5 oz)" intact

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
- Penalizes substantive differences (60% or lower)
- Returns 30% if no common words

**fuzzyMatchWithCategory(input, masterData, threshold)**
- Extracts main noun (last word) for priority matching
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
- Verify descriptor list includes: frozen, fresh, canned, dried, raw, cooked, baby, red, white, green, yellow, black, organic, large, small, whole
- Ensure descriptor-only differences return 80% similarity
- Check that 80% is above threshold (50%) and below auto-match (95%)

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
5. Test descriptor matching with color/processing variations

### Matching Threshold Tuning
- **>95%**: Auto-match (very high confidence)
- **80%**: Descriptor-only differences (show in dialog)
- **60-70%**: Substantive but related items (show in dialog)
- **50-60%**: Marginal matches (show if no better options)
- **<50%**: Not found (too dissimilar)

### Testing Scenarios
Test with these cases:
- [ ] "spinach" → Should show: baby spinach, frozen spinach
- [ ] "onion" → Should show: red onion, white onion, yellow onion
- [ ] "potato" → Should NOT show: potato chips
- [ ] "black chana" → Should show: kala chana (first), chana dal (second)
- [ ] "lemon" → Should NOT auto-match: lemon juice
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

### Descriptor List (for 80% similarity)
```javascript
const descriptors = ['frozen', 'fresh', 'canned', 'dried', 'raw', 'cooked', 
                    'baby', 'red', 'white', 'green', 'yellow', 'black', 
                    'organic', 'large', 'small', 'whole'];
```

## Example Usage Scenarios

### Scenario 1: Descriptor Matching
**Input:** "spinach"
**Master List:** Baby Spinach, Frozen Spinach, Fresh Spinach
**Result:** Shows all three with 80% similarity, user selects appropriate one

### Scenario 2: Color Variants
**Input:** "onion"
**Master List:** Red Onion, White Onion, Yellow Onion
**Result:** Shows all three with 80% similarity, sorted alphabetically within category

### Scenario 3: Noun-Based Priority
**Input:** "black chana"
**Master List:** Kala Chana, Chana Dal, Bengal Gram
**Process:**
1. Extract noun: "chana"
2. Boost items containing "chana": Kala Chana, Chana Dal
3. Sort by noun match
**Result:** Shows Kala Chana first, then Chana Dal

### Scenario 4: Avoiding False Matches
**Input:** "potato"
**Master List:** Potatoes, Potato Chips, Potato Starch
**Result:** Shows Potatoes (80%), may show Potato Starch (60%), excludes Potato Chips (30%)

### Scenario 5: Quantity Consolidation
**Input:**
- List 1: "Red Onion 5 lbs"
- List 2: "onion 3 lbs" → user selects "Red Onion"
- List 3: "Red Onion 2 count"

**Output:**
- Red Onion | 8 lbs, 2 count | lbs | [brand] | Produce | Lotte | RD

## Performance Considerations
- Noun extraction: O(1) per item
- Two-pass fuzzy matching: O(n) + O(n) = O(n)
- Category indexing at load time (one-time cost)
- Consolidation after matching: O(n) single pass
- Up to 8 matches per item (configurable)
- Efficient for typical use cases (3-10 lists, 20-100 items each)
