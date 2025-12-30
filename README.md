# cocktail-database-parser

Cocktail database architecture with inventory-based recipe finder system

## Documentation

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** - Complete database schema, entity relationships, query patterns, and insertion flows
- **[IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)** - Code examples for fuzzy matching, unit conversion, error logging, and circular dependency detection
- **[UPDATES.md](./UPDATES.md)** - Latest requirements: quantity rounding, dual spirit/liqueur/wine types, recipe scaling UI, adjustable ingredients, cocktail steps, version control

## Quick Overview

### Core Model

- **PREPS**: Reusable recipe templates (Simple Syrup, Matcha Infusion, etc.) used by many cocktails
- **BATCHES**: Cocktails marked with `is_batch = true` that yield more than one standard drink (display flag only)
- **BRANDS**: Spirit, liqueur, and wine brands that affect taste are stored separately for accuracy
- **COCKTAILS**: Main recipe table with ingredients tracked in a single `cocktail_ingredients` table

### Key Design Principles

✅ **Quantity Precision** - DECIMAL(10,2) for all measurements, always rounded to 2 decimal places
✅ **Dual Types** - Query by both spirit type (bourbon) AND brand (Buffalo Trace bourbon)
✅ **Unit Standardization** - oz for volume, grams for weight
✅ **Fuzzy Matching** - Admin-controlled merging of ingredient variants
✅ **Error Tracking** - Circular dependencies and parse errors logged centrally
✅ **Recipe Scaling** - ½x, 1x, 2x, 3x buttons with base_yield support
✅ **Adjustable Ingredients** - Min/max ranges and "to taste" support
✅ **Preparation Steps** - Full method documentation with techniques
✅ **Version Control** - Complete recipe history with superadmin tracking
✅ **Flexible Ingredients** - Supports spirit_brand, liqueur_brand, wine_brand, prep_ref, batch_ref, and other types
✅ **Reusable Preps** - One prep (Syrup: Simple) can be used by many cocktails (1:N relationship)
✅ **Normalized Schema** - No data duplication, relationships clearly defined

## Key Features

### 1. Quantity Precision
- All quantities stored as DECIMAL(10,2)
- Always rounded to exactly 2 decimal places
- Prevents floating-point errors (0.75 oz stays 0.75, not 0.749999...)
- Consistent for inventory calculations

### 2. Ingredient Parsing & Fuzzy Matching
- Automatically extracts quantities and units from ingredient strings
- Fuzzy matches against canonical ingredients at 80% threshold
- Admin dashboard reviews and approves/rejects/ignores matches
- Variants tracked in INGREDIENT_ALIASES table for data quality

### 3. Unit Conversion
- **Volume** (oz) - For liquids: spirits, juices, syrups, creams
- **Weight** (grams) - For solids: sugar, fruit, herbs, garnishes
- Automatic conversion from ml, cups, tbsp, lbs, etc. to standard units
- Ingredient-specific unit validation

### 4. Recipe Scaling
- Four scaling buttons: ½x, 1x (default), 2x, 3x
- base_yield field for batch recipes
- All quantities automatically scaled
- Perfect for batch recipes (multiply servings)

### 5. Adjustable Ingredients
- Support for min/max ranges (0.50-1.00 oz)
- "To taste" ingredients with no specific quantity
- Realistic recipe reflection
- Inventory flexibility

### 6. Cocktail Preparation Steps
- Full method documentation with techniques
- Step-by-step instructions (build, shake, stir, strain, float, muddle, etc.)
- Complete context for each recipe
- Flamenco Tears example with detailed steps

### 7. Version Control
- Superadmin can update recipes with tracking
- Complete version history (who changed what and when)
- Ability to revert to previous versions
- Audit trail for accountability

### 8. Error Handling
- **Circular Dependency Detection** - Prevents preps from creating infinite loops
- **Batch Self-Reference Prevention** - Cocktails can't reference themselves
- **Error Logging** - All errors tracked in ERROR_LOG with context for debugging
- **Admin Resolution** - Errors marked as resolved when fixed

### 9. Admin Tools
- Pending fuzzy matches dashboard
- Error log with severity levels
- Ingredient variant management
- Batch tracking and usage
- Recipe version history

## Getting Started

See documentation files for:
- Complete database schema in [ARCHITECTURE.md](./ARCHITECTURE.md)
- Python implementation examples in [IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)
- All new requirements in [UPDATES.md](./UPDATES.md)
- Query patterns for common operations
- Insertion flows for new cocktails and batches
- Admin workflow for ingredient management

## Latest Updates (December 30, 2025)

1. ✅ **Quantity Rounding** - DECIMAL(10,2) for all measurements
2. ✅ **Dual Types** - Query by both type and brand
3. ✅ **Recipe Scaling** - ½x, 1x, 2x, 3x buttons
4. ✅ **Adjustable Ingredients** - Min/max ranges and "to taste"
5. ✅ **Cocktail Steps** - Full method with techniques
6. ✅ **Version Control** - Superadmin recipe tracking
7. ✅ **Field Rename** - ingredient_notes for clarity
8. 💬 **Seasonality** - Future enhancement

See [UPDATES.md](./UPDATES.md) for complete details.
