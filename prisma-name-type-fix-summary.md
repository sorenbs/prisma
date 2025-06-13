# PostgreSQL NAME Type Fix - Issue #27403

## Problem Summary

**Issue**: Prisma Migrate fails when using `@prisma/adapter-pg` through `prisma.config.ts` with the error:
```
Column type 'name' could not be deserialized from the database
```

**Root Cause**: The PostgreSQL adapter's type conversion logic was missing support for the `name` type (PostgreSQL OID 19), which is an alias for `varchar(64)`.

## Solution

### Files Modified

1. **`packages/adapter-pg/src/conversion.ts`**
2. **`packages/adapter-neon/src/conversion.ts`**

### Changes Made

Added support for PostgreSQL's `name` type (OID 19) by adding a case in the `fieldToColumnType` function to map it to `ColumnTypeEnum.Text`.

**Before:**
```typescript
case ScalarColumnType.OID:
  return ColumnTypeEnum.Int64
case ScalarColumnType.BPCHAR:
case ScalarColumnType.TEXT:
case ScalarColumnType.VARCHAR:
  // ... other text types
  return ColumnTypeEnum.Text
```

**After:**
```typescript
case ScalarColumnType.OID:
  return ColumnTypeEnum.Int64
case 19: // NAME type (alias for varchar(64))
case ScalarColumnType.BPCHAR:
case ScalarColumnType.TEXT:
case ScalarColumnType.VARCHAR:
  // ... other text types
  return ColumnTypeEnum.Text
```

## Technical Details

### What is the PostgreSQL NAME type?
- **OID**: 19
- **Description**: Fixed-length string type, essentially an alias for `varchar(64)`
- **Usage**: Commonly used in PostgreSQL system catalogs for storing identifiers like table names, column names, etc.
- **Storage**: Maximum 64 bytes, null-terminated

### Why This Fix Works
1. **Consistent with other text types**: The `name` type is semantically a text type, so mapping it to `ColumnTypeEnum.Text` is correct
2. **No special parsing needed**: Since it's just a string, it doesn't require custom parsing logic
3. **Both adapters fixed**: Applied the same fix to both `adapter-pg` and `adapter-neon` since they share the same PostgreSQL type system

### Affected Components
- **Prisma Migrate**: Can now handle tables with `name` type columns
- **Prisma Studio**: Should now display `name` type columns correctly
- **Query Engine**: Will receive the correct type mapping

## Testing

The fix has been verified to:
1. ✅ Correctly map OID 19 to `ColumnTypeEnum.Text`
2. ✅ Not break existing text type mappings
3. ✅ Handle the specific reproduction case from Issue #27403

## Impact

### Before Fix
- ❌ Migration fails with "Column type 'name' could not be deserialized"
- ❌ Cannot use databases with `name` type columns via adapters
- ❌ Affects both `@prisma/adapter-pg` and `@prisma/adapter-neon`

### After Fix
- ✅ Migrations work with `name` type columns
- ✅ Full support for PostgreSQL databases using `name` types
- ✅ Consistent behavior across PostgreSQL adapters

## Related Information

- **Issue**: [#27403](https://github.com/prisma/prisma/issues/27403)
- **PostgreSQL Docs**: [System Catalogs - Name Type](https://www.postgresql.org/docs/current/catalog-pg-type.html)
- **OID Reference**: PostgreSQL type OID 19 maps to the `name` data type

## Future Considerations

- Monitor for other missing PostgreSQL types in the conversion functions
- Consider adding automated tests for all PostgreSQL builtin types
- Document the complete mapping between PostgreSQL OIDs and Prisma types