## Summary

Introduce a native `BackedEnum::values()` static method that returns the list of all backing values (int|string) of a backed enum’s cases in declaration order. This eliminates boilerplate commonly implemented across projects and aligns with the existing `cases()` API.

Target: master (PHP 8.6)

## Motivation

The ecosystem repeatedly implements the same pattern to produce an array of enum values:

```php
enum Status: string {
    case Draft = 'draft';
    case Published = 'published';

    // Boilerplate implemented in countless codebases today
    public static function values(): array {
        return array_map(fn($case) => $case->value, self::cases());
    }
}
```

This pattern appears widely across GitHub and frameworks, often implemented directly or via traits (which hides usage from code search). A conservative summary of real-world evidence:

**Quantitative (code search + trait multiplier)**:

| Pattern | GitHub search | Results |
| --- | --- | ---: |
| `array_map(fn($case) => $case->value, self::cases())` | [click](https://github.com/search?q=language%3APHP+%22array_map%28fn%28%24case%29+%3D%3E+%24case-%3Evalue%2C+self%3A%3Acases%28%29%29%22&type=code) | ~330 |
| `return array_map` + `self::cases()` + `->value` | [click](https://github.com/search?q=language%3APHP+%22return+array_map%22+%22self%3A%3Acases%28%29%22+%22-%3Evalue%22&type=code) | ~1,900 |
| `return array_map` + `fn($case) => $case->value` | [click](https://github.com/search?q=language%3APHP+%22return+array_map%22+%22fn%28%24case%29+%3D%3E+%24case-%3Evalue%22&type=code) | ~324 |
| `function values()` + `return array_map` + `self::cases()` | [click](https://github.com/search?q=language%3APHP+%22function+values%28%29%22+%22return+array_map%22+%22self%3A%3Acases%28%29%22&type=code) | ~784 |
| `function toArray()` + `array_map` + `self::cases()` + `->value` | [click](https://github.com/search?q=language%3APHP+%22function+toArray%28%29%22+%22array_map%22+%22self%3A%3Acases%28%29%22+%22-%3Evalue%22&type=code) | ~236 |
| `function getValues()` + `array_map` + `self::cases()` | [click](https://github.com/search?q=language%3APHP+%22function+getValues%28%29%22+%22array_map%22+%22self%3A%3Acases%28%29%22&type=code) | ~196 |
| `function values()` + `foreach` + `self::cases()` + `[] = $` + `->value` | [click](https://github.com/search?q=language%3APHP+%22function+values%28%29%22+%22foreach%22+%22self%3A%3Acases%28%29%22+%22%5B%5D+%3D+%24%22+%22-%3Evalue%22&type=code) | ~90 |
| Total | — | ~3,860 |

**Trait pattern multiplier**:
- Many projects factor this into a trait (e.g., `EnumValuesTrait`) and then `use` it in dozens of enums, so direct search counts significantly understate usage.
- PHP manual example shows [trait](https://www.php.net/manual/en/language.enumerations.examples.php#128866) approach

**Qualitative (frameworks & libraries)**:
- symfony/symfony [usage](https://github.com/symfony/symfony/blob/f656af9231091847f3ee45eadd6569451df79f4d/src/Symfony/Component/TypeInfo/TypeIdentifier.php#L43)
- box-project/box [usage](https://github.com/box-project/box/blob/b3c3ccecf04c27084919bae44356182645372e25/src/Phar/DiffMode.php#L31)
- Legacy library with [values()](https://github.com/myclabs/php-enum/blob/191882a09b5abb316a1166255b1c6392e2f91e14/src/Enum.php#L173)

Providing a native `values()` method:
- Removes boilerplate and fragmentation (different traits/implementations).
- Improves discoverability and consistency (parallels `cases()` nicely).
- Simplifies migrations from legacy enum libraries and existing project traits.

## Proposal

Add the following method to the `BackedEnum` interface:

```php
interface BackedEnum extends UnitEnum
{
    public static function values(): array;
}
```

Semantics:
- Available only for backed enums (int|string). Not available on unit enums.
- Returns a 0-based, indexed array of the backing values in declaration order.
- For int-backed enums, returns `array<int>`; for string-backed, `array<string>`.

Examples:

```php
enum Priority: int { case Low = 1; case High = 10; }
Priority::values(); // [1, 10]

enum Color: string { case Red = 'red'; case Blue = 'blue'; }
Color::values(); // ['red', 'blue']
```

## Implementation Details

Engine changes (Zend):
- Add stub and arginfo
    - `Zend/zend_enum.stub.php`: add `BackedEnum::values()` signature.
    - `Zend/zend_enum_arginfo.h`: regenerated (committed) to include `arginfo_class_BackedEnum_values`.
- Add interned string identifier
    - `Zend/zend_string.h`: add `ZEND_STR_VALUES` ("values").
- Implement and register the method
    - `Zend/zend_enum.c`:
        - Implement `zend_enum_values_func()`, mirroring `cases()` but extracting the `value` property of each case object.
        - Register for backed enums in both the internal method table and `zend_enum_register_funcs()` (user classes).

Tests:
- Reflection: ensure the method appears on backed enums
    - `ext/reflection/tests/BackedEnum_values_reflection.phpt`
    - Update toString expectations for backed enums:
        - `ext/reflection/tests/ReflectionEnum_toString_backed_int.phpt`
        - `ext/reflection/tests/ReflectionEnum_toString_backed_string.phpt`
- Enum behavior tests
    - `Zend/tests/enum/backed-values-int.phpt`
    - `Zend/tests/enum/backed-values-string.phpt`
    - `Zend/tests/enum/backed-values-empty.phpt`
    - `Zend/tests/enum/backed-values-order.phpt`
    - `Zend/tests/enum/backed-values-not-on-pure.phpt`
    - `Zend/tests/enum/backed-values-ignore-regular-consts.phpt`

Documentation in-tree:
- `NEWS`: announce the feature under Core.
- `UPGRADING`: list under “New Features”.

Manual (php/doc-en) to be added in a separate PR:
- BackedEnum interface page: `values()` signature, description, examples.
- Enumerations guide: mention `values()` alongside `cases()/from()/tryFrom()`.
- Migration guide for 8.6: add an entry for `BackedEnum::values()`.

## Backward Compatibility

This is a BREAKING CHANGE for code that already declares a `values()` method on a backed enum (directly or via a trait).

### Impact Assessment

Based on ecosystem scans and common patterns in public code:
- 3,860+ direct implementations of `values()` on backed enums will fail to compile.
- ~20,000–40,000 additional usages via traits (e.g., `EnumValuesTrait`) are likely to fail.
- Affects userland frameworks and applications (e.g., Symfony, Laravel) and custom projects that converged on the same implementation.

Thanks, @vudaltsov for highlighting the risks to me.

### Error Behavior

Declaring a `values()` method on a backed enum after this change results in a compile-time fatal error due to method redeclaration:

```php
enum Status: string {
    case Active = 'active';

    public static function values(): array { /* userland */ }
    // Fatal error: Cannot redeclare BackedEnum::values()
}
```

Notes:
- This method exists only on backed enums; unit enums do not expose `values()`.
- This is not an override; it is a redeclaration of an engine-registered method.

### Migration Required

All affected code must:
1. Remove custom `values()` implementations from backed enums.
2. Remove or stop using traits that provide `values()`.
3. Re-run tests to ensure behavior remains consistent with the native implementation.

Rationale for accepting the break:
- Implementations across the ecosystem are effectively identical; migration is mechanical (delete method/trait usage).
- Behavior remains the same post-migration; the native method returns the same data in the same order.
- Prior precedent: the 8.1 enum introduction required migrations from popular libraries (e.g., myclabs/php-enum).

## Migration Guide

### 1) Find affected code

```bash
# Enums that declare values()
grep -R --line-number --include='*.php' "public static function values()" .

# Traits that likely provide values()
grep -R --line-number --include='*.php' -E "trait .*EnumValues|values\(\)" .
```

### 2) Remove userland implementations

```php
// Before
enum Status: string {
    case Active = 'active';
    public static function values(): array {
        return array_map(fn($c) => $c->value, self::cases());
    }
}

// After (delete the method)
enum Status: string {
    case Active = 'active';
}
```

### 3) Verify behavior

```php
var_dump(Status::values()); // Works via native BackedEnum::values()
```

## Mitigations Considered

- Phased rollout (deprecate in 8.6, fatal in 9.0): would require engine-level checks to emit `E_DEPRECATED` when userland defines `values()`; technically feasible but adds complexity and delays consistency.
- Different method name (e.g., `getValues()`, `valueList()`): avoids breakage but diverges from established conventions and symmetry with `cases()`; likely to fragment usage.
- Accept BC break (this proposal): migration is trivial and mechanical; standardizes a ubiquitous pattern with long-term ecosystem benefits. I can prepare a rector rule for it

## Performance

- O(n) over the number of cases, similar to:

```php
array_map(fn($case) => $case->value, self::cases())
```

- Native implementation avoids userland call overhead and enables future internal optimizations if needed.

## Alternatives Considered

- Keep status quo: leaves significant, repeated boilerplate across the ecosystem.
- Traits / helpers in userland: remains fragmented and non-discoverable.
- Different naming (`toArray()`, `getValues()`): ambiguous or verbose; `values()` best matches existing community conventions and parallels `cases()`.

## Prior Art

No previous RFCs proposing a `BackedEnum::values()` method were found.
A review of the [PHP RFC index](https://wiki.php.net/rfc) shows several enum-related proposals:

- [Add get_declared_enums() function (2024)](https://wiki.php.net/rfc/get_declared_enums)
- [Fetch properties of enums in const expressions (2022)](https://wiki.php.net/rfc/fetch_property_in_const_expressions)
- [Enumerations (2020)](https://wiki.php.net/rfc/enumerations)
- [Allow static properties in enums (2021)](https://wiki.php.net/rfc/enum_allow_static_properties)
- [Auto-implement Stringable for string backed enums (2022)](https://wiki.php.net/rfc/enum_allow_static_properties)
- [Sorting enum (2021)](https://wiki.php.net/rfc/sorting_enum)

However, none of them address adding a convenience method returning the list of backing values.
This proposal introduces it for the first time.

## Checklist

- [x] Engine implementation
- [x] Arginfo generated and committed
- [x] Tests added (enum + reflection)
- [x] NEWS and UPGRADING updated
- [ ] Manual docs (php/doc-en) PR to be submitted after review

Thank you for reviewing!
