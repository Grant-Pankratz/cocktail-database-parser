# Cocktail Database Architecture - Visual Guide

## Core Concept

**PREPS** are reusable recipe templates (Syrup: Simple, Matcha Infusion, etc.) that are used by many cocktails. A prep is made once and referenced by multiple recipes—it's not tied to a specific cocktail.

**BATCHES** are cocktails marked with `is_batch = true` to indicate they yield more than one standard drink. This is a display flag for the user, not a separate database structure.

**BRANDS** critically affect taste. Spirit brands have different taste profiles (Plymouth Gin vs Tanqueray), liqueur brands vary dramatically (St. Germain vs other elderflower liqueurs are VERY different), and wine brands matter for consistency.

**UNITS** are standardized:
- **Volume measurements** → oz (fluid ounces)
  - 1 oz = 1 jigger (volume-based)
  - Convert: ml → oz, cup → oz, etc.
- **Weight measurements** → grams (g)
  - Use for solids: sugar, fruit, herbs, garnishes
  - All weight stored as grams internally

---

## Entity Relationship Diagram (Text Format)

```
┌─────────────────────────────────────────────────────────────────┐
│                          SPIRITS                                │
├─────────────────────────────────────────────────────────────────┤
│ PK: id (INTEGER)                                                │
│ FK: none                                                        │
│ Fields:                                                         │
│  - category: TEXT  (gin, rum, vodka, tequila, whiskey, etc)   │
│  - name: TEXT UNIQUE (canonical name like "gin")              │
│  - type: TEXT (clear, aged, spiced, etc)                       │
└─────────────────────────────────────────────────────────────────┘
          ▲                          ▲
          │ (1:N)                    │ (1:N)
          │ FK: spirit_id            │
          │                          │
┌──────────────────────────────────┐ │
│      SPIRIT_BRANDS               │ │
├──────────────────────────────────┤ │
│ PK: id                           │ │
│ FK: spirit_id → spirits.id       │ │
│ brand_name: TEXT UNIQUE          │ │
│ abv: REAL (41.2, 47, etc)        │ │
│ origin: TEXT                     │ │
└──────────────────────────────────┘ │
                                     │
                                     ├─ "gin"
                                     │  ├─ Plymouth Gin
                                     │  ├─ Beefeater Gin
                                     │  ├─ Tanqueray Gin
                                     │  └─ Ford's Gin
                                     │
                                     └─ "rum"
                                        ├─ El Dorado 3yr
                                        ├─ Plantation 5yr
                                        └─ Diplomatico

┌─────────────────────────────────────────────────────────────────┐
│                        LIQUEURS                                 │
├─────────────────────────────────────────────────────────────────┤
│ PK: id                                                          │
│ name: TEXT UNIQUE (elderflower liqueur, triple sec, etc)      │
│ category: TEXT (floral, fruit, herbal, cream)                 │
│ abv_min: REAL                                                  │
│ abv_max: REAL                                                  │
└─────────────────────────────────────────────────────────────────┘
          ▲
          │ (1:N) FK: liqueur_id
          │
     ┌────────────────────────────────┐
     │    LIQUEUR_BRANDS              │
     ├────────────────────────────────┤
     │ id, liqueur_id, brand_name     │
     │ abv, origin, notes             │
     └────────────────────────────────┘
          ├─ St. Germain → elderflower liqueur
          ├─ Luxardo Maraschino → maraschino liqueur
          └─ Chartreuse → herbal liqueur

┌─────────────────────────────────────────────────────────────────┐
│                         WINES                                   │
├─────────────────────────────────────────────────────────────────┤
│ PK: id                                                          │
│ name: TEXT UNIQUE (dry vermouth, sherry, champagne)            │
│ category: TEXT (fortified, sparkling, still)                   │
│ style: TEXT (dry, sweet, brut)                                 │
└─────────────────────────────────────────────────────────────────┘
          ▲
          │ (1:N) FK: wine_id
          │
     ┌────────────────────────────────┐
     │    WINE_BRANDS                 │
     ├────────────────────────────────┤
     │ id, wine_id, brand_name        │
     │ vintage_year, abv, origin      │
     └────────────────────────────────┘
          ├─ Dolin Blanc Vermouth → dry vermouth
          └─ La Gitana Fino Sherry → sherry


┌────────────────────────────────────────────────────────────────────┐
│           COCKTAILS (Main Recipe Table)                            │
├────────────────────────────────────────────────────────────────────┤
│ PK: id                                                             │
│ name: TEXT UNIQUE (Celine Fizz, Alligator Tears, etc)             │
│ description: TEXT                                                  │
│ glass_type: TEXT (coupe, rocks, highball)                         │
│ temperature: TEXT (up, on ice, hot)                               │
│ source: TEXT (Denver 2025, Dry Jan. 2025)                         │
│ is_batch: BOOLEAN (true = makes multiple servings)                │
│ created_at: TIMESTAMP                                             │
│ updated_at: TIMESTAMP                                             │
└────────────────────────────────────────────────────────────────────┘
          │
          │ (1:N) FK: cocktail_id
          │
    ┌─────────────────────────────────────────────────┐
    │ COCKTAIL_INGREDIENTS                            │
    ├─────────────────────────────────────────────────┤
    │ PK: id, FK: cocktail_id                         │
    │ ingredient_type: TEXT                           │
    │  - 'spirit_brand'                               │
    │  - 'liqueur_brand'                              │
    │  - 'wine_brand'                                 │
    │  - 'prep_ref'                                   │
    │  - 'batch_ref' (reference to another cocktail)  │
    │  - 'other'                                      │
    │ ingredient_id: INTEGER                          │
    │ quantity: REAL (volume in oz, weight in grams)  │
    │ unit: TEXT ('oz' for volume, 'g' for weight)    │
    │ notes: TEXT (for 'other' type, canonical name)  │
    │ sort_order: INTEGER                             │
    └─────────────────────────────────────────────────┘
          │
          └─ References based on ingredient_type:
             spirit_brands, liqueur_brands, wine_brands,
             beverage_preps, cocktails (batch_ref), or 'other'


┌────────────────────────────────────────────────────────────────────┐
│      BEVERAGE_PREP_TYPES (Prep Categories)                        │
├────────────────────────────────────────────────────────────────────┤
│ PK: id                                                             │
│ type_name: TEXT UNIQUE (syrup, infusion, garnish, mix)           │
│ description: TEXT                                                  │
└────────────────────────────────────────────────────────────────────┘
          ▲
          │ (1:N) FK: type_id
          │
┌────────────────────────────────────────────────────────────────────┐
│         BEVERAGE_PREPS (Reusable Recipe Templates)                │
├────────────────────────────────────────────────────────────────────┤
│ PK: id                                                             │
│ name: TEXT UNIQUE (Syrup: Simple, Garnish: Coconut Marshmallow)  │
│ canonical_name: TEXT (internal name for fuzzy matching)           │
│ type_id: INTEGER FK → beverage_prep_types                         │
│ description: TEXT                                                  │
│ instructions: TEXT (JSON array or string)                         │
│ yield: TEXT (1 quart, 500ml, etc)                                 │
│ shelf_life: TEXT (2 weeks refrigerated)                           │
│ active_time: TEXT (30 minutes)                                    │
│ total_time: TEXT (1 hour)                                         │
│ created_at: TIMESTAMP                                             │
│ updated_at: TIMESTAMP                                             │
└────────────────────────────────────────────────────────────────────┘
          │                          │
          │ (1:N)                    │ (1:N) Self-referential
          │ FK: prep_id              │      (preps using other preps)
          │                          │
    ┌──────────────────────────────┐│
    │   PREP_INGREDIENTS           ││
    ├──────────────────────────────┤│
    │ id, prep_id                  ││
    │ ingredient_type: TEXT        ││
    │  - 'spirit_brand'            ││
    │  - 'liqueur_brand'           ││
    │  - 'wine_brand'              ││
    │  - 'prep_ref' ───────────────┘│
    │  - 'other'                    │
    │ ingredient_id: INTEGER        │
    │ quantity: REAL (oz or grams)  │
    │ unit: TEXT ('oz' or 'g')      │
    │ sort_order: INTEGER           │
    └──────────────────────────────┘
          │
          └─ References:
             spirit_brands, liqueur_brands, wine_brands,
             OR self (prep_ref)


┌────────────────────────────────────────────────────────────────────┐
│       OTHER_INGREDIENTS (Structured Other Ingredients)            │
├────────────────────────────────────────────────────────────────────┤
│ PK: id                                                             │
│ canonical_name: TEXT UNIQUE (internal canonical form)             │
│ display_name: TEXT (how user sees it: "fresh lemon juice")       │
│ category: TEXT (juice, egg, garnish, produce, etc)               │
│ unit: TEXT ('oz' for liquids, 'g' for solids, 'count' for items) │
│ created_at: TIMESTAMP                                             │
│ updated_at: TIMESTAMP                                             │
└────────────────────────────────────────────────────────────────────┘


┌────────────────────────────────────────────────────────────────────┐
│    INGREDIENT_ALIASES (Fuzzy Matching & Admin Merging)            │
├────────────────────────────────────────────────────────────────────┤
│ PK: id                                                             │
│ ingredient_type: TEXT (other, prep, spirit_brand, etc)            │
│ fuzzy_name: TEXT (user-entered variant: "fresh lemon juice")     │
│ ingredient_id: INTEGER FK (canonical ingredient id)               │
│ match_score: REAL (0.0-1.0, similarity score)                     │
│ status: TEXT ('pending' | 'approved' | 'rejected')                │
│ reviewed_by: INTEGER FK → users (admin who approved)              │
│ reviewed_at: TIMESTAMP                                            │
│ created_at: TIMESTAMP                                             │
│ updated_at: TIMESTAMP                                             │
└────────────────────────────────────────────────────────────────────┘


┌────────────────────────────────────────────────────────────────────┐
│           ERROR_LOG (Circular Dependency & Parse Errors)          │
├────────────────────────────────────────────────────────────────────┤
│ PK: id                                                             │
│ error_type: TEXT (circular_prep_dependency, circular_batch_ref,   │
│                   parse_error, unit_conversion_error, etc)        │
│ severity: TEXT ('warning' | 'error' | 'critical')                 │
│ related_entity_type: TEXT (prep, cocktail, ingredient, etc)       │
│ related_entity_id: INTEGER (id of affected entity)                │
│ message: TEXT (detailed error description)                        │
│ context: TEXT (JSON with additional context)                      │
│ resolved: BOOLEAN (default false, admin marks when fixed)         │
│ resolved_at: TIMESTAMP                                            │
│ created_at: TIMESTAMP                                             │
└────────────────────────────────────────────────────────────────────┘
```

---

## Key Design Decisions

### 1. Unit Standardization

**All quantities stored in two units:**

- **oz (fluid ounces)** for volume measurements
  - 1 oz = 1 jigger (standard cocktail measurement)
  - Covers: liquor, syrups, juices, water, etc.
  - Store as REAL number (e.g., 0.5, 1.5, 2.0)

- **g (grams)** for weight measurements
  - Covers: sugar, fruit, herbs, egg whites, garnishes, spices
  - Store as REAL number (e.g., 15.5, 225.0)

**Conversion at data entry:**
```
User enters: "250 ml simple syrup"
Store as: 250 ml → 8.45 oz (volume)

User enters: "1 lb fresh strawberries"
Store as: 1 lb → 453.6 g (weight)

User enters: "1 egg white"
Store as: "other" ingredient, unit='count'
```

**Why this matters:**
- Consistency for inventory matching
- Easy conversion between measurement systems
- Supports international recipes
- Clear what is weight vs. volume

### 2. Ingredient Parsing & Fuzzy Matching with Admin Control

**User enters ingredient:** "0.75 oz fresh lemon juice"

**System performs:**
1. Parse quantity and unit: `0.75 oz`
2. Extract ingredient name: "fresh lemon juice"
3. Fuzzy match against canonical ingredients
   - "lemon juice" (canonical) matches at 92%
   - "lime juice" matches at 45% (below threshold)
4. If match score > 80%: Show to admin in "Pending Matches"
5. Admin can:
   - ✅ Approve & merge ("fresh lemon juice" → "lemon juice")
   - ❌ Reject ("fresh lemon juice" → new canonical entry)
   - ⚠️ Ignore (leave as-is for later review)

**INGREDIENT_ALIASES table tracks:**
- User variant: "fresh lemon juice"
- Canonical: "lemon juice"
- Match score: 0.92
- Status: pending/approved/rejected
- Reviewed by: admin_user_id
- Reviewed at: timestamp

**Admin Dashboard shows:**
```
Pending Matches (88 items)
├─ "fresh lemon juice" → "lemon juice" (92% match) [APPROVE] [REJECT]
├─ "lime juice, fresh" → "lime juice" (87% match) [APPROVE] [REJECT]
├─ "simple syrup" → "Syrup: Simple" (95% match) [APPROVE] [REJECT]
└─ "egg white" → "egg whites" (80% match) [APPROVE] [REJECT]
```

### 3. Circular Dependency Detection & Error Logging

**Prevention via database constraints:**

```sql
-- Prevent circular prep dependencies
CREATE OR REPLACE FUNCTION prevent_circular_preps()
RETURNS TRIGGER AS $$
DECLARE
  v_cycle_detected BOOLEAN;
BEGIN
  WITH RECURSIVE prep_chain AS (
    SELECT ingredient_id
    FROM prep_ingredients
    WHERE prep_id = NEW.ingredient_id AND ingredient_type = 'prep_ref'
    
    UNION ALL
    
    SELECT pi.ingredient_id
    FROM prep_ingredients pi
    JOIN prep_chain pc ON pi.prep_id = pc.ingredient_id
    WHERE pi.ingredient_type = 'prep_ref' AND depth < 10  -- limit recursion
  )
  SELECT EXISTS (SELECT 1 FROM prep_chain WHERE ingredient_id = NEW.prep_id)
  INTO v_cycle_detected;
  
  IF v_cycle_detected THEN
    INSERT INTO error_log (error_type, severity, related_entity_type, related_entity_id, message, context)
    VALUES (
      'circular_prep_dependency',
      'error',
      'prep',
      NEW.prep_id,
      'Circular dependency detected in prep references',
      jsonb_build_object('source_prep', NEW.prep_id, 'target_prep', NEW.ingredient_id, 'timestamp', NOW())
    );
    
    RAISE EXCEPTION 'Circular prep dependency would be created';
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_circular_preps
BEFORE INSERT OR UPDATE ON prep_ingredients
FOR EACH ROW
EXECUTE FUNCTION prevent_circular_preps();
```

**Similarly for batch_ref (cocktail referencing itself):**

```sql
CREATE OR REPLACE FUNCTION prevent_batch_self_reference()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.ingredient_type = 'batch_ref' AND NEW.ingredient_id = NEW.cocktail_id THEN
    INSERT INTO error_log (error_type, severity, related_entity_type, related_entity_id, message)
    VALUES ('circular_batch_ref', 'error', 'cocktail', NEW.cocktail_id, 'Cocktail cannot reference itself as batch');
    
    RAISE EXCEPTION 'Cocktail cannot reference itself as a batch';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_batch_self_reference
BEFORE INSERT ON cocktail_ingredients
FOR EACH ROW
EXECUTE FUNCTION prevent_batch_self_reference();
```

**Error Log Usage:**
- All constraint violations logged with context
- Admin dashboard shows errors with "resolve" button
- Errors prevent data insertion but don't crash system
- Historical tracking for debugging

### 4. is_batch as Display Flag, Not Database Separation

**is_batch = true means:**
- Recipe yields more than one standard drink
- User sees a visual indicator: "🍹 Makes 4 servings"
- Can be used as ingredient in other cocktails via `batch_ref`
- Same table, same queries, just a flag

**Example: "Strawberry Banana Creme Fraiche"**
```sql
INSERT INTO cocktails (name, is_batch, glass_type)
VALUES ('Strawberry Banana Creme Fraiche', true, NULL);

INSERT INTO cocktail_ingredients (cocktail_id, ingredient_type, ...)
VALUES 
  (123, 'other', 'strawberries', 1, 'lb', 1),
  (123, 'other', 'bananas', 2, 'count', 2),
  (123, 'other', 'heavy cream', 1, 'cup', 3);

-- Later, when used in "Alligator Tears":
INSERT INTO cocktail_ingredients (cocktail_id, ingredient_type, ingredient_id, quantity, unit)
VALUES (alligator_tears_id, 'batch_ref', 123, 0.75, 'oz');
```

### 5. OTHER_INGREDIENTS Table for Better Organization

**Instead of storing all "other" ingredients in notes field:**

```sql
-- Better: Structured OTHER_INGREDIENTS table
INSERT INTO other_ingredients (canonical_name, display_name, category, unit)
VALUES 
  ('lemon_juice', 'fresh lemon juice', 'juice', 'oz'),
  ('lime_juice', 'fresh lime juice', 'juice', 'oz'),
  ('egg_white', 'egg white', 'egg', 'count'),
  ('fresh_strawberries', 'fresh strawberries', 'produce', 'g');

-- Then reference it:
INSERT INTO cocktail_ingredients 
  (cocktail_id, ingredient_type, ingredient_id, quantity, unit, notes)
VALUES 
  (123, 'other', 5, 0.75, 'oz', NULL)  -- references other_ingredients.id=5
```

**Benefits:**
- Searchable
- Sortable
- Allows fuzzy matching to canonical names
- Tracks variants via INGREDIENT_ALIASES
- Better data quality

---

## Query Examples

### Get all recipes using a specific spirit brand
```sql
SELECT DISTINCT c.id, c.name, c.is_batch
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
WHERE ci.ingredient_type = 'spirit_brand' AND ci.ingredient_id = 101
```

### Get all recipes using any gin (generic spirit query)
```sql
SELECT DISTINCT c.id, c.name, c.is_batch
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
JOIN spirit_brands sb ON ci.ingredient_id = sb.id
JOIN spirits s ON sb.spirit_id = s.id
WHERE ci.ingredient_type = 'spirit_brand' AND s.name = 'gin'
ORDER BY c.name
```

### Get all recipes using a specific prep
```sql
SELECT DISTINCT c.id, c.name, ci.quantity, ci.unit, c.is_batch
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
WHERE ci.ingredient_type = 'prep_ref' AND ci.ingredient_id = 456
```

### Get all ingredients for a cocktail (with unit display)
```sql
SELECT 
  CASE 
    WHEN ci.ingredient_type = 'spirit_brand' THEN (
      SELECT CONCAT(sb.brand_name, ' (', s.name, ')')
      FROM spirit_brands sb
      JOIN spirits s ON sb.spirit_id = s.id
      WHERE sb.id = ci.ingredient_id
    )
    WHEN ci.ingredient_type = 'liqueur_brand' THEN (
      SELECT CONCAT(lb.brand_name, ' (', l.name, ')')
      FROM liqueur_brands lb
      JOIN liqueurs l ON lb.liqueur_id = l.id
      WHERE lb.id = ci.ingredient_id
    )
    WHEN ci.ingredient_type = 'wine_brand' THEN (
      SELECT CONCAT(wb.brand_name, ' (', w.name, ')')
      FROM wine_brands wb
      JOIN wines w ON wb.wine_id = w.id
      WHERE wb.id = ci.ingredient_id
    )
    WHEN ci.ingredient_type = 'prep_ref' THEN (
      SELECT name FROM beverage_preps WHERE id = ci.ingredient_id
    )
    WHEN ci.ingredient_type = 'batch_ref' THEN (
      SELECT CONCAT(name, ' (batch)')
      FROM cocktails WHERE id = ci.ingredient_id
    )
    WHEN ci.ingredient_type = 'other' THEN (
      SELECT display_name FROM other_ingredients WHERE id = ci.ingredient_id
    )
  END as ingredient_name,
  ci.quantity,
  CASE ci.unit WHEN 'oz' THEN 'fl oz' WHEN 'g' THEN 'g' WHEN 'count' THEN '' ELSE ci.unit END as unit,
  ci.ingredient_type
FROM cocktail_ingredients ci
WHERE ci.cocktail_id = ?
ORDER BY ci.sort_order
```

### Find pending fuzzy matches for admin review
```sql
SELECT 
  ia.fuzzy_name,
  ia.ingredient_type,
  CASE 
    WHEN ia.ingredient_type = 'other' THEN (SELECT display_name FROM other_ingredients WHERE id = ia.ingredient_id)
    WHEN ia.ingredient_type = 'prep' THEN (SELECT name FROM beverage_preps WHERE id = ia.ingredient_id)
  END as canonical_name,
  ia.match_score,
  COUNT(*) as times_used
FROM ingredient_aliases ia
WHERE ia.status = 'pending'
GROUP BY ia.ingredient_type, ia.ingredient_id, ia.fuzzy_name
ORDER BY ia.match_score DESC
```

### View error log
```sql
SELECT 
  error_type,
  severity,
  related_entity_type,
  related_entity_id,
  message,
  resolved,
  created_at
FROM error_log
WHERE resolved = false
ORDER BY severity DESC, created_at DESC
```

---

## Insertion Flow

### New Cocktail: "Celine Fizz"

```
Parse HTML:
  - name: "Celine Fizz"
  - glass: "coupe"
  - is_batch: false
  - ingredients: [
      "2 oz Plymouth Gin",
      "0.5 oz St. Germain",
      "0.75 oz fresh lemon juice",
      "0.5 oz Syrup: Simple",
      "2 oz Champagne"
    ]

Processing:
  1. "2 oz Plymouth Gin" 
     ✓ type: spirit_brand, brand: Plymouth Gin, qty: 2, unit: oz
  
  2. "0.5 oz St. Germain"
     ✓ type: liqueur_brand, brand: St. Germain, qty: 0.5, unit: oz
  
  3. "0.75 oz fresh lemon juice"
     → Fuzzy match "fresh lemon juice" against other_ingredients
     → Found match "lemon juice" at 92%
     → Add to INGREDIENT_ALIASES as pending
     → Store: type: other, ingredient_id: (lemon_juice), qty: 0.75, unit: oz
  
  4. "0.5 oz Syrup: Simple"
     ✓ type: prep_ref, prep: Syrup: Simple, qty: 0.5, unit: oz
  
  5. "2 oz Champagne"
     ✓ type: wine_brand, wine: Champagne, qty: 2, unit: oz

Database Insertion:
  INSERT INTO cocktails (name, glass_type, is_batch)
  VALUES ('Celine Fizz', 'coupe', false)
  → id = 123

  INSERT INTO cocktail_ingredients VALUES
    (cocktail_id=123, ingredient_type='spirit_brand', ingredient_id=101, qty=2, unit='oz', sort=0)
    (cocktail_id=123, ingredient_type='liqueur_brand', ingredient_id=202, qty=0.5, unit='oz', sort=1)
    (cocktail_id=123, ingredient_type='other', ingredient_id=5, qty=0.75, unit='oz', sort=2)
    (cocktail_id=123, ingredient_type='prep_ref', ingredient_id=456, qty=0.5, unit='oz', sort=3)
    (cocktail_id=123, ingredient_type='wine_brand', ingredient_id=303, qty=2, unit='oz', sort=4)

  INSERT INTO ingredient_aliases
  VALUES (ingredient_type='other', fuzzy_name='fresh lemon juice', ingredient_id=5, match_score=0.92, status='pending')
```

### New Batch Cocktail: "Strawberry Banana Creme Fraiche"

```
Insert batch cocktail:
  INSERT INTO cocktails (name, is_batch)
  VALUES ('Strawberry Banana Creme Fraiche', true)
  → id = 500

Insert base ingredients (convert units):
  - "1 lb fresh strawberries" → 453.6 g
  - "2 bananas" → stored as 'count'
  - "1 cup heavy cream" → 236.588 ml ≈ 8 oz
  - "0.25 oz vanilla extract" → 0.25 oz

  INSERT INTO cocktail_ingredients VALUES
    (cocktail_id=500, ingredient_type='other', ingredient_id=101, qty=453.6, unit='g', notes='strawberries', sort=0)
    (cocktail_id=500, ingredient_type='other', ingredient_id=102, qty=2, unit='count', notes='bananas', sort=1)
    (cocktail_id=500, ingredient_type='other', ingredient_id=103, qty=8, unit='oz', notes='heavy cream', sort=2)
    (cocktail_id=500, ingredient_type='other', ingredient_id=104, qty=0.25, unit='oz', notes='vanilla extract', sort=3)

When used in another cocktail ("Alligator Tears"):
  INSERT INTO cocktail_ingredients (cocktail_id, ingredient_type, ingredient_id, qty, unit)
  VALUES (alligator_tears_id, 'batch_ref', 500, 0.75, 'oz')
```

---

## Performance Considerations

**Indexes:**
- `cocktails.name` (frequently searched)
- `cocktails.is_batch` (filter batch recipes)
- `cocktail_ingredients.cocktail_id` (list ingredients for recipe)
- `cocktail_ingredients.ingredient_type` (filter by type)
- `cocktail_ingredients.ingredient_id` (find recipes using ingredient)
- `beverage_preps.name` (fuzzy matching/lookups)
- `other_ingredients.canonical_name` (fuzzy matching)
- `ingredient_aliases.fuzzy_name` (find pending matches)
- Composite: `(ingredient_type, ingredient_id)` on cocktail_ingredients
- Composite: `(ingredient_type, ingredient_id, status)` on ingredient_aliases

**Views (Optional):**
```sql
-- Pending fuzzy matches for admin dashboard
CREATE VIEW admin_pending_matches AS
SELECT 
  ia.fuzzy_name,
  ia.ingredient_type,
  ia.ingredient_id,
  ia.match_score,
  COUNT(*) as times_used
FROM ingredient_aliases ia
WHERE ia.status = 'pending'
GROUP BY ia.ingredient_type, ia.ingredient_id
ORDER BY ia.match_score DESC;

-- Active errors for monitoring
CREATE VIEW active_errors AS
SELECT *
FROM error_log
WHERE resolved = false
ORDER BY severity DESC, created_at DESC;
```

---

## Summary

✅ **Unit standardization:** oz for volume, grams for weight  
✅ **Fuzzy matching with admin control:** Variants merged or tracked  
✅ **Circular dependency prevention:** Database constraints + error logging  
✅ **is_batch as display flag:** No separate table needed  
✅ **OTHER_INGREDIENTS table:** Better organization of miscellaneous items  
✅ **INGREDIENT_ALIASES table:** Tracks fuzzy matches and variants  
✅ **ERROR_LOG table:** Centralized error tracking with context  
✅ **Single source of truth:** All data in cocktail_ingredients  
✅ **Proper normalization:** No data duplication  
✅ **Scalable patterns:** Consistent query logic across all ingredient types  
