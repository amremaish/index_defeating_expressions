# Database Index Optimization Guide

A comprehensive guide to identifying and fixing SQL expressions that defeat database indexes, leading to poor query performance.

## Table of Contents
- [Overview](#overview)
- [Common Index-Defeating Patterns](#common-index-defeating-patterns)
- [Quick Reference](#quick-reference)
- [Contributing](#contributing)

## Overview

Database indexes are crucial for query performance, but certain SQL expressions can prevent the query optimizer from using them effectively. This guide covers 20 common patterns that defeat indexes and provides practical solutions.

**Key Principle:** Keep indexed columns "bare" on one side of comparisons and compare to constants or bind parameters of the same type.

## Common Index-Defeating Patterns

### 1. Functions on Indexed Columns

**❌ Problem**
```sql
WHERE LOWER(email) = 'amr@site.com'
WHERE DATE(created_at) = '2025-08-24'
```

**Why it fails:** B-tree indexes can't use raw column ordering when columns are transformed first.

**✅ Solution**
```sql
-- Compare raw column to prepared constant
WHERE email = 'amr@site.com'  -- (or use matching collation)
WHERE created_at >= '2025-08-24' AND created_at < '2025-08-25'

-- Or create functional index (PostgreSQL)
CREATE INDEX ON users (LOWER(email));
```

### 2. Arithmetic on Indexed Columns

**❌ Problem**
```sql
WHERE amount * 1.2 > 100
WHERE id + 1 = 42
```

**✅ Solution**
```sql
-- Move math to the constant side
WHERE amount > 100 / 1.2
WHERE id = 41
```

### 3. Leading Wildcards in Text Searches

**❌ Problem**
```sql
WHERE name LIKE '%ma'
WHERE name ~ '...ma$'
```

**Why it fails:** B-tree uses left-to-right ordering; leading wildcards destroy prefix search capability.

**✅ Solution**
```sql
-- Prefer prefix searches
WHERE name LIKE 'amr%'

-- For substring/regex: use appropriate indexes
-- PostgreSQL: pg_trgm GIN/GIST
-- MySQL: FULLTEXT indexes
```

### 4. Implicit Type Casts on Columns

**❌ Problem**
```sql
WHERE phone = 2012345678          -- when phone is VARCHAR
WHERE text_col = 123              -- PostgreSQL may cast text_col
```

**✅ Solution**
```sql
-- Cast literals, not columns
WHERE phone = '2012345678'

-- Keep data types aligned with index definitions
```

### 5. Date/Time Function Wrappers

**❌ Problem**
```sql
WHERE EXTRACT(YEAR FROM created_at) = 2025
```

**✅ Solution**
```sql
-- Use range queries on raw columns
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'
```

### 6. Expressions in JOIN Predicates

**❌ Problem**
```sql
ON a.key = b.key + 1
```

**✅ Solution**
```sql
-- Better approaches:
-- 1. Store comparable values in indexed columns
-- 2. Compute values before join
-- 3. Use raw column joins with adjusted logic
```

### 7. Inequality and Negative Predicates

**❌ Problem (Often Expensive)**
```sql
WHERE status <> 'active'
WHERE NOT (x = 5)
WHERE field NOT IN (...)
WHERE field NOT LIKE 'x%'
```

**Why it fails:** Hard to create selective ranges; often requires scanning large data portions.

**✅ Solution**
```sql
-- Use positive, selective conditions
WHERE status IN ('pending', 'blocked')

-- Consider partial indexes for dominant subsets
```

### 8. OR Across Different Columns

**❌ Problem**
```sql
WHERE first_name = 'Amr' OR last_name = 'Amr'
```

**Why it fails:** Single-column indexes only help one side of the OR.

**✅ Solution**
- Create composite/covering indexes
- Use bitmap OR-able indexes (PostgreSQL)
- Rewrite as `UNION ALL` of sargable queries + `DISTINCT`

### 9. Skipping Leftmost Columns in Composite Indexes

**❌ Problem (MySQL & Most B-trees)**
```sql
-- Index on (country, city), but querying only:
WHERE city = 'Cairo'
```

**✅ Solution**
```sql
-- Respect leftmost prefix rule
WHERE country = 'Egypt' AND city = 'Cairo'

-- Or create separate index on city
CREATE INDEX idx_city ON table_name(city);
```

### 10. ORDER BY/GROUP BY Mismatches

**❌ Problem**
```sql
ORDER BY LOWER(name)              -- no matching functional index
ORDER BY created_at DESC          -- while index is ASC (engine-dependent)
```

**✅ Solution**
```sql
-- Match index keys and sort order exactly
CREATE INDEX idx_name_lower ON table_name(LOWER(name));
CREATE INDEX idx_created_desc ON table_name(created_at DESC);
```

### 11. SELECT Patterns Blocking Index-Only Scans

**❌ Problem**
Selecting many non-indexed columns forces table access even with sargable predicates.

**✅ Solution**
- Create covering indexes (include needed columns)
- Limit selected columns to indexed ones when possible

### 12. Non-Deterministic Functions

**❌ Problem**
```sql
-- This is actually OK - column stays raw
WHERE updated_at < NOW() - INTERVAL '1 day'

-- But volatile functions in indexes are problematic
```

**✅ Solution**
```sql
-- Use stable constants
WHERE updated_at >= :cutoff  -- bind cutoff = NOW() - interval

-- Functional indexes: use only immutable/stable functions
```

### 13. JSON/Array Expressions Without Proper Indexes

**❌ Problem**
```sql
WHERE jsonb_col->>'code' = 'X'    -- no functional index
WHERE tags @> ARRAY['x']          -- no GIN index
```

**✅ Solution**
```sql
-- PostgreSQL JSON
CREATE INDEX ON table_name((jsonb_col->>'code'));
-- or
CREATE INDEX ON table_name USING GIN(jsonb_col);

-- PostgreSQL Arrays
CREATE INDEX ON table_name USING GIN(tags);
```

### 14. Case-Insensitive Comparisons

**❌ Problem**
```sql
WHERE name ILIKE 'amr%'  -- PostgreSQL with plain btree
```

**✅ Solution**
```sql
-- PostgreSQL: functional index
CREATE INDEX ON table_name(LOWER(name));
-- then use:
WHERE LOWER(name) LIKE 'amr%'

-- Or use citext type with appropriate index
```

### 15. User-Defined Functions Without Index Support

**❌ Problem**
```sql
WHERE my_custom_normalizer(col) = 'x'  -- no functional index
```

**✅ Solution**
```sql
-- Create functional index matching exact expression
CREATE INDEX ON table_name(my_custom_normalizer(col));
-- Ensure function is marked IMMUTABLE
```

### 16. Date/Number Formatting

**❌ Problem**
```sql
WHERE TO_CHAR(created_at, 'YYYY-MM') = '2025-08'
```

**✅ Solution**
```sql
-- Use range predicates on raw types
WHERE created_at >= '2025-08-01' AND created_at < '2025-09-01'
```

### 17. Mixed Predicates with Complex Logic

**❌ Problem**
```sql
WHERE COALESCE(flag, false) = true
```

**✅ Solution**
```sql
-- Option 1: Normalize data with default values
-- Option 2: Create functional index
CREATE INDEX ON table_name((COALESCE(flag, false)));
```

### 18. LIKE Patterns with Leading Underscores

**❌ Problem**
```sql
WHERE code LIKE '_234%'  -- first character unknown
```

**✅ Solution**
- Constrain leading characters when possible
- Use full-text or trigram indexes for fuzzy matching

### 19. OR with Disjoint Ranges

**❌ Problem**
```sql
WHERE created_at < '2025-01-01' OR created_at > '2025-08-01'
```

**✅ Solution**
```sql
-- Sometimes UNION ALL performs better
(SELECT * FROM table_name WHERE created_at < '2025-01-01')
UNION ALL
(SELECT * FROM table_name WHERE created_at > '2025-08-01')
```

### 20. Computed Search Keys Not Indexed

**❌ Problem**
Searching on normalized values (formatted phones, canonical emails) but only storing raw fields.

**✅ Solution**
- Store normalized values in indexed shadow columns
- Create functional indexes on normalization expressions

## Quick Reference

### ✅ DO
- Keep indexed columns bare in comparisons
- Compare to constants or bind parameters
- Use matching data types
- Respect composite index column order
- Create functional indexes for computed values
- Use range queries instead of function wrappers

### ❌ DON'T
- Apply functions to indexed columns in WHERE clauses
- Use arithmetic on indexed columns
- Start LIKE patterns with wildcards
- Force type casts on columns
- Use negative predicates when positive alternatives exist
- Skip leftmost columns in composite indexes

## Database-Specific Notes

### PostgreSQL
- Supports functional indexes: `CREATE INDEX ON table(LOWER(column))`
- GIN indexes for JSON, arrays, full-text search
- Can use bitmap index scans for OR conditions
- pg_trgm extension for fuzzy text matching

### MySQL
- Limited functional index support (MySQL 8.0+)
- FULLTEXT indexes for text search
- Supports index condition pushdown
- Cannot skip leftmost columns in composite indexes

### SQL Server
- Computed columns can be indexed
- Includes columns in covering indexes
- Query optimizer may use index intersection
- Supports filtered indexes (partial indexes)

## Contributing

Found additional patterns or have improvements? Contributions are welcome!

1. Fork the repository
2. Add your examples with clear explanations
3. Include database-specific solutions where applicable
4. Submit a pull request

## License

This guide is provided under the MIT License. Feel free to use and adapt for your projects.

---
