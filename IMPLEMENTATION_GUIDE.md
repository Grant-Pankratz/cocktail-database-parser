# Implementation Guide

## Fuzzy Matching & Admin Merging

### Overview

When parsing ingredient strings like "0.75 oz fresh lemon juice", the system:
1. Extracts the ingredient name: "fresh lemon juice"
2. Fuzzy matches against canonical ingredients
3. If score > 80%, shows to admin for approval/rejection
4. Tracks all variants in INGREDIENT_ALIASES table

### Admin Workflow

```python
# Admin Dashboard: View Pending Matches
GET /admin/pending-matches?sort=match_score&order=desc

Response:
[
  {
    "id": 1,
    "fuzzy_name": "fresh lemon juice",
    "canonical_name": "lemon juice",
    "ingredient_type": "other",
    "match_score": 0.92,
    "times_used": 3,
    "status": "pending"
  },
  {
    "id": 2,
    "fuzzy_name": "simple syrup",
    "canonical_name": "Syrup: Simple",
    "ingredient_type": "prep",
    "match_score": 0.95,
    "times_used": 12,
    "status": "pending"
  }
]

# Admin Action: Approve & Merge
POST /admin/ingredient-aliases/{id}/approve
Body: { "approved": true }

# Backend:
# 1. Update INGREDIENT_ALIASES: status = 'approved', reviewed_by = admin_id
# 2. All cocktail_ingredients with fuzzy_name -> use canonical ingredient_id
# 3. Return confirmation

# Admin Action: Reject (Create New Canonical)
POST /admin/ingredient-aliases/{id}/reject
Body: { "create_new_canonical": true }

# Backend:
# 1. Update INGREDIENT_ALIASES: status = 'rejected'
# 2. Create new OTHER_INGREDIENTS entry: canonical_name = fuzzy_name
# 3. Update cocktail_ingredients to use new canonical
# 4. Return confirmation
```

### Python Implementation

```python
from difflib import SequenceMatcher
import logging

class IngredientMatcher:
    """Fuzzy match and track ingredient variants."""
    
    FUZZY_MATCH_THRESHOLD = 0.80  # 80% similarity
    
    def __init__(self, db_connection):
        self.db = db_connection
        self.logger = logging.getLogger(__name__)
    
    def find_ingredient_match(self, user_input: str, ingredient_type: str) -> tuple[int, float]:
        """
        Find best fuzzy match for user ingredient input.
        
        Args:
            user_input: "fresh lemon juice"
            ingredient_type: "other" | "prep" | etc
        
        Returns:
            (ingredient_id, match_score) or (None, None) if no match
        """
        # Normalize input
        normalized = user_input.lower().strip()
        
        # Get all canonical ingredients of this type
        if ingredient_type == "other":
            canonicals = self.db.query(
                "SELECT id, canonical_name, display_name FROM other_ingredients"
            )
        elif ingredient_type == "prep":
            canonicals = self.db.query(
                "SELECT id, name as canonical_name, name as display_name FROM beverage_preps"
            )
        else:
            return None, None
        
        best_match = None
        best_score = 0
        
        for row in canonicals:
            # Try both canonical and display names
            for name in [row['canonical_name'], row['display_name']]:
                ratio = SequenceMatcher(None, normalized, name.lower()).ratio()
                
                if ratio > best_score:
                    best_score = ratio
                    best_match = row['id']
        
        if best_score >= self.FUZZY_MATCH_THRESHOLD:
            return best_match, best_score
        else:
            return None, best_score
    
    def log_match_for_review(self, fuzzy_name: str, ingredient_id: int, 
                            ingredient_type: str, match_score: float) -> int:
        """
        Log a fuzzy match for admin review.
        
        Returns:
            alias_id for tracking
        """
        alias_id = self.db.insert(
            'ingredient_aliases',
            {
                'fuzzy_name': fuzzy_name,
                'ingredient_id': ingredient_id,
                'ingredient_type': ingredient_type,
                'match_score': match_score,
                'status': 'pending',
                'created_at': datetime.now()
            }
        )
        
        self.logger.info(
            f"Logged fuzzy match for review: {fuzzy_name} -> ID {ingredient_id} "
            f"(score: {match_score:.2%})"
        )
        
        return alias_id
    
    def approve_match(self, alias_id: int, admin_id: int):
        """
        Admin approves a fuzzy match. All future variants use the canonical.
        """
        alias = self.db.query_one(
            "SELECT * FROM ingredient_aliases WHERE id = %s", alias_id
        )
        
        if not alias:
            raise ValueError(f"Alias {alias_id} not found")
        
        # Mark as approved
        self.db.update(
            'ingredient_aliases',
            {'status': 'approved', 'reviewed_by': admin_id, 'reviewed_at': datetime.now()},
            {'id': alias_id}
        )
        
        # All future uses of this fuzzy_name will resolve to the canonical
        self.logger.info(
            f"Approved fuzzy match: {alias['fuzzy_name']} -> "
            f"{alias['ingredient_type']} ID {alias['ingredient_id']}"
        )
    
    def reject_match(self, alias_id: int, admin_id: int, create_new: bool = False):
        """
        Admin rejects a fuzzy match. Optionally creates new canonical entry.
        """
        alias = self.db.query_one(
            "SELECT * FROM ingredient_aliases WHERE id = %s", alias_id
        )
        
        if not alias:
            raise ValueError(f"Alias {alias_id} not found")
        
        if create_new:
            # Create new canonical ingredient
            if alias['ingredient_type'] == 'other':
                new_id = self.db.insert('other_ingredients', {
                    'canonical_name': alias['fuzzy_name'].lower().replace(' ', '_'),
                    'display_name': alias['fuzzy_name'],
                    'category': 'other',  # Will need manual categorization
                    'unit': 'oz',  # Default, may need adjustment
                    'created_at': datetime.now()
                })
                self.logger.info(
                    f"Created new canonical ingredient: {alias['fuzzy_name']} "
                    f"(ID: {new_id})"
                )
                alias_ingredient_id = new_id
            else:
                raise ValueError(f"Cannot create new canonical for type {alias['ingredient_type']}")
        else:
            alias_ingredient_id = alias['ingredient_id']
        
        # Mark as rejected
        self.db.update(
            'ingredient_aliases',
            {
                'status': 'rejected',
                'reviewed_by': admin_id,
                'reviewed_at': datetime.now(),
                'ingredient_id': alias_ingredient_id  # Update if created new
            },
            {'id': alias_id}
        )
```

---

## Unit Conversion

### Volume vs Weight

```python
class UnitConverter:
    """Convert between measurement units."""
    
    # Volume conversions (all to oz)
    VOLUME_CONVERSIONS = {
        'oz': 1.0,          # Base unit: fluid ounces
        'ml': 0.033814,     # 1 ml = 0.033814 oz
        'l': 33.814,        # 1 liter = 33.814 oz
        'cup': 8.0,         # 1 cup = 8 oz
        'tbsp': 0.5,        # 1 tablespoon = 0.5 oz
        'tsp': 0.166667,    # 1 teaspoon = 0.166667 oz
        'jigger': 1.0,      # 1 jigger = 1 oz (equiv)
        'dash': 0.0078125,  # 1 dash = 1/128 oz
        'barspoon': 0.1,    # ~1 barspoon = 0.1 oz
    }
    
    # Weight conversions (all to grams)
    WEIGHT_CONVERSIONS = {
        'g': 1.0,           # Base unit: grams
        'oz': 28.3495,      # 1 oz (weight) = 28.3495 g
        'lb': 453.592,      # 1 pound = 453.592 g
        'kg': 1000.0,       # 1 kg = 1000 g
    }
    
    # Ingredient unit mappings (what unit should be used)
    INGREDIENT_UNITS = {
        # Liquids - use oz
        'lime_juice': 'oz',
        'lemon_juice': 'oz',
        'simple_syrup': 'oz',
        'heavy_cream': 'oz',
        'agave_nectar': 'oz',
        
        # Solids - use grams
        'sugar': 'g',
        'salt': 'g',
        'strawberries': 'g',
        'raspberries': 'g',
        'fresh_basil': 'g',
        'cinnamon': 'g',
        
        # Count items
        'egg_white': 'count',
        'mint_leaf': 'count',
        'lime_wheel': 'count',
    }
    
    @staticmethod
    def is_volume_unit(unit: str) -> bool:
        """Check if unit is a volume measurement."""
        return unit.lower() in UnitConverter.VOLUME_CONVERSIONS
    
    @staticmethod
    def is_weight_unit(unit: str) -> bool:
        """Check if unit is a weight measurement."""
        return unit.lower() in UnitConverter.WEIGHT_CONVERSIONS
    
    @staticmethod
    def convert_volume(quantity: float, from_unit: str, to_unit: str = 'oz') -> float:
        """
        Convert volume measurement.
        
        Args:
            quantity: 250
            from_unit: 'ml'
            to_unit: 'oz' (default)
        
        Returns:
            quantity in target unit
        """
        from_unit = from_unit.lower()
        to_unit = to_unit.lower()
        
        if from_unit not in UnitConverter.VOLUME_CONVERSIONS:
            raise ValueError(f"Unknown volume unit: {from_unit}")
        if to_unit not in UnitConverter.VOLUME_CONVERSIONS:
            raise ValueError(f"Unknown volume unit: {to_unit}")
        
        # Convert to base (oz), then to target
        oz_value = quantity * UnitConverter.VOLUME_CONVERSIONS[from_unit]
        target_value = oz_value / UnitConverter.VOLUME_CONVERSIONS[to_unit]
        
        return round(target_value, 3)  # 3 decimal places
    
    @staticmethod
    def convert_weight(quantity: float, from_unit: str, to_unit: str = 'g') -> float:
        """
        Convert weight measurement.
        
        Args:
            quantity: 1
            from_unit: 'lb'
            to_unit: 'g' (default)
        
        Returns:
            quantity in target unit
        """
        from_unit = from_unit.lower()
        to_unit = to_unit.lower()
        
        if from_unit not in UnitConverter.WEIGHT_CONVERSIONS:
            raise ValueError(f"Unknown weight unit: {from_unit}")
        if to_unit not in UnitConverter.WEIGHT_CONVERSIONS:
            raise ValueError(f"Unknown weight unit: {to_unit}")
        
        # Convert to base (g), then to target
        gram_value = quantity * UnitConverter.WEIGHT_CONVERSIONS[from_unit]
        target_value = gram_value / UnitConverter.WEIGHT_CONVERSIONS[to_unit]
        
        return round(target_value, 1)  # 1 decimal place
    
    @staticmethod
    def standardize_unit(ingredient_name: str, quantity: float, user_unit: str) -> tuple[float, str]:
        """
        Convert user unit to standard unit based on ingredient type.
        
        Returns:
            (quantity, standard_unit)
        """
        # Look up what unit this ingredient should use
        expected_unit = UnitConverter.INGREDIENT_UNITS.get(
            ingredient_name.lower().replace(' ', '_'),
            'oz'  # Default to oz
        )
        
        user_unit = user_unit.lower()
        
        if expected_unit == 'count':
            # Can't convert, must be count
            if user_unit != 'count':
                raise ValueError(
                    f"Ingredient '{ingredient_name}' must be stored as 'count', "
                    f"not '{user_unit}'"
                )
            return quantity, 'count'
        
        elif expected_unit == 'oz':
            # Convert to oz if needed
            if UnitConverter.is_volume_unit(user_unit):
                converted_qty = UnitConverter.convert_volume(quantity, user_unit, 'oz')
                return converted_qty, 'oz'
            else:
                raise ValueError(
                    f"Ingredient '{ingredient_name}' expects volume (oz), "
                    f"got weight unit '{user_unit}'"
                )
        
        elif expected_unit == 'g':
            # Convert to grams if needed
            if UnitConverter.is_weight_unit(user_unit):
                converted_qty = UnitConverter.convert_weight(quantity, user_unit, 'g')
                return converted_qty, 'g'
            else:
                raise ValueError(
                    f"Ingredient '{ingredient_name}' expects weight (g), "
                    f"got volume unit '{user_unit}'"
                )
        
        return quantity, expected_unit
```

### Usage

```python
# Convert 250 ml to oz
qty = UnitConverter.convert_volume(250, 'ml', 'oz')  # 8.45 oz

# Convert 1 lb to grams
qty = UnitConverter.convert_weight(1, 'lb', 'g')  # 453.592 g

# Standardize ingredient
qty, unit = UnitConverter.standardize_unit('lemon juice', 0.75, 'ml')
# Returns: (0.025, 'oz')
```

---

## Circular Dependency Detection

### Database Triggers

See ARCHITECTURE.md for full SQL implementation.

### Python-Side Validation

```python
class DependencyValidator:
    """Validate and prevent circular dependencies."""
    
    def __init__(self, db_connection):
        self.db = db_connection
        self.logger = logging.getLogger(__name__)
    
    def detect_circular_prep_dependency(self, prep_id: int, ingredient_id: int) -> bool:
        """
        Check if adding prep_id -> ingredient_id would create a circle.
        
        Args:
            prep_id: Source prep
            ingredient_id: Target prep (ingredient_type='prep_ref')
        
        Returns:
            True if circular dependency would be created
        """
        if prep_id == ingredient_id:
            return True  # Self-reference
        
        # Get all ancestors of ingredient_id
        ancestors = self._get_prep_ancestors(ingredient_id)
        
        # If prep_id is in ancestors, adding this edge would create circle
        return prep_id in ancestors
    
    def _get_prep_ancestors(self, prep_id: int, depth: int = 0, max_depth: int = 10) -> set[int]:
        """
        Recursively get all prep IDs that this prep depends on (ancestors).
        
        Args:
            prep_id: Prep to analyze
            depth: Current recursion depth
            max_depth: Maximum recursion depth (prevent infinite loops)
        
        Returns:
            Set of ancestor prep IDs
        """
        if depth > max_depth:
            self.logger.warning(
                f"Max recursion depth reached for prep {prep_id}. "
                f"Possible circular dependency."
            )
            return set()
        
        # Find all preps that this prep references
        dependencies = self.db.query(
            """SELECT ingredient_id FROM prep_ingredients 
               WHERE prep_id = %s AND ingredient_type = 'prep_ref'""",
            prep_id
        )
        
        ancestors = set()
        for dep in dependencies:
            ingredient_id = dep['ingredient_id']
            ancestors.add(ingredient_id)
            
            # Recursively add ancestors of dependencies
            ancestors.update(self._get_prep_ancestors(ingredient_id, depth + 1, max_depth))
        
        return ancestors
    
    def detect_circular_batch_reference(self, cocktail_id: int, ingredient_id: int) -> bool:
        """
        Check if cocktail is referencing itself as a batch.
        
        Args:
            cocktail_id: Cocktail being modified
            ingredient_id: Batch being referenced
        
        Returns:
            True if self-reference
        """
        return cocktail_id == ingredient_id
```

---

## Error Logging

### Log All Errors

```python
class ErrorLogger:
    """Central error logging for cocktail database."""
    
    ERROR_TYPES = {
        'CIRCULAR_PREP_DEPENDENCY': 'Circular dependency in prep references',
        'CIRCULAR_BATCH_REF': 'Cocktail references itself as batch',
        'PARSE_ERROR': 'Failed to parse ingredient string',
        'UNIT_CONVERSION_ERROR': 'Unable to convert units',
        'INVALID_INGREDIENT': 'Ingredient not found and no match',
        'DUPLICATE_INGREDIENT': 'Ingredient appears multiple times in recipe',
    }
    
    SEVERITY_LEVELS = ['warning', 'error', 'critical']
    
    def __init__(self, db_connection):
        self.db = db_connection
        self.logger = logging.getLogger(__name__)
    
    def log_error(self, error_type: str, severity: str, 
                  entity_type: str, entity_id: int, 
                  message: str, context: dict = None) -> int:
        """
        Log an error to the database.
        
        Args:
            error_type: 'CIRCULAR_PREP_DEPENDENCY'
            severity: 'warning' | 'error' | 'critical'
            entity_type: 'prep' | 'cocktail' | 'ingredient'
            entity_id: ID of affected entity
            message: Human-readable error message
            context: Additional JSON context
        
        Returns:
            error_log.id for tracking
        """
        if error_type not in self.ERROR_TYPES:
            raise ValueError(f"Unknown error type: {error_type}")
        
        if severity not in self.SEVERITY_LEVELS:
            raise ValueError(f"Unknown severity: {severity}")
        
        error_id = self.db.insert('error_log', {
            'error_type': error_type,
            'severity': severity,
            'related_entity_type': entity_type,
            'related_entity_id': entity_id,
            'message': message,
            'context': json.dumps(context or {}),
            'resolved': False,
            'created_at': datetime.now()
        })
        
        # Log locally too
        log_method = getattr(self.logger, severity.lower(), self.logger.info)
        log_method(
            f"[{error_type}] {entity_type} {entity_id}: {message} "
            f"(error_log_id: {error_id})"
        )
        
        return error_id
    
    def resolve_error(self, error_id: int):
        """
        Mark an error as resolved (admin action).
        """
        self.db.update('error_log',
            {'resolved': True, 'resolved_at': datetime.now()},
            {'id': error_id}
        )
        
        self.logger.info(f"Marked error {error_id} as resolved")
    
    def get_unresolved_errors(self, severity: str = None) -> list[dict]:
        """
        Get all unresolved errors for admin dashboard.
        
        Args:
            severity: Filter by severity level (optional)
        
        Returns:
            List of error records
        """
        query = "SELECT * FROM error_log WHERE resolved = false"
        params = []
        
        if severity:
            query += " AND severity = %s"
            params.append(severity)
        
        query += " ORDER BY severity DESC, created_at DESC"
        
        return self.db.query(query, *params)
```

### Usage

```python
error_logger = ErrorLogger(db)

# Log circular dependency
error_id = error_logger.log_error(
    error_type='CIRCULAR_PREP_DEPENDENCY',
    severity='error',
    entity_type='prep',
    entity_id=5,
    message='Prep "Syrup: Vanilla" references "Cacao Matcha Cream" which references "Syrup: Vanilla"',
    context={
        'source_prep': 5,
        'target_prep': 12,
        'chain': [5, 12, 5]
    }
)

# Resolve when admin fixes the issue
error_logger.resolve_error(error_id)

# View dashboard
errors = error_logger.get_unresolved_errors(severity='critical')
for error in errors:
    print(f"[{error['severity'].upper()}] {error['message']}")
```

---

## Summary

- **Fuzzy Matching**: 80% threshold, admin approval, INGREDIENT_ALIASES tracking
- **Unit Conversion**: oz for volume, grams for weight, ingredient-specific defaults
- **Circular Dependencies**: Database triggers + Python validation + error logging
- **Error Handling**: Centralized ERROR_LOG table for all issues

All critical operations should log errors rather than silently failing.
