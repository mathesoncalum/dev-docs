# 00001 Engraving: LD_ACCESS

Date: 2023-09-08   
Tags: engraving, layout   
Maintainers: Igor Korsukov   

## Status: Accepted

## Context

In engraving, an element has two kinds of data:

* Properties of the element itself (parent, text, symbol, and so on).
* Data calculated during layout (position, geometry, and so on).

These were separated: the data calculated during layout lives in its own structure, `LayoutData`.

During layout, some elements read the layout data of other elements. For example, the width of a container depends on the widths of the elements it contains. In practice the dependencies are more involved than that.

The problem is that layout code often reads data that has not been calculated yet — the layout of that element has not run. That usually means one of two things:

* A flaw in the algorithm: the read happens in the wrong order.
* An inefficient implementation: the same data is read more than once. The first read happens while the value is still empty, and a later read is what makes the result look correct.

Both should be fixed. Fixing them reduces layout bugs and removes redundant work from layout.

## Decision

Every field of `LayoutData` is stored in an `ld_field` wrapper. The wrapper knows whether the value has been calculated.

In a debug build, the CMake option `MUE_ENABLE_ENGRAVING_LD_ACCESS` turns on detection of a read that happens before the value has been calculated.

A read declares how it should be treated:

* `CHECK` — the value is expected to be calculated already. If it is not, the access is reported.
* `BAD` — the access is known to be wrong. It is not reported, and it will be fixed later.
* `MAYBE_NOTINITED` — reading the field before it has been calculated is acceptable here.
* `PASS` — used while debugging.

The default is `CHECK`. Call sites that are already known to be wrong are marked explicitly, so they can be fixed one by one and new mistakes stay visible.

## Consequences

* A read of layout data before it has been calculated shows up during development, as a diagnostic, before it shows up as a wrong engraving result.
* Such a read is either a logic error or a redundant read. Both become visible, so layout gets fewer bugs and less repeated work.
* Places that are already known to be wrong are marked `BAD` and stay on a list to fix, and the log stays usable.
* Places where an early read is legitimate are marked `MAYBE_NOTINITED`, so the check stays meaningful.
* Every layout field goes through `ld_field`, and a call site that is not yet correct names its access mode.
* The check runs only in a debug build with the option enabled. A release build keeps a plain field and does not pay for the check.

## Alternatives

Leaving layout fields as plain values. Nothing then distinguishes "not calculated yet" from a real zero, `false`, or empty geometry, so both failure modes stay invisible.

Reporting every uninitialized read, with no `BAD` mode. That is the end state we want, but the existing call sites would make a debug build unusable until all of them are fixed. Marking them `BAD` lets the check be enabled now, and the backlog can be burned down.

## Implementation

`LayoutData` for an element. Each calculated field is an `ld_field`, constructed with a diagnostic name and a default:

```cpp
struct LayoutData : public EngravingItem::LayoutData
{
    ld_field<bool> up = { "[Articulation] up", true };
    ld_field<SymId> symId = { "[Articulation] symId", SymId::noSym };
};
DECLARE_LAYOUTDATA_METHODS(Articulation)
```

Access modes:

```cpp
//! NOTE Enum for detect layoutdata bad access errors
enum class LD_ACCESS : unsigned char {
    CHECK = 0,          // should be correct; assert if still not
    BAD,                // known to be bad; don't assert, we'll fix it later
    MAYBE_NOTINITED,    // in this case it's okay if we access it before it's been inited
    PASS                // for debug
};
```

The diagnostic is off by default. Enable it for a debug build:

```
-DMUE_ENABLE_ENGRAVING_LD_ACCESS=ON
```

In a debug build, `ld_field` stores the value in `std::optional`. A read through `value()` uses `CHECK` unless a mode is passed. When the option is on and the value has not been set, a `CHECK` read logs `BAD ACCESS` together with the field name. Assigning the field, or calling `set_value` / `mut_value`, marks it as calculated. `reset()` clears it for the next layout pass.

In a release build (`NDEBUG`), `ld_field` is a plain value: `has_value()` is always true, and `value()` does not check the mode.

A call site that is not yet correct passes the mode explicitly:

```cpp
note->ldata()->mirror.value(LD_ACCESS::BAD);
item->shape(LD_ACCESS::MAYBE_NOTINITED);
```
