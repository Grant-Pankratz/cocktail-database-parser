# cocktail-database-parser

Cocktail database architecture with inventory-based recipe finder system

## Documentation

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** - Complete database schema, entity relationships, query patterns, and insertion flows
- **[IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)** - Code examples for fuzzy matching, unit conversion, error logging, and circular dependency detection

## Quick Overview

### Core Model

- **PREPS**: Reusable recipe templates (Simple Syrup, Matcha Infusion, etc.) used by many cocktails
- **BATCHES**: Cocktails marked with `is_batch = true` that yield more than one standard drink (display flag only)
- **BRANDS**: Spirit, liqueur, and wine brands that affect taste are stored separately for accuracy
- **COCKTAILS**: Main recipe table with ingredients tracked in a single `cocktail_ingredients` table

### Key Design Principles

✅ **Single Source of Truth** - All ingredient data in `cocktail_ingredients` table
✅ **Brand Accuracy** - Taste-critical brands properly differentiated
✅ **Unit Standardization** - oz for volume, grams for weight
✅ **Fuzzy Matching** - Admin-controlled merging of ingredient variants
✅ **Error Tracking** - Circular dependencies and parse errors logged centrally
✅ **Flexible Ingredients** - Supports spirit_brand, liqueur_brand, wine_brand, prep_ref, batch_ref, and other ingredient types
✅ **Reusable Preps** - One prep (Syrup: Simple) can be used by many cocktails (1:N relationship)
✅ **Normalized Schema** - No data duplication, relationships clearly defined

## Key Features

### 1. Ingredient Parsing & Fuzzy Matching

- Automatically extracts quantities and units from ingredient strings
- Fuzzy matches against canonical ingredients at 80% threshold
- Admin dashboard reviews and approves/rejects matches
- Variants tracked in INGREDIENT_ALIASES table for data quality

### 2. Unit Conversion

- **Volume** (oz) - For liquids: spirits, juices, syrups, creams
- **Weight** (grams) - For solids: sugar, fruit, herbs, garnishes
- Automatic conversion from ml, cups, tbsp, lbs, etc. to standard units
- Ingredient-specific unit validation

### 3. Error Handling

- **Circular Dependency Detection** - Prevents preps from creating infinite loops
- **Batch Self-Reference Prevention** - Cocktails can't reference themselves
- **Error Logging** - All errors tracked in ERROR_LOG with context for debugging
- **Admin Resolution** - Errors marked as resolved when fixed

### 4. Admin Tools

- Pending fuzzy matches dashboard
- Error log with severity levels
- Ingredient variant management
- Batch tracking and usage

## Getting Started

See documentation files for:
- Complete database schema in [ARCHITECTURE.md](./ARCHITECTURE.md)
- Python implementation examples in [IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)
- Query patterns for common operations
- Insertion flows for new cocktails and batches
- Admin workflow for ingredient management
