====== PHP RFC: Add values() Method to BackedEnum ======
* Version: 1.0
* Date: 2025-11-09
* Author: Savin Mikhail, mikhail.d.savin@gmail.com
* Status: Under Discussion
* Target Version: PHP 8.6
* Implementation: https://github.com/php/php-src/pull/20398
* Discussion: https://externals.io/message/129186

===== Introduction =====

This RFC proposes adding a native ''values()'' method to the ''BackedEnum'' interface that returns an indexed array of all backing values. The native implementation is added **conditionally** - only when the enum doesn't already define its own ''values()'' method, ensuring **zero backward compatibility breaks**.

<PHP>
<?php

enum Status: string {
case Active = 'active';
case Inactive = 'inactive';
case Archived = 'archived';
}

// Automatically available - no manual implementation needed:
var_dump(Status::values());
// array(3) { [0]=> string(6) "active" [1]=> string(8) "inactive" [2]=> string(8) "archived" }

?>
</PHP>

**Common use cases:**
* Database schema definitions: ''$table->enum('status', Status::values())''
* Form validation: ''$validator->rule('status', 'in', Status::values())''
* API responses: ''['allowed_statuses' => Status::values()]''

===== Proposal =====

Add a native ''values()'' static method to the ''BackedEnum'' interface that is conditionally registered based on whether the user has already defined it:

<PHP>
<?php

interface BackedEnum extends UnitEnum
{
/**
* Returns an indexed array of all backing values for the enum cases.
*
* This method is automatically available unless the enum defines its own
* values() method, in which case the user-defined implementation is used.
*
* @return int[]|string[]
*/
public static function values(): array;
}

?>
</PHP>

==== Conditional Registration ====

The native implementation is only registered when the enum **does not** already define a ''values()'' method:

<PHP>
<?php

// Case 1: No user-defined values() - native implementation added automatically
enum Status: string {
case Active = 'active';
case Inactive = 'inactive';
}

Status::values(); // Native implementation: ['active', 'inactive']

// Case 2: User-defined values() - native implementation NOT added, user's version is used
enum Priority: int {
case Low = 1;
case High = 10;

    public static function values(): array {
        // Custom implementation - maybe sorted
        $values = array_map(fn($c) => $c->value, self::cases());
        sort($values);
        return $values;
    }
}

Priority::values(); // User's implementation: [1, 10] (sorted)

?>
</PHP>

This approach ensures:
* **Zero BC breaks** - existing code with ''values()'' continues working unchanged
* **Immediate benefit** - new enums automatically get ''values()''
* **Library compatibility** - libraries can maintain their implementation for older PHP versions

==== Behavior ====

When the native implementation is used:
* **Returns:** An indexed array (keys: 0, 1, 2, ...) containing the backing values of all enum cases
* **Order:** Declaration order (same as ''cases()'')
* **Type:** ''array<int>'' for int-backed enums, ''array<string>'' for string-backed enums
* **Empty enums:** Returns ''[]''
* **Availability:** Only on ''BackedEnum'', not on ''UnitEnum'' (pure enums)

==== Examples ====

**Basic usage (native implementation):**
<PHP>
<?php

enum Priority: int {
    case Low = 1;
    case Medium = 5;
    case High = 10;
}

var_dump(Priority::values());
// array(3) { [0]=> int(1) [1]=> int(5) [2]=> int(10) }

?>
</PHP>

**Database migrations:**
<PHP>
<?php

enum OrderStatus: string {
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
    case Cancelled = 'cancelled';
}

// Laravel migration
Schema::create('orders', function (Blueprint $table) {
    $table->enum('status', OrderStatus::values());
    // ['pending', 'processing', 'completed', 'cancelled']
});

?>
</PHP>

**Form validation:**
<PHP>
<?php

enum Country: string {
    case USA = 'us';
    case Canada = 'ca';
    case Mexico = 'mx';
}

// Symfony validation
use Symfony\Component\Validator\Constraints as Assert;

class Address {
    #[Assert\Choice(callback: [Country::class, 'values'])]
    public string $countryCode;
}

// Laravel validation
$validator = Validator::make($data, [
    'country' => ['required', 'in:' . implode(',', Country::values())]
]);

?>
</PHP>

**API responses:**
<PHP>
<?php

enum Feature: string {
    case BasicPlan = 'basic';
    case ProPlan = 'pro';
    case EnterprisePlan = 'enterprise';
}

// OpenAPI / JSON Schema
return response()->json([
    'available_plans' => Feature::values(),
    // ['basic', 'pro', 'enterprise']
]);

?>
</PHP>

**User-defined implementation (respects custom behavior):**
<PHP>
<?php

enum Color: string {
    case Red = 'red';
    case Green = 'green';
    case Blue = 'blue';
    
    // Custom implementation - returns uppercase values
    public static function values(): array {
        return array_map(
            fn($c) => strtoupper($c->value),
            self::cases()
        );
    }
}

var_dump(Color::values());
// array(3) { [0]=> string(3) "RED" [1]=> string(5) "GREEN" [2]=> string(4) "BLUE" }

?>
</PHP>

**Library compatibility example:**
<PHP>
<?php

// Library code that needs to support PHP 8.4+
enum LibraryEnum: string {
    case Option1 = 'opt1';
    case Option2 = 'opt2';
    
    // Defined for PHP 8.4/8.5 compatibility
    // In PHP 8.6+, this is used instead of native (no conflict)
    public static function values(): array {
        return array_map(fn($c) => $c->value, self::cases());
    }
}

// Works in both PHP 8.4 and PHP 8.6+
var_dump(LibraryEnum::values());
// array(2) { [0]=> string(4) "opt1" [1]=> string(4) "opt2" }

?>
</PHP>

**Trait-based implementations work unchanged:**
<PHP>
<?php

trait EnumValues {
    public static function values(): array {
        return array_map(fn($c) => $c->value, self::cases());
    }
}

enum Status: string {
    use EnumValues; // User's trait takes precedence
    
    case Draft = 'draft';
    case Published = 'published';
}

// Uses trait implementation, native is not added
var_dump(Status::values());

?>
</PHP>

**Empty enum edge case:**
<PHP>
<?php

enum EmptyEnum: string {}

var_dump(EmptyEnum::values());
// array(0) { }

?>
</PHP>

===== Backward Incompatible Changes =====

**This RFC introduces ZERO backward compatibility breaks.**

The conditional registration approach ensures that:
* **Existing enums** with custom ''values()'' methods continue working unchanged
* **Trait-based** implementations are respected
* **Libraries** can maintain their implementations for older PHP version support
* **No migration** is required for any existing code

==== How It Works ====

During enum registration, the engine checks if a ''values()'' method already exists:

<code c>
// Simplified concept (actual implementation in zend_enum.c)
if (user_defined_values_exists(enum_class)) {
    // User has values() - respect their implementation
    return;
}

// No user-defined values() - register native implementation
register_native_values(enum_class);
</code>

This means:
* **PHP 8.4/8.5 code** with custom ''values()'' works identically in **PHP 8.6**
* **New enums** automatically get ''values()'' without any code
* **Gradual migration** is possible - libraries can remove their implementation when they drop PHP 8.5 support

==== Impact on Ecosystem ====

**Positive impacts:**
* ~3,860+ enums with custom ''values()'' - **continue working** (no changes needed)
* ~20,000-40,000 enums via traits - **continue working** (no changes needed)
* New enums - **automatically get** ''values()'' without boilerplate
* **Zero migration effort** for existing code

**Optional cleanup opportunity:**
Over time, projects can optionally remove redundant ''values()'' implementations when they drop support for PHP < 8.6:

<PHP>
<?php

// PHP 8.4/8.5: Need custom implementation
enum Status: string {
case Active = 'active';

    public static function values(): array {
        return array_map(fn($c) => $c->value, self::cases());
    }
}

// PHP 8.6+: Can remove (but not required)
enum Status: string {
case Active = 'active';
// Native values() available automatically
}

?>
</PHP>

==== Trade-off: API Consistency ====

This approach makes ''values()'' **the only enum method that can be user-defined**:

| Method | User-definable? |
|--------|----------------|
| ''cases()'' | ❌ No (always native) |
| ''from()'' | ❌ No (always native) |
| ''tryFrom()'' | ❌ No (always native) |
| ''values()'' | ✅ Yes (conditional) |

**Rationale for this trade-off:**
* Avoids **all** backward compatibility breaks
* Provides **immediate benefit** without ecosystem disruption
* Allows **gradual adoption** - libraries can migrate at their own pace
* **Practical** over theoretical consistency - solves real problem (24k-44k instances of boilerplate) without forcing changes

**Future path to full consistency:**
If desired, a future RFC could:
1. Emit ''E_DEPRECATED'' for user-defined ''values()'' in PHP 8.x
2. Make it non-overridable in PHP 9.0
3. Achieve full consistency with other enum methods

This RFC deliberately prioritizes **pragmatic value delivery** over **perfect API consistency**.

===== Proposed PHP Version(s) =====

Next PHP 8.x (PHP 8.6)

This is a feature addition with zero BC breaks, appropriate for a minor version.

===== RFC Impact =====

==== To the Ecosystem ====

**Positive impacts:**
* **IDEs/LSPs:** Native method appears in autocomplete for enums without custom ''values()''
* **Static Analyzers:** Can infer ''values()'' availability more reliably
* **Frameworks:** No changes needed, but can gradually simplify helpers/traits
* **Documentation:** Single standard approach simplifies teaching
* **Libraries:** No forced migrations, can maintain compatibility across PHP versions

**No negative impacts:**
* Zero breaking changes
* All existing code continues working
* Optional cleanup can happen gradually

==== To Existing Extensions ====

No impact to existing extensions. This is a core enum feature with no extension dependencies.

==== To SAPIs ====

No impact. This is a language-level feature that behaves identically across all SAPIs (CLI, FPM, embedded, etc.).

===== Open Issues =====

None. The conditional approach addresses the BC concerns raised during initial discussion.

===== Future Scope =====

==== Optional: Future Convergence on Mandatory values() ====

If the community later prefers full consistency (non-overridable ''values()'' like other enum methods):

**Phase 1 (PHP 8.x):** Emit ''E_DEPRECATED'' for user-defined ''values()''
<PHP>
<?php
enum Status: string {
    case Active = 'active';
    
    public static function values(): array { ... }
    // Deprecated: Status::values() is provided natively and should not be redeclared
}
?>
</PHP>

**Phase 2 (PHP 9.0):** Make user-defined ''values()'' an error

This gives the ecosystem years to migrate while eventually achieving full API consistency.

**This future scope is NOT part of the current RFC** - just documenting the possibility.

==== Potential Enhancement: Array Keys ====

Future RFC could add optional parameter to control array keys:

<PHP>
<?php
// Hypothetical future enhancement (NOT part of this RFC)
Status::values(preserveKeys: true);  // ['Active' => 'active', 'Inactive' => 'inactive']
Status::values(preserveKeys: false); // ['active', 'inactive'] (default)
?>
</PHP>

This RFC deliberately keeps the API simple with indexed arrays matching ''cases()'' behavior.

==== Additional Helper Methods ====

Future RFCs could add related methods:
* ''names()'': Return array of case names
* ''toArray()'': Return associative array of name => value pairs

These are intentionally excluded from this RFC to keep scope focused.

===== Voting Choices =====

This is a simple yes/no vote requiring 2/3 majority as it's a language feature addition.

Vote will open 2 weeks after RFC announcement and remain open for 2 weeks.

<doodle title="Add BackedEnum::values() method with conditional registration as described in this RFC?" voteType="single" closed="true" closeon="2025-12-07T00:00:00Z">
   * Yes
   * No
</doodle>

===== Patches and Tests =====

**Pull Request:** https://github.com/php/php-src/pull/20398

**Implementation includes:**
* Core implementation in ''Zend/zend_enum.c'' with conditional registration logic
* Stub files updated (''zend_enum.stub.php'')
* Comprehensive test coverage (9+ test files) including:
    * Native implementation (when user doesn't define ''values()'')
    * User-defined implementation (when user defines ''values()'')
    * Trait-based implementation
    * Reflection behavior for both cases
    * Edge cases (empty enums, order preservation, etc.)
* Documentation in NEWS and UPGRADING

**Key implementation detail:**

The conditional check happens during enum registration:

<code c>
// In zend_enum_register_funcs() - simplified
zend_function *existing = zend_hash_str_find_ptr(
    &ce->function_table, "values", sizeof("values")-1
);

if (existing && existing->common.scope == ce) {
// User defined values() on this enum - respect it
return;
}

// No user-defined values() - register native implementation
zend_internal_function *values_function = ...;
zend_enum_register_func(ce, ZEND_STR_VALUES, values_function);
</code>

All tests pass. Implementation is ready for merge pending RFC approval.

===== Implementation =====

After the RFC is approved, this section will contain:
* Version merged into: PHP 8.6.0
* Git commit: (link will be added)
* PHP manual entry: (link will be added)

===== References =====

**Research and Evidence:**
* GitHub code search: ~3,860 direct implementations found
* Estimated real usage: ~20,000-40,000 (accounting for trait pattern)
* Symfony core usage: ''symfony/symfony/src/Symfony/Component/TypeInfo/TypeIdentifier.php''
* PHP.net manual: Documents ''EnumValuesTrait'' pattern
* Internals discussion: (link will be added after initial email)

**Search queries performed:**
* https://github.com/search?q=language:PHP+"array_map(fn($case)+=>+$case->value,+self::cases())"
* https://github.com/search?q=language:PHP+"return+array_map"+"self::cases()"+"->value"
* https://github.com/search?q=language:PHP+"function+values()"+"return+array_map"+"self::cases()"

**Prior Art:**
* **TypeScript:** ''Object.values(EnumType)''
* **Python:** ''[e.value for e in EnumType]''
* **myclabs/php-enum** (legacy): Had ''values()'' method (4,900 stars)

**Related RFCs:**
* PHP 8.1 Enumerations: https://wiki.php.net/rfc/enumerations
* No previous RFC for ''values()'' method

===== Rejected Features =====

==== Mandatory Native Implementation (Breaking Change) ====

Initially considered making ''values()'' always native and non-overridable (like ''cases()''/''from()''/''tryFrom()'').

**Rejected because:**
* Would break ~24,000-44,000 existing enum instances
* Disproportionate impact for the benefit provided
* Libraries particularly affected (cannot easily drop old PHP version support)
* Internals feedback indicated BC break too large

**Conditional approach chosen instead:** Provides benefit without breaking changes.

==== Alternative Method Names ====

**''getValues()''**: More verbose, doesn't match ''cases()'' style
**''toArray()''**: Ambiguous - case objects or values? Names or values?
**''valueList()''**: Unnecessarily verbose
**''extractValues()''**: Too long, unclear

**Decision:** ''values()'' best matches:
* Existing community usage (3,860+ examples use this name)
* Parallel naming with ''cases()''
* Simplicity and clarity

==== Virtual/Magic Properties ====

Suggestion to use ''Status::$values'' instead of ''Status::values()''.

**Rejected because:**
* Enums cannot have static properties (language restriction)
* No mechanism for static virtual properties exists
* Inconsistent with ''cases()'', ''from()'', ''tryFrom()'' (all methods)
* Would require complex engine changes

==== User-land Trait in Standard Library ====

Suggestion to provide standard library trait instead of native method.

**Rejected because:**
* Requires ''use'' statement in every enum (boilerplate persists)
* Not automatically available (discoverability issue)
* Fragmentation - multiple competing trait implementations exist
* Conditional native approach provides better UX

==== array_column() Alternative ====

Suggestion that ''array_column(Status::cases(), 'value')'' is sufficient.

**Rejected because:**
* Not discoverable for newcomers
* Less readable than dedicated method
* Doesn't solve standardization problem
* Evidence shows community strongly prefers explicit method (3,860+ implementations)

==== Different Signature ====

Considered allowing customization: ''values(sorted: true)'' or ''values(unique: true)''.

**Rejected because:**
* Adds complexity for rare use cases
* Users can easily pipe through ''array_unique()'' or ''sort()'' if needed
* Keeps API consistent with simple ''cases()''

<PHP>
<?php
// Users can customize as needed
$sorted = Status::values();
sort($sorted);

$unique = array_unique(Status::values());
?>
</PHP>

==== Deprecation Period for User-Defined values() ====

Considered emitting ''E_DEPRECATED'' for user-defined ''values()'' immediately.

**Rejected because:**
* Unnecessary - the conditional approach works well
* Would create noise without benefit
* Can be revisited in future if full API consistency is desired

===== Changelog =====

* **2025-11-09:** Initial RFC published with conditional implementation approach
* **2025-11-09:** Announced on internals@lists.php.net
* (Future updates will be listed here)