# cocktail-database-parser

Cocktail database architecture with inventory-based recipe finder system

## Documentation

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** - Complete database schema, entity relationships, query patterns, and insertion flows

## Quick Overview

### Core Model

- **PREPS**: Reusable recipe templates (Simple Syrup, Matcha Infusion, etc.) used by many cocktails
- **BATCHES**: Cocktails marked with `is_batch = true` that are made in bulk, then portioned with additional ingredients
- **BRANDS**: Spirit, liqueur, and wine brands that affect taste are stored separately for accuracy
- **COCKTAILS**: Main recipe table with ingredients tracked in a single `cocktail_ingredients` table

### Key Design Principles

✅ **Single Source of Truth** - All ingredient data in `cocktail_ingredients` table
✅ **Brand Accuracy** - Taste-critical brands (St. Germain vs other elderflower liqueurs) properly differentiated
✅ **Flexible Ingredients** - Supports spirit_brand, liqueur_brand, wine_brand, prep_ref, batch_ref, and other ingredient types
✅ **Reusable Preps** - One prep (Syrup: Simple) can be used by many cocktails (1:N relationship)
✅ **Normalized Schema** - No data duplication, relationships clearly defined

## Getting Started

See [ARCHITECTURE.md](./ARCHITECTURE.md) for:
- Complete entity relationship diagram
- Detailed field descriptions
- Query examples for common operations
- Insertion flows for new cocktails and batches
- Performance optimization tips
