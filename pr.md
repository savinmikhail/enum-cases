# Add values() Method to BackedEnum

## Summary

Introduce a native `BackedEnum::values()` static method that returns the list of all backing values (int|string) of a backed enum's cases in declaration order. This eliminates boilerplate commonly implemented across projects and aligns with the existing `cases()` API.

Target: master (PHP 8.6)

## Motivation

The ecosystem repeatedly implements the same pattern to produce an array of enum values:

```php
enum Status: string {
    case Draft = 'draft';
    case Published = 'published';

    // Boilerplate implemented in countless codebases today
    public static function values(): array {
        return array_column(self::cases(), 'value');
    }
}
```

This pattern appears widely across GitHub and frameworks, often implemented directly or via traits (which hides usage from code search).

### Quantitative Evidence

**Direct implementations** (comprehensive GitHub code search as of 2025-11-12):

| Pattern | Results | Search Link |
|---------|---------|-------------|
| `array_column(self::cases(), 'value')` | 3,600 | [search](https://github.com/search?q=language%3APHP+%22array_column%28self%3A%3Acases%28%29%2C+%27value%27%29%22+%22enum%22&type=code) |
| `array_map(fn($case) => $case->value, self::cases())` | 330 | [search](https://github.com/search?q=language%3APHP+%22array_map%28fn%28%24case%29+%3D%3E+%24case-%3Evalue%2C+self%3A%3Acases%28%29%29%22&type=code) |
| `return array_map` + `self::cases()` + `->value` | 1,900 | [search](https://github.com/search?q=language%3APHP+%22return+array_map%22+%22self%3A%3Acases%28%29%22+%22-%3Evalue%22&type=code) |
| `return array_map` + `fn($case) => $case->value` | 324 | [search](https://github.com/search?q=language%3APHP+%22return+array_map%22+%22fn%28%24case%29+%3D%3E+%24case-%3Evalue%22&type=code) |
| `function values()` + `return array_map` + `self::cases()` | 784 | [search](https://github.com/search?q=language%3APHP+%22function+values%28%29%22+%22return+array_map%22+%22self%3A%3Acases%28%29%22&type=code) |
| `function toArray()` + `array_map` + `self::cases()` + `->value` | 236 | [search](https://github.com/search?q=language%3APHP+%22function+toArray%28%29%22+%22array_map%22+%22self%3A%3Acases%28%29%22+%22-%3Evalue%22&type=code) |
| `function getValues()` + `array_map` + `self::cases()` | 196 | [search](https://github.com/search?q=language%3APHP+%22function+getValues%28%29%22+%22array_map%22+%22self%3A%3Acases%28%29%22&type=code) |
| `function values()` + `foreach` loop + `->value` | 90 | [search](https://github.com/search?q=language%3APHP+%22function+values%28%29%22+%22foreach%22+%22self%3A%3Acases%28%29%22+%22%5B%5D+%3D+%24%22+%22-%3Evalue%22&type=code) |
| **Total direct implementations** | **~7,460** | — |

**Implementation distribution:**
- `array_column` pattern: 3,600 (48%) - most popular ✨
- `array_map` patterns: 2,554 (34%)
- Other patterns: 1,306 (18%)

**Trait pattern multiplier** ([2,900 results](https://github.com/search?q=language%3APHP+%22trait%22++%22function+values%28%29%22&type=code)):
- Many projects factor this into a trait (e.g., `EnumValuesTrait`) and then `use` it in dozens of enums
- Direct search counts significantly understate total usage
- Conservative estimate: **20,000-40,000 total implementations** in the ecosystem

**Qualitative evidence** (frameworks & libraries):
- symfony/symfony [usage](https://github.com/symfony/symfony/blob/f656af9231091847f3ee45eadd6569451df79f4d/src/Symfony/Component/TypeInfo/TypeIdentifier.php#L43)
- box-project/box [usage](https://github.com/box-project/box/blob/b3c3ccecf04c27084919bae44356182645372e25/src/Phar/DiffMode.php#L31)
- Legacy library [values() method](https://github.com/myclabs/php-enum/blob/191882a09b5abb316a1166255b1c6392e2f91e14/src/Enum.php#L173) (pre-8.1 enums)

Providing a native `values()` method:
- Removes boilerplate and fragmentation (different traits/implementations)
- Improves discoverability and consistency (parallels `cases()` nicely)
- Simplifies migrations from legacy enum libraries and existing project traits

## Proposal

Add the following method to the `BackedEnum` interface:

```php
interface BackedEnum extends UnitEnum
{
    /**
     * Returns an indexed array of all backing values for the enum cases.
     *
     * @return int[]|string[]
     */
    public static function values(): array;
}
```

### Semantics

- Available only for backed enums (int|string). Not available on unit enums.
- Returns a 0-based, indexed array of the backing values in declaration order.
- For int-backed enums, returns `array<int>`; for string-backed, `array<string>`.

### Examples

```php
enum Priority: int { 
    case Low = 1; 
    case High = 10; 
}
Priority::values(); // [1, 10]

enum Color: string { 
    case Red = 'red'; 
    case Blue = 'blue'; 
}
Color::values(); // ['red', 'blue']
```

## Implementation Options

Two implementation options are possible, differing only in whether the interface declares a return type:

### Option A: Interface WITH Return Type (Current Implementation)

```php
interface BackedEnum extends UnitEnum
{
    public static function values(): array;  // ← WITH return type
}
```

**Characteristics:**
- Full type safety in interface contract
- IDEs and static analyzers can infer return type
- Consistent with modern PHP practices (explicit return types)
- Compatible with `cases()` which also has `: array` in interface

### Option B: Interface WITHOUT Return Type (Alternative)

```php
interface BackedEnum extends UnitEnum
{
    public static function values();  // ← NO return type
}
```

**Characteristics:**
- Permissive interface contract
- Allows implementations to specify any return type or none
- All existing userland implementations remain compatible
- Provides the method while maintaining maximum BC compatibility

## Backward Compatibility Analysis

Comprehensive GitHub search analysis (2025-11-12) of existing `values()` implementations:

| Category | Count | % | Search Link |
|----------|-------|---|-------------|
| **Total enums with values()** | **6,800** | **100%** | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29%22&type=code) |
| Compatible: `: array` return type | 6,200 | 91.2% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29%3A+array%22&type=code) |
| Missing return type | 64 | 0.9% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29+%7B%22&type=code) |
| Incompatible: `: string` | 4 | 0.06% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29%3A+string%22&type=code) |
| Incompatible: `: int` | 0 | 0% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29%3A+int%22&type=code) |
| Incompatible: `: Closure` | 0 | 0% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29%3A+Closure%22&type=code) |
| Incompatible: `: iterable` | 1 | 0.01% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29%3A+iterable%22&type=code) |
| Incompatible: `: Iterator` | 2 | 0.03% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29%3A+Iterator%22&type=code) |
| Incompatible: `: Generator` | 0 | 0% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22public+static+function+values%28%29%3A+Generator%22&type=code) |
| Returns case objects (not values) | 0 | 0% | [search](https://github.com/search?q=language%3APHP+%22enum%22+%22function+values%28%29%22+%22return+self%3A%3Acases%28%29%22+-%22-%3Evalue%22&type=code) |
| Unaccounted (~529 remaining) | ~529 | ~7.8% | — |

### BC Impact: Option A (WITH Return Type)

**Breaks:** ❌ **71-600** implementations (1.0-8.8%)

Adding `: array` to the interface signature causes compile-time errors for implementations without it:

**Confirmed breaks:**
- Missing return type: 64 (0.9%)
- Wrong return types: 7 (0.1%)
- **Total confirmed:** 71 (1.0%)

**Possible breaks:**
- Unaccounted implementations (~529) may have missing or incompatible types
- **Estimated range:** 71-600 (1.0-8.8%)

**Example breaking code:**
```php
enum Status: string {
    case Active = 'active';
    
    public static function values() {  // ❌ Missing : array
        return array_column(self::cases(), 'value');
    }
}
// Fatal error: Declaration of Status::values() must be compatible 
// with BackedEnum::values(): array
```

**Migration required:**
```php
- public static function values() {
+ public static function values(): array {
      return array_column(self::cases(), 'value');
  }
```

### BC Impact: Option B (WITHOUT Return Type)

**Breaks:** ✅ **ZERO** (0%)

Without a return type in the interface, PHP allows implementations to specify **any** return type (or no return type). All existing implementations remain compatible:

```php
// All of these are compatible with interface without return type:
public static function values(): array { }      // ✅ Compatible
public static function values() { }             // ✅ Compatible
public static function values(): string { }     // ✅ Compatible
public static function values(): Iterator { }   // ✅ Compatible
```

**Example - all work fine:**
```php
// Interface
interface BackedEnum {
    public static function values();  // No return type
}

// Implementation 1 - works
enum Status1: string {
    public static function values(): array {
        return array_column(self::cases(), 'value');
    }
}

// Implementation 2 - also works
enum Status2: string {
    public static function values() {
        return array_column(self::cases(), 'value');
    }
}

// Implementation 3 - even this works (though semantically wrong)
enum Status3: string {
    public static function values(): string {
        return 'custom';
    }
}
```

All three are **syntactically compatible** with the interface. Semantic correctness (returning actual array of values) is enforced by the native implementation, not the interface contract.

### Option Comparison

| Aspect | Option A<br/>(WITH `: array`) | Option B<br/>(WITHOUT type) |
|--------|-------------------------------|------------------------------|
| **BC Breaks** | 71-600 (1.0-8.8%) | **0 (0%)** |
| **Type Safety** | Full | Partial |
| **IDE Support** | Better | Good |
| **Static Analysis** | Better | Partial |
| **Migration Required** | Add `: array` to 71-600 | None |
| **Consistency** | Matches `cases(): array` | Allows flexibility |
| **Ecosystem Disruption** | Small (1-9%) | None |

### Recommendation

**Option B (WITHOUT return type)** is recommended because:

1. **Zero BC breaks:**
   - All 6,800 existing implementations continue working
   - No migration required
   - No ecosystem disruption

2. **Still provides standardization:**
   - Native implementation returns proper array
   - Method is discoverable on all BackedEnum
   - Eliminates boilerplate for new code

3. **Graceful deprecation path:**
   - Start without return type (PHP 8.6) - zero breaks
   - Add E_DEPRECATED for non-array return in PHP 8.7
   - Add `: array` to interface in PHP 9.0
   - Gives ecosystem years to migrate naturally

4. **Practical over theoretical:**
   - 91.2% already have correct signature
   - Remaining 8.8% can migrate over time
   - No forced breakage today

**Option A** could be reconsidered for PHP 9.0 (major version where BC breaks are more acceptable) or if the community strongly prefers immediate full type safety despite the breaks.

## Current Implementation

The current PR implements **Option A** (WITH return type `: array`).

The implementation can be easily changed to **Option B** by removing `: array` from the interface declaration in `Zend/zend_enum.stub.php`:

```diff
  interface BackedEnum extends UnitEnum
  {
-     public static function values(): array;
+     public static function values();
  }
```

## Implementation Details

### Engine changes (Zend)

**Add stub and arginfo:**
- `Zend/zend_enum.stub.php`: add `BackedEnum::values()` signature
  ```php
  // Option A (current):
  public static function values(): array;
  
  // Option B (alternative):
  public static function values();
  ```
- `Zend/zend_enum_arginfo.h`: regenerated (committed)

**Add interned string identifier:**
- `Zend/zend_string.h`: add `ZEND_STR_VALUES` ("values")

**Implement and register the method:**
- `Zend/zend_enum.c`:
   - Implement `zend_enum_values_func()`, extracting the `value` property from each case
   - Register for all backed enums
   - Native implementation always returns array regardless of interface declaration

### Tests

**Reflection tests:**
- `ext/reflection/tests/BackedEnum_values_reflection.phpt` - ensure method appears
- `ext/reflection/tests/ReflectionEnum_toString_backed_int.phpt` - update toString
- `ext/reflection/tests/ReflectionEnum_toString_backed_string.phpt` - update toString

**Enum behavior tests:**
- `Zend/tests/enum/backed-values-int.phpt` - int-backed enum values
- `Zend/tests/enum/backed-values-string.phpt` - string-backed enum values
- `Zend/tests/enum/backed-values-empty.phpt` - empty enum edge case
- `Zend/tests/enum/backed-values-order.phpt` - declaration order preservation
- `Zend/tests/enum/backed-values-not-on-pure.phpt` - pure enums don't have values()
- `Zend/tests/enum/backed-values-ignore-regular-consts.phpt` - regular constants ignored

**BC compatibility test (for Option A - expected failure):**
```php
<?php
enum Status: string {
    case Active = 'active';
    
    // This fails in Option A (missing : array)
    public static function values() {
        return array_column(self::cases(), 'value');
    }
}
?>
--EXPECTF--
Fatal error: Declaration of Status::values() must be compatible with BackedEnum::values(): array
```

### Documentation in-tree

- `NEWS`: announce the feature under Core
- `UPGRADING`:
   - List under "New Features"
   - For Option A: List under "Backward Incompatible Changes"
   - For Option B: No BC section needed

### Manual (php/doc-en)

To be added in a separate PR:
- BackedEnum interface page: `values()` signature, description, examples
- Enumerations guide: mention `values()` alongside `cases()/from()/tryFrom()`
- Migration guide for 8.6: add entry for `BackedEnum::values()`

## Performance

Native implementation provides:
- O(n) complexity over number of cases
- Similar performance to `array_column(self::cases(), 'value')`
- No userland call overhead

**Benchmark** (100,000 iterations):
```
array_column: 0.012s
array_map:    0.035s (2.9x slower)
native:       ~0.012s (similar to array_column)
```

Reference: https://3v4l.org/Q5AYg

**Note:** `array_column` is the most popular pattern (48% of implementations, 3,600 results) and is ~3x faster than `array_map`.

## Alternatives Considered

### Different method name

- `getValues()`, `valueList()`: More verbose, doesn't match `cases()` style
- `toArray()`: Ambiguous - case objects or values? Names or values?

**Decision:** `values()` best matches:
- Existing community usage (7,460+ examples)
- Parallel naming with `cases()`
- Simplicity and clarity

### Virtual/magic properties

`Status::$values` instead of `Status::values()`

**Rejected:**
- Enums cannot have static properties
- Inconsistent with `cases()`, `from()`, `tryFrom()` (all methods)

## Impact on Ecosystem

### Positive impacts

**For Option A (WITH return type):**
- ✅ **91.2% of implementations (6,200)** continue working unchanged
- ✅ **1.0% confirmed breaks (71)** require mechanical fix (add `: array`)
- ✅ **Up to 8.8% possible breaks** if unaccounted implementations lack types
- ✅ Better type safety and IDE support

**For Option B (WITHOUT return type):**
- ✅ **100% of implementations (6,800)** continue working unchanged
- ✅ **Zero BC breaks** - no migration needed
- ✅ **Zero ecosystem disruption**
- ✅ Native implementation still returns array
- ✅ Can add stricter typing in PHP 9.0

**Both options:**
- ✅ New enums automatically get `values()` without boilerplate
- ✅ Standardization reduces ecosystem fragmentation
- ✅ Improved discoverability for new developers
- ✅ Simplified documentation (single standard approach)

## Prior Art

No previous RFCs proposing a `BackedEnum::values()` method were found.

### Related RFCs

Review of the [PHP RFC index](https://wiki.php.net/rfc) shows enum-related proposals:
- [Add get_declared_enums() function (2024)](https://wiki.php.net/rfc/get_declared_enums)
- [Fetch properties of enums in const expressions (2022)](https://wiki.php.net/rfc/fetch_property_in_const_expressions)
- [Enumerations (2020)](https://wiki.php.net/rfc/enumerations)
- [Allow static properties in enums (2021)](https://wiki.php.net/rfc/enum_allow_static_properties)
- [Auto-implement Stringable for string backed enums (2022)](https://wiki.php.net/rfc/enum_allow_static_properties)
- [Sorting enum (2021)](https://wiki.php.net/rfc/sorting_enum)

None address adding a convenience method returning backing values.

### Other languages

- **TypeScript:** `Object.values(EnumType)`
- **Python:** `[e.value for e in EnumType]`
- **myclabs/php-enum** (legacy): Had `values()` method (4,900 stars, pre-8.1)


## Checklist

- [x] Engine implementation
- [x] Arginfo generated and committed
- [x] Tests added (enum + reflection + BC compatibility)
- [x] NEWS and UPGRADING updated
- [x] Comprehensive BC analysis for both options with GitHub searches
- [ ] Manual docs (php/doc-en) PR to be submitted after review

## Links

- **RFC:** https://wiki.php.net/rfc/add_values_method_to_backed_enum
- **Discussion:** https://externals.io/message/129186
- **Pre-RFC Discussion:** https://externals.io/message/129107
- **Implementation:** https://github.com/php/php-src/pull/20398

Thank you for reviewing!
