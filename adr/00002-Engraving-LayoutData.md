# 00002 Engraving: LayoutData

Date: 2023-09-08   
Tags: engraving, layout   
Maintainers: Igor Korsukov   

## Status: Accepted

## Context

Every engraving element carries two kinds of data.

* Properties of the element itself: parent, text, the symbol the user chose, a placement preference.
* Data calculated during layout: position, geometry, and values that depend on other elements.

These used to live side by side on the element. Looking at a field, it was unclear which kind it was.

A value can also be honestly either. The element can store a preference, such as a mode or "prefers up", and layout can resolve that preference to a concrete value once it sees the neighbours. Writing the resolved value back as a property mixes the user's choice with every context layout discovers. Each new piece of context is another combination that has to be stored on the element and kept in agreement with the others.

## Decision

Properties stay on the element. Data calculated during layout lives in `LayoutData`. This is the rule for every element.

The element owns the structure, and layout fills it. `EngravingItem::LayoutData` holds what every element receives from layout: position, shape, and mask. An element that calculates something of its own subclasses `LayoutData` and adds those fields.

When a value could be stored either way, the property stores the request and `LayoutData` stores the result:

```cpp
void setMode(...);          // property: what was asked for
// or
void setPrefersUp(true);    // property

struct LayoutData : public EngravingItem::LayoutData
{
    ld_field<bool> up;      // layout data: what was decided
};
```

Layout reads the property, looks at the surrounding elements, and writes the field. The property stays as it was set.

`Articulation` is one demonstration of the same rule. Its `symId` used to be ambiguous: the symbol the user chose, or the symbol layout settles on. The resolved symbol now sits in `LayoutData`, next to the resolved placement, so it is plain that layout takes both into account:

```cpp
struct LayoutData : public EngravingItem::LayoutData
{
    ld_field<bool> up = { "[Articulation] up", true };
    ld_field<SymId> symId = { "[Articulation] symId", SymId::noSym };
};
```

Fields of `LayoutData` are `ld_field`s. How a read of a field that has not been calculated yet is detected is described in [00001 Engraving: LD_ACCESS](00001-Engraving-LD_ACCESS.md).

## Consequences

* For every element, the place of a field answers what it is. A member of the element is a property. A member of `LayoutData` is calculated during layout.
* Layout recomputes these fields on each pass. `LayoutData::reset()` drops the previous result.
* A preference stays as the user set it when the context changes. The layout field changes.
* Each element declares its own calculated fields and exposes them through `ldata()` and `mutldata()`.

## Alternatives

Store the resolved value as a property, next to the preference that produced it.

The resolved value depends on context that changes during layout. Saving it on the element means every combination of preference and context is stored and then kept consistent. The user's choice and the layout result become one muddle. This was the shape of the data before the split.

## Implementation

Every element has a `LayoutData`. The base holds the geometry layout produces for all of them:

```cpp
struct LayoutData {
    ld_field<PointF> m_pos = "pos";
    ld_field<Shape> m_shape = "shape";
    ld_field<Shape> m_mask = "mask";

    virtual void reset();
};
```

An element adds its own calculated fields and the accessors. `Articulation` again, only as the shape of that declaration:

```cpp
struct LayoutData : public EngravingItem::LayoutData
{
    ld_field<bool> up = { "[Articulation] up", true };
    ld_field<SymId> symId = { "[Articulation] symId", SymId::noSym };
};
DECLARE_LAYOUTDATA_METHODS(Articulation)
```

`DECLARE_LAYOUTDATA_METHODS` gives `ldata()`, `mutldata()`, and `createLayoutData()`. Layout writes through `mutldata()`. Readers use `ldata()`.
