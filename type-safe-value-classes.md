# Implementation Plan: Type-Safe Value Classes

## Summary

Implement four type-safe value classes (`Currency`, `MonetaryAmount`, `Percentage`, `ProfitLoss`) as `data class` with `@Parcelize` annotation in the `core:model` module. These classes provide high-precision financial calculations using `BigDecimal`, type safety to prevent mixing incompatible values, and Parcelable support for Android state preservation. The implementation follows established patterns from `ResourceText` and `PortfolioUi` in the codebase.

## Technical Context

**Language/Version**: Kotlin (per gradle/libs.versions.toml)  
**Primary Dependencies**: Jetpack Compose, Hilt, Retrofit, Room, Coroutines  
**Storage**: Room for local persistence, DataStore for preferences  
**Testing**: JUnit, Kover for coverage measurement  
**Target Platform**: Android (minSdk per app/build.gradle.kts)
**Project Type**: Mobile - modular Android architecture  
**Performance Goals**: 60 fps UI, responsive trading operations  
**Constraints**: Diff-based 40%+ test coverage, Detekt maxIssues=0  
**Scale/Scope**: Multi-module financial trading application

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Compliance | Notes |
|-----------|------------|-------|
| I. Modular Architecture | ✅ | Implementation in `core:model` module, shared by all feature modules |
| II. Clean Architecture & MVVM | ✅ | Pure domain layer types, no dependencies on UI/data layers |
| III. Code Quality Gates | ✅ | Will include tests, follows coding standards |
| IV. Test Coverage Standards | ✅ | Unit tests for all operations, targeting 80%+ coverage |
| V. Security & Compliance | ✅ | No sensitive data handling, pure value types |
| VI. Feature Flag Requirement | ⚪ N/A | Infrastructure types exempt - no user-facing behavior, no API calls, no side effects |

**Constitution VI Exemption Rationale**: 
This feature implements passive library types that:
- Have no runtime behavior when unused
- Cause no side effects on existing features
- Require explicit adoption by consuming code
- Are pure data types with no API calls or user interactions

The types only become active when feature code explicitly uses them, at which point the consuming feature's flag gates the usage.

## Project Structure

### Documentation (this feature)

```text
specs/002-typesafe-value-classes/
├── spec.md              # Feature specification
├── plan.md              # This file - implementation plan
├── research.md          # Research findings and decisions
├── data-model.md        # Entity definitions
├── quickstart.md        # Developer usage guide
├── contracts/
│   └── api.md           # Public API specification
├── checklists/
│   └── requirements.md  # Requirements checklist
└── tasks.md             # Task breakdown (Phase 2 - /speckit.tasks)
```

### Source Code (implementation target)

```text
core/model/
├── build.gradle.kts                    # Add kotlin-parcelize plugin
└── src/main/java/com/derayah/core/model/
    ├── currency/
    │   └── Currency.kt                 # Currency value class
    ├── money/
    │   ├── MonetaryAmount.kt           # Monetary value class
    │   └── BigDecimalParceler.kt       # Custom Parceler for BigDecimal
    ├── percentage/
    │   └── Percentage.kt               # Percentage value class
    └── profitloss/
        └── ProfitLoss.kt               # ProfitLoss value class

core/model/
└── src/test/java/com/derayah/core/model/
    ├── currency/
    │   └── CurrencyTest.kt
    ├── money/
    │   └── MonetaryAmountTest.kt
    ├── percentage/
    │   └── PercentageTest.kt
    └── profitloss/
        └── ProfitLossTest.kt
```

## Complexity Tracking

> **No violations identified.** All design decisions align with constitution principles.

---

## Design Decisions

### DD-01: Data Class with @Parcelize over Value Class

**Decision**: Use `data class` with `@Parcelize` annotation instead of Kotlin `@JvmInline value class`.

**Rationale**:
- `@JvmInline value class` has restrictions that complicate Parcelable implementation
- `data class` pattern is established in codebase (ResourceText, PortfolioUi, CustomerStrategy)
- Automatic `equals()`, `hashCode()`, `copy()`, `toString()` implementations
- Minor memory overhead is acceptable for type safety benefits

### DD-02: BigDecimal for Internal Representation

**Decision**: Use `java.math.BigDecimal` for all numeric values.

**Rationale**:
- Arbitrary precision arithmetic prevents floating-point errors
- Already used extensively throughout the codebase for financial calculations
- Native serialization support with existing Moshi/Gson converters

### DD-03: Optional Currency in MonetaryAmount

**Decision**: `currency` field is nullable in `MonetaryAmount`.

**Rationale**:
- Allows flexibility when currency is implicit (all values known to be SAR)
- Simplifies arithmetic between amounts without currency matching validation
- Follows existing patterns where currency context is often external

### DD-04: Percentage Stores Actual Value

**Decision**: Store 5.0 for 5%, not 0.05.

**Rationale**:
- More intuitive for developers: `Percentage(5.0)` means 5%
- Matches how users think about percentages
- Conversion to decimal (0.05) happens only during calculations via `asDecimal` property

### DD-05: ProfitLoss Uses Composition

**Decision**: `ProfitLoss` wraps `MonetaryAmount` internally via composition.

**Rationale**:
- Promotes code reuse for arithmetic and formatting
- Ensures consistent precision handling
- Semantic properties derived from underlying value

### DD-06: ArithmeticException for Division by Zero

**Decision**: Throw `ArithmeticException` on division by zero.

**Rationale**:
- Follows standard Kotlin/Java conventions
- Consistent with `BigDecimal` behavior
- Programmer errors should be caught during development

---

## Implementation Approach

### Phase 1: Core Module Setup

1. **Update `core/model/build.gradle.kts`**:
   ```kotlin
   plugins {
       id("derayah.library")
       id("derayah.core")
       id("kotlin-parcelize")  // Add this
   }
   ```

2. **Create package structure**:
   ```
   com.derayah.core.model.currency
   com.derayah.core.model.money
   com.derayah.core.model.percentage
   com.derayah.core.model.profitloss
   ```

### Phase 2: Implement BigDecimalParceler

```kotlin
// com.derayah.core.model.money.BigDecimalParceler
object BigDecimalParceler : Parceler<BigDecimal> {
    override fun create(parcel: Parcel): BigDecimal = 
        BigDecimal(parcel.readString()!!)
    
    override fun BigDecimal.write(parcel: Parcel, flags: Int) = 
        parcel.writeString(this.toPlainString())
}
```

### Phase 3: Implement Currency

- Validation in `init` block for ISO 4217 codes
- Predefined companion constants (SAR, USD, EUR, GBP, AED)
- Symbol and display name mapping

### Phase 4: Implement MonetaryAmount

- Arithmetic operators: `plus`, `minus`, `times`, `div`, `unaryMinus`
- Comparison via `Comparable<MonetaryAmount>`
- Formatting methods consistent with `NumberExt.kt` patterns
- **Backend factory**: `fromDouble(value: Double, currency: Currency? = null)` using `BigDecimal.valueOf()` for safe conversion

### Phase 5: Implement Percentage

- `asDecimal` property for calculation conversion
- `of(amount)` method to apply percentage
- **Backend factories**:
  - `fromDecimal(decimal: Double)` - converts 0.05 → 5%
  - `fromDecimal(decimal: BigDecimal)` - converts 0.05 → 5%
  - `of(value: Double)` - creates from actual percentage (5.0 = 5%)

### Phase 6: Implement ProfitLoss

- Semantic properties: `isProfit`, `isLoss`, `isBreakEven`
- Delegate arithmetic to underlying `MonetaryAmount`
- Factory methods: `profit()`, `loss()`, `breakEven()`

### Phase 7: Unit Tests

- Test all arithmetic operations for precision
- Test Parcelable serialization/deserialization
- Test edge cases (zero, negative, large values)
- Test formatting output

---

## Dependencies

| Dependency | Version | Status | Purpose |
|------------|---------|--------|---------|
| kotlin-parcelize | (bundled) | Existing | @Parcelize annotation |
| kotlin-stdlib | 2.1.21 | Existing | BigDecimal, standard library |
| JUnit | 5.10.1 | Existing | Unit testing |

**No new external dependencies required.**

---

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Performance vs primitives | Low | Low | Profile if issues; BigDecimal already used extensively |
| Serialization compatibility | Low | Medium | Use toPlainString(); comprehensive tests |
| Breaking existing code | None | N/A | Additive change; no existing code affected |
| Adoption resistance | Low | Low | Provide quickstart guide; gradual migration |

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Test coverage | 80%+ | Kover report |
| Compilation | Zero errors | CI build |
| Static analysis | Detekt pass | Zero issues |
| Documentation | Complete | All public APIs documented |

---

## References

- [spec.md](./spec.md) - Feature specification
- [research.md](./research.md) - Research findings
- [data-model.md](./data-model.md) - Entity definitions
- [contracts/api.md](./contracts/api.md) - Public API specification
- [quickstart.md](./quickstart.md) - Developer usage guide
