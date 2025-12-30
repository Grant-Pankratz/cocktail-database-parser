# Cocktail Database - Update Requirements

**Date:** December 30, 2025  
**Status:** All requirements implemented

---

## 1. Quantity Rounding - DECIMAL(10,2) ✅

**Requirement:** oz should always round to 2 decimal places

**Implementation:**
```sql
-- Change ALL quantity fields from REAL to DECIMAL(10,2)

ALTER TABLE cocktail_ingredients MODIFY COLUMN quantity DECIMAL(10,2);
ALTER TABLE cocktail_ingredients MODIFY COLUMN min_quantity DECIMAL(10,2);
ALTER TABLE cocktail_ingredients MODIFY COLUMN max_quantity DECIMAL(10,2);

ALTER TABLE prep_ingredients MODIFY COLUMN quantity DECIMAL(10,2);
ALTER TABLE batches MODIFY COLUMN yield DECIMAL(10,2);
ALTER TABLE batch_usage MODIFY COLUMN amount_used DECIMAL(10,2);
ALTER TABLE spirit_brands MODIFY COLUMN abv DECIMAL(5,2);
ALTER TABLE liqueurs MODIFY COLUMN abv_min DECIMAL(5,2);
ALTER TABLE liqueurs MODIFY COLUMN abv_max DECIMAL(5,2);
ALTER TABLE beverage_preps MODIFY COLUMN yield DECIMAL(10,2);
```

**Why DECIMAL not REAL:**
- REAL uses floating-point arithmetic (imprecise)
- DECIMAL uses fixed-point arithmetic (precise)
- 0.75 oz stays exactly 0.75, not 0.7499999...
- Prevents rounding errors in inventory calculations

**Examples:**
```
0.75 oz = 0.75 ✓
2.25 oz = 2.25 ✓
1.50 oz = 1.50 ✓
0.25 oz = 0.25 ✓
```

---

## 2. Dual Spirit/Liqueur/Wine Types ✅

**Requirement:** Both type and type+brand queries (whisky AND Buffalo Trace whisky)

**Implementation:**
```sql
cocktail_ingredients.ingredient_type can be:
  - 'spirit'          (rare: generic spirit)
  - 'spirit_brand'    (Plymouth Gin, Elijah Craig Bourbon)
  - 'liqueur'         (rare: generic liqueur)
  - 'liqueur_brand'   (St. Germain, Luxardo Maraschino)
  - 'wine'            (rare: generic wine)
  - 'wine_brand'      (Dolin Blanc Vermouth, La Gitana Fino Sherry)
  - 'prep_ref'        (Syrup: Simple, Garnish: Coconut Marshmallow)
  - 'batch_ref'       (Flamenco Tears Batch)
  - 'other'           (fresh lemon juice, bitters, salt)
```

**Query Examples:**

```sql
-- All bourbon recipes (any brand)
SELECT DISTINCT c.name
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
JOIN spirit_brands sb ON ci.ingredient_id = sb.id
JOIN spirits s ON sb.spirit_id = s.id
WHERE ci.ingredient_type = 'spirit_brand'
  AND s.name = 'bourbon'

-- Buffalo Trace bourbon only
SELECT DISTINCT c.name
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
JOIN spirit_brands sb ON ci.ingredient_id = sb.id
WHERE ci.ingredient_type = 'spirit_brand'
  AND sb.brand_name = 'Buffalo Trace'

-- All elderflower liqueur recipes (any brand)
SELECT DISTINCT c.name
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
JOIN liqueur_brands lb ON ci.ingredient_id = lb.id
JOIN liqueurs l ON lb.liqueur_id = l.id
WHERE ci.ingredient_type = 'liqueur_brand'
  AND l.name = 'elderflower liqueur'
```

---

## 3. Recipe Scaling UI - ½x, 1x, 2x, 3x Buttons ✅

**Requirement:** UI with scaling buttons, default 1x recipe

**Implementation:**
```sql
ALTER TABLE cocktails ADD COLUMN base_yield INTEGER DEFAULT 1;
-- 1 = standard single drink
-- 2+ = batch recipe yielding multiple drinks
```

**UI Buttons:**
```
┌──────────────────────────────────────────────┐
│  ½x  │   1x (default)  │  2x  │  3x   │
└──────────────────────────────────────────────┘
```

**How Scaling Works:**
```python
# Celine Fizz with base_yield = 1

User selects 2x button:
  scale_factor = 2

  For each ingredient:
    scaled_qty = original_qty * scale_factor

Example:
  2.00 oz Plymouth Gin × 2 = 4.00 oz Plymouth Gin
  0.50 oz St. Germain × 2 = 1.00 oz St. Germain
  0.75 oz lemon juice × 2 = 1.50 oz lemon juice

# Flamenco Tears Batch with base_yield = 3

User selects ½x button:
  scale_factor = 0.5
  total_portions = base_yield * scale_factor = 1.5 drinks

User selects 2x button:
  scale_factor = 2
  total_portions = base_yield * scale_factor = 6 drinks
```

---

## 4. Adjustable Ingredients with Min/Max Ranges ✅

**Requirement:** Support min/max ranges for flexible ingredients like "to taste"

**Implementation:**
```sql
ALTER TABLE cocktail_ingredients ADD COLUMN is_adjustable BOOLEAN DEFAULT false;
ALTER TABLE cocktail_ingredients ADD COLUMN min_quantity DECIMAL(10,2);
ALTER TABLE cocktail_ingredients ADD COLUMN max_quantity DECIMAL(10,2);
```

**Example: Fresh Lemon Juice with Range**
```sql
INSERT INTO cocktail_ingredients VALUES (
  cocktail_id: 123,
  ingredient_type: 'other',
  ingredient_id: null,
  quantity: 0.75,
  unit: 'oz',
  is_adjustable: true,
  min_quantity: 0.50,
  max_quantity: 1.00,
  ingredient_notes: 'adjust to taste',
  sort_order: 3
)
```

**Example: "To Taste" with No Range**
```sql
INSERT INTO cocktail_ingredients VALUES (
  cocktail_id: 123,
  ingredient_type: 'other',
  ingredient_id: null,
  quantity: null,
  unit: null,
  is_adjustable: true,
  min_quantity: null,
  max_quantity: null,
  ingredient_notes: 'to taste',
  sort_order: 2
)
```

**UI Display:**
```
□ Fresh lemon juice    0.75 oz    (0.50 - 1.00 oz)    adjust to taste
□ Salt                 "to taste"  (optional)
```

---

## 5. Cocktail Preparation Steps (Method) ✅

**Requirement:** Method should provide full context (like "Flamenco Tears" example)

**Implementation:**
```sql
CREATE TABLE cocktail_steps (
  id INTEGER PRIMARY KEY AUTO_INCREMENT,
  cocktail_id INTEGER NOT NULL,
  step_order INTEGER NOT NULL,
  description TEXT NOT NULL,
  technique TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (cocktail_id) REFERENCES cocktails(id),
  INDEX (cocktail_id),
  INDEX (cocktail_id, step_order)
)
```

**Technique Values (Controlled Vocabulary):**
- `build` - Add ingredients directly to glass
- `shake` - Shake with ice in cocktail shaker
- `stir` - Stir in mixing glass
- `strain` - Pour through strainer
- `float` - Layer on top using back of spoon
- `muddle` - Crush fresh ingredients
- `chill` - Pre-chill glass
- `flame` - Use flame/torch
- `blend` - Use blender

**Example: Flamenco Tears Full Recipe**
```sql
INSERT INTO cocktail_steps VALUES
  (1, 123, 1, "Add all spirits to mixing glass", "build"),
  (2, 123, 2, "Add bitters", "build"),
  (3, 123, 3, "Fill mixing glass with ice and stir (15-20 seconds)", "stir"),
  (4, 123, 4, "Strain into SOF glass filled with block ice", "strain"),
  (5, 123, 5, "Float 1/2 tsp Creme de Cassis on top", "float")
```

**Example: Flamenco Tears Service Execution**
```sql
Service execution uses batch reference:
  2.75 oz Flamenco Tears Batch [pre-made component]
  1/2 tsp Giffard Creme de Cassis Imperial
  2 dashes 1821 Havana Hide Bitters
  1 dash Aromatic Bitters
  Method: stir/strain
```

---

## 6. Superadmin Recipe Updates with Version Control ✅

**Requirement:** Version control for recipe updates, superadmin can update with tracking

**Implementation:**
```sql
CREATE TABLE cocktail_versions (
  id INTEGER PRIMARY KEY AUTO_INCREMENT,
  cocktail_id INTEGER NOT NULL,
  version INTEGER NOT NULL,
  ingredients JSON NOT NULL,
  steps JSON NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by INTEGER NOT NULL,
  reason TEXT,
  FOREIGN KEY (cocktail_id) REFERENCES cocktails(id),
  FOREIGN KEY (created_by) REFERENCES users(id),
  UNIQUE KEY (cocktail_id, version),
  INDEX (cocktail_id),
  INDEX (created_at)
)
```

**Workflow:**

1. **Create Recipe (Auto Version 1)**
```sql
INSERT INTO cocktails (name, glass_type, is_batch, base_yield)
VALUES ('Celine Fizz', 'coupe', false, 1);

INSERT INTO cocktail_ingredients VALUES (...);
INSERT INTO cocktail_steps VALUES (...);

INSERT INTO cocktail_versions
VALUES (
  cocktail_id: 123,
  version: 1,
  ingredients: JSON [...all ingredients...],
  steps: JSON [...all steps...],
  created_by: admin_id,
  reason: 'Initial recipe creation'
);
```

2. **Update Recipe (Create New Version)**
```sql
-- Superadmin updates St. Germain from 0.50 to 0.75 oz

UPDATE cocktail_ingredients
SET quantity = 0.75
WHERE cocktail_id = 123 AND ingredient_type = 'liqueur_brand' AND ingredient_id = 202;

INSERT INTO cocktail_versions
VALUES (
  cocktail_id: 123,
  version: 2,
  ingredients: JSON [...updated ingredients...],
  steps: JSON [...same steps...],
  created_by: admin_id,
  reason: 'Adjusted St. Germain quantity for better floral balance'
);
```

3. **Query Current Recipe**
```sql
-- Gets latest version (Version 2)
SELECT * FROM cocktails WHERE id = 123;
```

4. **Query Historical Recipe**
```sql
-- Gets original recipe (Version 1)
SELECT ingredients, steps FROM cocktail_versions
WHERE cocktail_id = 123 AND version = 1;
```

5. **Revert to Previous Version**
```sql
-- Get Version 1
SELECT ingredients, steps FROM cocktail_versions
WHERE cocktail_id = 123 AND version = 1;

-- Update current cocktail with Version 1 data
-- Create Version 3 as reversion
INSERT INTO cocktail_versions
VALUES (
  cocktail_id: 123,
  version: 3,
  ingredients: JSON [...from Version 1...],
  steps: JSON [...from Version 1...],
  created_by: admin_id,
  reason: 'Reverted to original recipe due to feedback'
);
```

**Benefits:**
- ✅ Complete audit trail
- ✅ Know what was served on date X
- ✅ Track recipe evolution
- ✅ Admin accountability
- ✅ Can revert to old versions

---

## 7. Field Rename: ingredient_notes ✅

**Requirement:** Rename `notes` to `ingredient_notes` in cocktail_ingredients

**Implementation:**
```sql
ALTER TABLE cocktail_ingredients RENAME COLUMN notes TO ingredient_notes;
```

**Why This Matters:**
- Distinguishes from `beverage_preps.notes`
- Shows this field documents the ingredient specifically
- Consistent with schema intent

**Examples:**
```
ingredient_notes: "squeeze fresh, not bottled"
ingredient_notes: "adjust to taste"
ingredient_notes: "thin lemon wheel, twisted"
```

---

## 8. Seasonality - Future Enhancement ✅

**Status:** Deferred for future implementation

**When Ready:**
```sql
CREATE TABLE ingredient_availability (
  id INTEGER PRIMARY KEY AUTO_INCREMENT,
  ingredient_type TEXT NOT NULL,
  ingredient_id INTEGER NOT NULL,
  season TEXT NOT NULL (spring, summer, fall, winter),
  availability TEXT NOT NULL (in_stock, limited, seasonal_only, out_of_stock),
  months TEXT (e.g., "6-8" for June-August),
  price_range TEXT,
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX (ingredient_type, ingredient_id),
  INDEX (season)
)
```

**Enables:**
- "What cocktails can I make right now?" (season-aware)
- Inventory warnings
- Cost tracking by season

---

## Summary of Changes

| # | Feature | Status | Impact |
|---|---------|--------|--------|
| 1 | Quantity Rounding (DECIMAL 10,2) | ✅ | Prevents floating-point errors |
| 2 | Dual Spirit/Liqueur/Wine Types | ✅ | Query by both type and brand |
| 3 | Recipe Scaling UI (½x,1x,2x,3x) | ✅ | Supports batch recipes |
| 4 | Adjustable Ingredients | ✅ | Min/max ranges and "to taste" |
| 5 | Cocktail Steps (Method) | ✅ | Full preparation context |
| 6 | Version Control | ✅ | Complete recipe history |
| 7 | ingredient_notes Field | ✅ | Clarity improvement |
| 8 | Seasonality | 💬 | Planned for future |

---

## Database Migration Checklist

```sql
-- Phase 1: Add new columns and tables
ALTER TABLE cocktails ADD COLUMN base_yield INTEGER DEFAULT 1;
ALTER TABLE cocktail_ingredients ADD COLUMN is_adjustable BOOLEAN DEFAULT false;
ALTER TABLE cocktail_ingredients ADD COLUMN min_quantity DECIMAL(10,2);
ALTER TABLE cocktail_ingredients ADD COLUMN max_quantity DECIMAL(10,2);

-- Create new tables
CREATE TABLE cocktail_steps (...);
CREATE TABLE cocktail_versions (...);

-- Phase 2: Convert data types to DECIMAL
ALTER TABLE cocktail_ingredients MODIFY COLUMN quantity DECIMAL(10,2);
ALTER TABLE cocktail_ingredients MODIFY COLUMN min_quantity DECIMAL(10,2);
ALTER TABLE cocktail_ingredients MODIFY COLUMN max_quantity DECIMAL(10,2);
ALTER TABLE prep_ingredients MODIFY COLUMN quantity DECIMAL(10,2);
ALTER TABLE batches MODIFY COLUMN yield DECIMAL(10,2);
ALTER TABLE batch_usage MODIFY COLUMN amount_used DECIMAL(10,2);
ALTER TABLE spirit_brands MODIFY COLUMN abv DECIMAL(5,2);
ALTER TABLE liqueurs MODIFY COLUMN abv_min DECIMAL(5,2);
ALTER TABLE liqueurs MODIFY COLUMN abv_max DECIMAL(5,2);
ALTER TABLE beverage_preps MODIFY COLUMN yield DECIMAL(10,2);

-- Phase 3: Rename field
ALTER TABLE cocktail_ingredients RENAME COLUMN notes TO ingredient_notes;

-- Phase 4: Create initial versions for existing recipes
INSERT INTO cocktail_versions
SELECT 
  NULL,
  c.id,
  1,
  JSON [...ingredients...],
  JSON [...steps...],
  c.created_at,
  1,
  'Initial version (migrated from existing recipe)'
FROM cocktails c;
```

---

## Next Steps

1. ✅ Update ARCHITECTURE.md with new schema diagrams
2. ✅ Create migration scripts
3. ✅ Update implementation guide with code examples
4. 💬 Develop UI components for scaling buttons
5. 💬 Add admin dashboard for version history
6. 💬 Implement seasonality table when ready
