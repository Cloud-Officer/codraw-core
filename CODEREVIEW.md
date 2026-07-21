# Code Review — codraw/core

**Package:** `codraw/core` (namespace `Draw\Component\Core`)
**Reviewed:** all PHP sources, `composer.json`, PHPStan/PHPUnit configuration, tests (skimmed for coverage).

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **M5** — `composer.json`: added `"symfony/service-contracts": "^2.5 || ^3.0"` to `require` (`Evaluator` type-hints `Symfony\Contracts\Service\ServiceProviderInterface` in its public constructor). Constraint mirrors what `symfony/dependency-injection` 6.4 itself allows, so it cannot conflict with consumers. The `symfony/dependency-injection` requirement was left in place (its `ServiceLocator` is still hard-used as the default provider).
- **H1** — `FilterExpression/Query.php`: `getExpression()` return type corrected to `?Expression`; `FilterExpression/Evaluator.php`: `execute()` now yields all rows when the query has no expression instead of throwing a `TypeError`. Only the previously-fatal path changes behavior.
- **H2** — `Reflection/ReflectionAccessor.php`: static-property write now uses `$property->setValue(null, $value)` (drop-in replacement for the PHP 8.3+ deprecated single-argument form).
- **M1** — `DynamicArrayObject.php`: the `null`-input guard now runs before the `parent::__construct()` call (substituting `[]`), so `new DynamicArrayObject(null)` produces an empty object instead of a `TypeError`.
- **M4** — `FilterExpression/Expression/Expression.php`: `validate()` constraint parameter widened from `?EqualTo` to `Constraint|array|null`, matching what `ConstraintExpression`/`ValidatorInterface::validate()` accept. Existing callers are unaffected; previously-`TypeError`ing calls now work.
- **L6** — `FilterExpression/Expression/ConstraintExpressionEvaluator.php`: constructor now accepts `?PropertyAccessorInterface` instead of the concrete `?PropertyAccessor` (the property was already interface-typed; constructors are exempt from LSP signature checks).
- **L4 (partial)** — `DataAccessor.php`: stale copy-paste docblocks ("asserts", "currently tested", "Tester instance") corrected. The `static` vs `self` native return types and untyped `$path` parameters were left as-is (signature changes, out of quick-win scope).
- **L5 (partial)** — `README.md`: removed the "Ignore Annotations" section documenting a feature/file that does not exist in the package. Still open: `>=8.5` vs `^8.5` PHP constraint (repo convention is `>=8.5`, kept), missing `autoload-dev`/`exclude-from-classmap` split for `Tests/`, and the unrelated `.phpstorm.meta.php` content.

Verification: `php -l` clean on all modified files; `composer validate --no-check-publish` passes; no `vendor/` present so the PHPUnit suite was not run (a dependency-free smoke script exercised the H2, M1 and recursive-copy paths successfully).

### Validation pass (2026-07-20, same day)

- `composer install --optimize-autoloader --no-interaction --prefer-dist --no-scripts`: OK (lock file resolves with the added `symfony/service-contracts` requirement and the `^8.5` PHP constraint; no adjustment needed).
- PHPUnit 12.5.31 (PHP 8.5.8): **54 tests, 100 assertions, 0 failures/errors** — no test-expectation updates were required by the fixes. The H2 fix removed the 1 PHP deprecation the unmodified tree reports. The 7 remaining PHPUnit notices ("no expectations configured for mock object" in `Tests/FilterExpression/QueryTest.php` and `Tests/FilterExpression/Expression/CompositeExpressionEvaluatorTest.php`) are pre-existing PHPUnit 12 strictness notices, confirmed identical via `git stash` on the unmodified tree; left untouched.
- PHPStan (`phpstan.dist.neon`, empty baseline): **no errors** — no baseline drift from the fixes.
- markdownlint-cli2: **0 errors** across all 6 Markdown files.
- No additional findings were fixed in this pass: the remaining open items (M2, M3, L1, L2, L3, L7, rest of L4/L5) are not directly pinned by existing unit tests, so changing them was out of scope.

---

## Overall Assessment

This is a small, focused utility package (date/time helpers, reflection helpers, a validator-backed filter-expression engine, a dynamic array object, and a property-access wrapper). The code is modern (constructor promotion, `match`, first-class callables), classes are small and single-purpose, static analysis runs at PHPStan level 5 with an empty baseline, and most of the public surface is covered by well-structured data-provider tests. There are no security-sensitive code paths (no I/O, no deserialization, no shell/SQL/network access), so the risk profile is low. The issues found are correctness bugs in edge cases, a deprecated Reflection API usage that fires on every static-property write under the required PHP version, misleading API semantics (`millisecondDiff` with second precision), and a few dependency/metadata hygiene problems. Grade: **B**.

---

## Findings

### High

#### **[FIXED]** H1. `Query::getExpression()` violates its return type and crashes `Evaluator::execute()` for an empty query

`FilterExpression/Query.php:40-43` declares `getExpression(): Expression` but returns the property `?Expression $expression` (line 9), which is `null` until `where()`/`andWhere()`/`orWhere()` is called. Executing a freshly-created `Query` via `Evaluator::execute()` (`FilterExpression/Evaluator.php:29`) throws a `TypeError` ("Return value must be of type Expression, null returned") instead of either returning all rows or raising a meaningful domain exception. Either the return type should be `?Expression` with a null-check in `Evaluator::execute()`, or `getExpression()` should throw a descriptive `LogicException`. (PHPStan level 5 should flag this; the baseline is empty, suggesting analysis is not run consistently.)

#### **[FIXED]** H2. Deprecated `ReflectionProperty::setValue()` single-argument call for static properties

`Reflection/ReflectionAccessor.php:30` calls `$property->setValue($value)` for static properties. Since PHP 8.3, calling `ReflectionProperty::setValue()` with a single argument is deprecated (you must pass `null` as the first argument: `$property->setValue(null, $value)`). The package requires PHP `>=8.5` (`composer.json:7`), so every static-property write through `ReflectionAccessor::setPropertyValue()` emits a deprecation notice today and will break on a future PHP major. The package's own test suite exercises this path (`Tests/Reflection/ReflectionAccessorTest.php`, `testSetPropertyValueStatic`) but deprecations are silenced by `SYMFONY_DEPRECATIONS_HELPER=weak` and `error_reporting=-1` only surfaces it as a notice.

### Medium

#### **[FIXED]** M1. `DynamicArrayObject` constructor's `null` handling is dead code — passing `null` throws a `TypeError`

`DynamicArrayObject.php:7-13`: the constructor calls `parent::__construct($input, ...)` **before** the `if (null === $input) return;` check. `ArrayObject::__construct()` is typed `array|object`, so `new DynamicArrayObject(null)` throws a `TypeError` on line 9 and the null-guard on line 11 is unreachable. Either the guard should run before the parent call (substituting `[]`), or the parameter should be typed `array|object` and the guard removed.

#### M2. `DateTimeUtils::millisecondDiff()` has only second-level precision despite its name

`DateTimeUtils.php:36-43` computes `($dateTime->getTimestamp() - $compareTo->getTimestamp()) * 1000`. `getTimestamp()` truncates to whole seconds, so two instants 900 ms apart report a diff of `0`. A method named `millisecondDiff` strongly implies millisecond accuracy; use `format('Uv')` or `(float) format('U.u')` arithmetic, or rename/document the truncation. Any caller using this for timing/latency measurements gets systematically wrong values.

#### M3. `toDateTimeImmutable()` / `toDateTime()` silently discard microseconds and timezone

`DateTimeUtils.php:20-34` round-trips through `createFromFormat('U', (string) $timestamp)`. The result always has UTC (`+00:00`) timezone and `.000000` microseconds, regardless of the input's timezone or sub-second precision. For a general-purpose "convert between mutable/immutable" utility this is lossy and surprising — the native `\DateTimeImmutable::createFromInterface()` / `\DateTime::createFromInterface()` (PHP 8.0+) preserve both and would be strictly better here. Additionally, `createFromFormat()` returns `false` on failure, which would produce a `TypeError` against the declared return types.

#### **[FIXED]** M4. `Expression::validate()` over-narrows the constraint type to `EqualTo`

`FilterExpression/Expression/Expression.php:22` types the parameter `?EqualTo $constraints`, but the underlying `ConstraintExpression` (`ConstraintExpression.php:20-24`) accepts `Constraint|Constraint[]|null` and the evaluator passes it straight to `ValidatorInterface::validate()`. The static factory therefore arbitrarily prevents using any other constraint (`NotBlank`, `Choice`, `Range`, callbacks, arrays of constraints), forcing users to bypass the factory. The type should be `Constraint|array|null`.

#### **[FIXED]** M5. Missing direct dependency on `symfony/service-contracts`

`FilterExpression/Evaluator.php:10,14` type-hints `Symfony\Contracts\Service\ServiceProviderInterface`, which lives in `symfony/service-contracts`. `composer.json` only requires `symfony/dependency-injection` (which pulls contracts in transitively today). Relying on a transitive dependency for a public constructor signature is fragile; declare `symfony/service-contracts` explicitly. Relatedly, `symfony/dependency-injection` is a heavyweight requirement used solely for the `ServiceLocator` fallback in the same constructor.

### Low

#### L1. `DynamicArrayObject::offsetExists()` unconditionally returns `true`

`DynamicArrayObject.php:38-41`: `isset($obj['missing'])` and `isset($obj->missing)` always report `true`, and `offsetGet()` (lines 43-50) auto-vivifies missing keys on **read**, mutating the object (`count()` changes after a read). This is presumably intentional for the "dynamic" use case, but it breaks normal ArrayAccess contracts and deserves prominent class-level documentation; there is currently none, and the class has no tests pinning this behavior.

#### L2. `ReflectionAccessor::callMethod()` with a class-string and a non-static method fails confusingly

`Reflection/ReflectionAccessor.php:7-14`: when `$objectOrClass` is a class name and the resolved method is not static, `$object` stays the class-string... actually `$object` is set to the class-string only when non-static (`$methodReflection->isStatic() ? null : $objectOrClass`), so `invoke('ClassName', ...)` throws a low-level `TypeError`/`ReflectionException` instead of a clear message. A guard with an explicit exception would make misuse obvious.

#### L3. Exception-by-side-effect pattern in `ReflectionAccessor`

`Reflection/ReflectionAccessor.php:56-58` and `74-76`: `new \ReflectionMethod($class, $methodName);` / `new \ReflectionProperty(...)` are instantiated purely so their constructors throw. If Reflection ever stopped throwing there, the surrounding `while (true)` loops become infinite. `throw new \ReflectionException(...)` would be explicit and safe.

#### **[PARTIALLY FIXED]** L4. Stale copy-paste documentation in `DataAccessor`

`DataAccessor.php:29-31,37,58-59`: docblocks refer to "asserts", "what is currently tested", and "a new Tester instance" — clearly copied from a test-assertion class in another package. Also, `transform()`/`path()` are annotated `@return static` but typed `: self`; with a `final` constructor this works, but the native return type should be `static`. `getData($path = null)` and `path($path)`/`isReadable($path)` have untyped parameters where `string|PropertyPathInterface|null` unions are expressible in PHP 8.

#### **[PARTIALLY FIXED]** L5. Dependency and metadata hygiene

- `composer.json:7`: `"php": ">=8.5"` is open-ended (allows PHP 9.x with breaking changes); prefer `^8.5` — and consider whether 8.5 is genuinely required, since no 8.4/8.5-only features are used and this constraint blocks the large PHP 8.1-8.4 install base from an otherwise compatible library.
- No `autoload-dev`: `Tests/` is autoloadable in production via the root PSR-4 mapping `"Draw\\Component\\Core\\": ""`. Adding `exclude-from-classmap` for `Tests/` (or an `autoload-dev` split) keeps test stubs (`StubClass`, `StubTrait`) out of production classmaps.
- `README.md:7-16` documents an "Ignore Annotations" feature and links `ignore_annotations.php`, but neither the file nor any annotation-related code exists in the package.
- `.phpstorm.meta.php:1-13` contains overrides for `Symfony\Component\Security\...\Passport`, unrelated to anything in this package.

#### **[FIXED]** L6. `ConstraintExpressionEvaluator` constructor hints the concrete `PropertyAccessor`

`FilterExpression/Expression/ConstraintExpressionEvaluator.php:17` accepts `?PropertyAccessor` (concrete class) while the property is typed `PropertyAccessorInterface` (line 13). Accepting the interface would allow decorated/custom accessors.

#### L7. Evaluator/Expression coupling via naming convention

`FilterExpression/Expression/Expression.php:40-43`: `evaluateBy()` returns `static::class.'Evaluator'`, so custom expressions must have an evaluator class of exactly that FQCN registered in the service provider. Additionally, if a caller supplies a custom `ServiceProviderInterface` to `Evaluator::__construct()` (`Evaluator.php:16-22`) it fully replaces the defaults, so built-in `ConstraintExpression`/`CompositeExpression` silently stop resolving unless the caller re-registers both. Worth documenting; a decorating/fallback lookup would be more robust.

---

## Strengths

- **Small, cohesive API surface.** Each class does one thing; utility classes (`DateTimeUtils`, `ReflectionAccessor`) are `final` with private constructors where appropriate.
- **Modern, clean PHP.** `match` expressions, constructor property promotion, first-class callable syntax (`static::andX(...)`), strict `in_array(..., true)` checks.
- **Lazy evaluation done right.** `Evaluator::execute()` (`FilterExpression/Evaluator.php:27-35`) yields rows through a generator, so filtering large iterables is memory-safe, and `CompositeExpressionEvaluator` short-circuits AND/OR correctly (verified against the truth table, including the empty-expression-list `true` case).
- **Correct trait resolution.** `use_trait()` (`functions.php`) correctly walks the parent-class chain and recurses into traits-of-traits, and is idempotently guarded with `function_exists` for the `replace: draw/core` scenario.
- **Reflection walks parents for private members.** `ReflectionAccessor` correctly resolves private methods/properties declared on ancestors, which naive `new \ReflectionMethod()` usage misses.
- **Static-analysis discipline.** PHPStan level 5 across the whole package with an **empty** baseline file.
- **No security-sensitive surface.** No file/network/process/deserialization operations anywhere in the package.

---

## Test Coverage

Overall coverage is good for the package's size (10 test classes, ~730 lines of tests), with idiomatic data providers and named cases.

**Well covered:**

- `DateTimeUtils` — all four methods, including null handling, mixed mutable/immutable comparisons, and diff direction (`Tests/DateTimeUtilsTest.php`).
- `use_trait()` — direct use, parent-class use, non-existent trait (`Tests/FunctionTest.php`).
- FilterExpression — end-to-end `Evaluator::execute()` with AND/OR/no-match/missing-path scenarios (`Tests/FilterExpression/EvaluatorTest.php`), `Query` builder semantics (`QueryTest.php`), factory methods (`ExpressionTest.php`), and evaluator error paths (`CompositeExpressionEvaluatorTest.php`, `ConstraintExpressionEvaluatorTest.php`).
- `ReflectionAccessor` — instance/static properties and methods, including inherited-private cases (`Tests/Reflection/ReflectionAccessorTest.php`).

**Gaps:**

- `DynamicArrayObject` — **no tests at all**, despite having the most surprising semantics in the package (always-true `offsetExists`, auto-vivification on read, recursive `getArrayCopy`). The `null`-input `TypeError` (M1) would have been caught by a single constructor test.
- `DataAccessor` — **no tests** (`getData`, `path`, `transform`, `isReadable` untested).
- `ReflectionExtractor` — **no tests** (named/union/intersection/null type branches untested).
- Error paths: empty `Query` through `Evaluator::execute()` (finding H1), `ConstraintExpressionEvaluator` happy path is only covered indirectly via `EvaluatorTest`.
- Deprecation visibility: `SYMFONY_DEPRECATIONS_HELPER=weak` in `phpunit.xml.dist` hides the PHP 8.3+ Reflection deprecation (H2) from the test run.
