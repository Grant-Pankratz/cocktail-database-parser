# Cocktail Database Architecture - Visual Guide

## Core Concept

**PREPS** are reusable recipe templates (Syrup: Simple, Matcha Infusion, etc.) that are used by many cocktails. A prep is made once and referenced by multiple recipes—it's not tied to a specific cocktail.

**BATCHES** are cocktails that are made in bulk and then portioned out with additional ingredients added before serving. This is indicated by an `is_batch` flag on the cocktails table.

**BRANDS** critically affect taste. Spirit brands have different taste profiles (Plymouth Gin vs Tanqueray), liqueur brands vary dramatically (St. Germain vs other elderflower liqueurs are VERY different), and wine brands matter for consistency.

---

## Entity Relationship Diagram (Text Format)

```
┌─────────────────────────────────────────────────────────────────┐
│                          SPIRITS                                │
├─────────────────────────────────────────────────────────────────┤
│ PK: id (INTEGER)                                                │
│ Fields:                                                         │
│  - category: TEXT  (gin, rum, vodka, tequila, whiskey, etc)   │
│  - name: TEXT UNIQUE (canonical name like "gin")              │
│  - type: TEXT (clear, aged, spiced, etc)                       │
└─────────────────────────────────────────────────────────────────┘
          ▲
          │ (1:N) FK: spirit_id
          │
┌──────────────────────────────────┐
│      SPIRIT_BRANDS               │
├──────────────────────────────────┤
│ PK: id                           │
│ FK: spirit_id → spirits.id       │
│ brand_name: TEXT UNIQUE          │
│ abv: REAL (41.2, 47, etc)        │
│ origin: TEXT                     │
└──────────────────────────────────┘
     ├─ Plymouth Gin (spirit: gin)
     ├─ Beefeater Gin (spirit: gin)
     ├─ Tanqueray Gin (spirit: gin)
     ├─ El Dorado 3yr (spirit: rum)
     └─ Plantation 5yr (spirit: rum)

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
     │ PK: id                         │
     │ FK: liqueur_id → liqueurs.id   │
     │ brand_name: TEXT UNIQUE        │
     │ abv, origin, notes             │
     └────────────────────────────────┘
          ├─ St. Germain (liqueur: elderflower)
          ├─ Luxardo Maraschino (liqueur: maraschino)
          └─ Chartreuse (liqueur: herbal)

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
     │ PK: id                         │
     │ FK: wine_id → wines.id         │
     │ brand_name: TEXT UNIQUE        │
     │ vintage_year, abv, origin      │
     └────────────────────────────────┘
          ├─ Dolin Blanc Vermouth (wine: dry vermouth)
          └─ La Gitana Fino Sherry (wine: sherry)


┌────────────────────────────────────────────────────────────────────┐
│           COCKTAILS (Main Recipe Table)                            │
├────────────────────────────────────────────────────────────────────┤
│ PK: id                                                             │
│ name: TEXT UNIQUE (Celine Fizz, Alligator Tears, etc)             │
│ description: TEXT                                                  │
│ glass_type: TEXT (coupe, rocks, highball)                         │
│ temperature: TEXT (up, on ice, hot)                               │
│ source: TEXT (Denver 2025, Dry Jan. 2025)                         │
│ is_batch: BOOLEAN (true = made in bulk, false = standard recipe)  │
│ created_at: TIMESTAMP                                             │
│ updated_at: TIMESTAMP                                             │
└────────────────────────────────────────────────────────────────────┘
          │
          │ (1:N) FK: cocktail_id
          │
    ┌─────────────────────────┐
    │ COCKTAIL_INGREDIENTS    │
    ├─────────────────────────┤
    │ PK: id                  │
    │ FK: cocktail_id         │
    │ ingredient_type: TEXT   │
    │  - 'spirit_brand'       │
    │  - 'liqueur_brand'      │
    │  - 'wine_brand'         │
    │  - 'prep_ref'           │
    │  - 'batch_ref'          │
    │  - 'other'              │
    │ ingredient_id: INTEGER  │
    │ quantity: REAL          │
    │ unit: TEXT (oz, ml)     │
    │ notes: TEXT (for 'other'│
    │ sort_order: INTEGER     │
    └─────────────────────────┘
          │
          ▼ (references based on type)
    ┌──────────────┬──────────────┬──────────────┬─────────────┐
    │              │              │              │             │
    ▼              ▼              ▼              ▼             ▼
spirit_brands liqueur_brands wine_brands beverage_preps cocktails


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
    │  - 'prep_ref'────────────────┘│
    │  - 'other'                    │
    │ ingredient_id: INTEGER        │
    │ quantity: REAL                │
    │ unit: TEXT (oz, ml, tsp)      │
    │ sort_order: INTEGER           │
    └──────────────────────────────┘
          │
          └─ References:
             spirit_brands, liqueur_brands, wines_brands,
             OR self via prep_ref for nested preps
```

---

## Key Design Decisions

### 1. Single Source of Truth: COCKTAIL_INGREDIENTS

All ingredient data for a recipe is stored in `cocktail_ingredients`. This table is the source of truth.

- **For display:** Query `cocktail_ingredients` and join to the appropriate ingredient table based on `ingredient_type`
- **For prep dependencies:** Filter by `ingredient_type = 'prep_ref'`
- **For batch dependencies:** Filter by `ingredient_type = 'batch_ref'`

This eliminates data duplication and keeps queries consistent.

### 2. Branded → Generic Mapping for Queries

Spirits, liqueurs, and wines are stored with both generic and branded versions:

```
Query: "All recipes using gin"
Flow:
  cocktail_ingredients.ingredient_type = 'spirit_brand'
    → spirit_brands.spirit_id
      → spirits.id
        → spirits.name = 'gin'

Result: Both "Plymouth Gin" and "Beefeater Gin" recipes
```

This allows:
- ✅ Brand-specific queries ("recipes using Plymouth Gin")
- ✅ Generic queries ("recipes using any gin")
- ✅ Accurate taste-critical data (brands DO matter)

### 3. Preps as Reusable Templates

A prep is used by MANY cocktails but made ONCE. Examples:
- Syrup: Simple (used in 50+ cocktails)
- Matcha Infusion (used in 5 cocktails)
- Garnish: Coconut Marshmallow (used in 3 cocktails)

One prep → many cocktails (1:N relationship via `cocktail_ingredients.ingredient_type = 'prep_ref'`)

### 4. Batches as a Flag, Not a Table

Batches are cocktails marked with `is_batch = true`. When building one:

```
1. Create the batch cocktail recipe:
   INSERT INTO cocktails (name, is_batch)
   VALUES ('Strawberry Banana Creme Fraiche', true)
   → id = 5

2. Add its base ingredients to cocktail_ingredients
   (e.g., fresh strawberries, banana, cream, etc.)

3. When used in another cocktail:
   INSERT INTO cocktail_ingredients
   VALUES (..., ingredient_type: 'batch_ref', ingredient_id: 5, quantity: 0.75, unit: 'oz')
```

Why a flag instead of a separate table?
- ✅ Batches ARE cocktails (they have recipes, serve counts, etc.)
- ✅ No duplicate data
- ✅ Simpler queries
- ✅ Easier to track which cocktails are batches

### 5. Brand Importance Varies by Category

- **Spirit brands:** Matters (Plymouth vs Tanqueray gin taste different), kept for flexibility
- **Liqueur brands:** CRITICAL (St. Germain vs other elderflower liqueurs are drastically different)
- **Wine brands:** Useful for consistency, kept for completeness

---

## Query Examples

### Get all recipes using a specific spirit brand
```sql
SELECT DISTINCT c.id, c.name
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
WHERE ci.ingredient_type = 'spirit_brand' AND ci.ingredient_id = 101 -- Plymouth Gin
```

### Get all recipes using any gin (generic spirit query)
```sql
SELECT DISTINCT c.id, c.name
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
JOIN spirit_brands sb ON ci.ingredient_id = sb.id
JOIN spirits s ON sb.spirit_id = s.id
WHERE ci.ingredient_type = 'spirit_brand' AND s.name = 'gin'
```

### Get all recipes using a specific prep
```sql
SELECT DISTINCT c.id, c.name, ci.quantity, ci.unit
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
WHERE ci.ingredient_type = 'prep_ref' AND ci.ingredient_id = 456 -- Syrup: Simple
```

### Get all ingredients for a cocktail (display recipe)
```sql
SELECT 
  ci.quantity, ci.unit,
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
    WHEN ci.ingredient_type = 'other' THEN ci.notes
  END as ingredient_name,
  ci.ingredient_type
FROM cocktail_ingredients ci
WHERE ci.cocktail_id = ?
ORDER BY ci.sort_order
```

### Get all prep dependencies for a cocktail
```sql
SELECT bp.name, bp.type_id, ci.quantity, ci.unit
FROM cocktail_ingredients ci
JOIN beverage_preps bp ON ci.ingredient_id = bp.id
WHERE ci.cocktail_id = ? AND ci.ingredient_type = 'prep_ref'
ORDER BY ci.sort_order
```

### Get all cocktails that use a specific prep
```sql
SELECT DISTINCT c.id, c.name
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
WHERE ci.ingredient_type = 'prep_ref' AND ci.ingredient_id = ?
```

### Get all batch cocktails
```sql
SELECT id, name, glass_type
FROM cocktails
WHERE is_batch = true
ORDER BY name
```

### Get recipes that use a batch cocktail
```sql
SELECT DISTINCT c.id, c.name, ci.quantity, ci.unit
FROM cocktails c
JOIN cocktail_ingredients ci ON c.id = ci.cocktail_id
WHERE ci.ingredient_type = 'batch_ref' AND ci.ingredient_id = ? -- batch cocktail id
```

### Get all nested ingredients for a prep (including sub-preps)
```sql
-- Recursive CTE to resolve all sub-ingredients
WITH RECURSIVE prep_resolution AS (
  -- Base: direct ingredients of the prep
  SELECT 
    pi.ingredient_type,
    pi.ingredient_id,
    pi.quantity,
    pi.unit,
    1 as depth
  FROM prep_ingredients pi
  WHERE pi.prep_id = ? -- starting prep id

  UNION ALL

  -- Recursive: ingredients of sub-preps
  SELECT 
    pi.ingredient_type,
    pi.ingredient_id,
    pi.quantity,
    pi.unit,
    pr.depth + 1
  FROM prep_ingredients pi
  JOIN prep_resolution pr ON pi.prep_id = pr.ingredient_id
  WHERE pr.ingredient_type = 'prep_ref' AND pr.depth < 5 -- limit nesting
)
SELECT * FROM prep_resolution
ORDER BY depth, ingredient_type
```

---

## Insertion Flow

### New Cocktail: "Celine Fizz"

```
Parse HTML:
  - name: "Celine Fizz"
  - glass: "coupe"
  - ingredients: [
      "2 oz Plymouth Gin",
      "0.5 oz St. Germain",
      "0.75 oz fresh lemon juice",
      "0.5 oz Syrup: Simple",
      "2 oz Champagne"
    ]

Classification:
  1. "2 oz Plymouth Gin" 
     → type: spirit_brand, brand: "Plymouth Gin", quantity: 2, unit: "oz"
  
  2. "0.5 oz St. Germain"
     → type: liqueur_brand, brand: "St. Germain", quantity: 0.5, unit: "oz"
  
  3. "0.75 oz fresh lemon juice"
     → type: other, notes: "lemon juice", quantity: 0.75, unit: "oz"
  
  4. "0.5 oz Syrup: Simple"
     → type: prep_ref, prep_name: "Syrup: Simple", quantity: 0.5, unit: "oz"
  
  5. "2 oz Champagne"
     → type: wine_brand, name: "Champagne", quantity: 2, unit: "oz"

Database Insertion:

  1. Insert cocktail:
     INSERT INTO cocktails (name, glass_type, is_batch)
     VALUES ('Celine Fizz', 'coupe', false)
     → id = 123

  2. Insert ingredients:
     INSERT INTO cocktail_ingredients VALUES
       (id=1, cocktail_id=123, ingredient_type='spirit_brand', ingredient_id=101, qty=2, unit='oz', sort=0)
       (id=2, cocktail_id=123, ingredient_type='liqueur_brand', ingredient_id=202, qty=0.5, unit='oz', sort=1)
       (id=3, cocktail_id=123, ingredient_type='other', ingredient_id=null, qty=0.75, unit='oz', sort=2, notes='lemon juice')
       (id=4, cocktail_id=123, ingredient_type='prep_ref', ingredient_id=456, qty=0.5, unit='oz', sort=3)
       (id=5, cocktail_id=123, ingredient_type='wine_brand', ingredient_id=303, qty=2, unit='oz', sort=4)
```

### New Batch Cocktail: "Strawberry Banana Creme Fraiche"

```
Parse:
  - name: "Strawberry Banana Creme Fraiche"
  - is_batch: true
  - base ingredients: [
      "1 lb fresh strawberries",
      "2 bananas",
      "1 cup heavy cream",
      "0.25 oz vanilla extract"
    ]

Database Insertion:

  1. Insert batch cocktail:
     INSERT INTO cocktails (name, is_batch)
     VALUES ('Strawberry Banana Creme Fraiche', true)
     → id = 500

  2. Insert base ingredients:
     INSERT INTO cocktail_ingredients VALUES
       (cocktail_id=500, ingredient_type='other', qty=1, unit='lb', notes='strawberries', sort=0)
       (cocktail_id=500, ingredient_type='other', qty=2, unit='count', notes='bananas', sort=1)
       (cocktail_id=500, ingredient_type='other', qty=1, unit='cup', notes='heavy cream', sort=2)
       (cocktail_id=500, ingredient_type='other', qty=0.25, unit='oz', notes='vanilla extract', sort=3)

  3. When used in another cocktail ("Alligator Tears"):
     INSERT INTO cocktail_ingredients (cocktail_id, ingredient_type, ingredient_id, qty, unit, sort)
     VALUES (alligator_tears_id, 'batch_ref', 500, 0.75, 'oz', 2)
```

---

## Performance Considerations

**Indexes:**
- `cocktails.name` (frequently searched)
- `cocktails.is_batch` (filter for batch recipes)
- `cocktail_ingredients.cocktail_id` (list ingredients for a recipe)
- `cocktail_ingredients.ingredient_type` (filter by type)
- `cocktail_ingredients.ingredient_id` (find recipes using an ingredient)
- `beverage_preps.name` (fuzzy matching/lookups)
- Composite: `(ingredient_type, ingredient_id)` on cocktail_ingredients

**Views (Optional):**
```sql
-- View for prep dependencies
CREATE VIEW cocktail_prep_dependencies AS
SELECT ci.cocktail_id, ci.ingredient_id as prep_id, ci.quantity, ci.unit
FROM cocktail_ingredients ci
WHERE ci.ingredient_type = 'prep_ref';

-- View for batch dependencies
CREATE VIEW cocktail_batch_dependencies AS
SELECT ci.cocktail_id, ci.ingredient_id as batch_id, ci.quantity, ci.unit
FROM cocktail_ingredients ci
WHERE ci.ingredient_type = 'batch_ref';
```

---

## Summary

✅ **Single source of truth:** All ingredient data in `cocktail_ingredients`
✅ **Brands for accuracy:** Spirit, liqueur, and wine brands preserved for taste-critical data
✅ **Preps as templates:** One prep → many cocktails (1:N relationship)
✅ **Batches as a flag:** Cocktails marked with `is_batch = true`, no separate table
✅ **Flexible ingredient types:** Supports spirit_brand, liqueur_brand, wine_brand, prep_ref, batch_ref, other
✅ **Proper normalization:** No data duplication, relationships clearly defined
✅ **Scalable query patterns:** Consistent join logic across all query types
