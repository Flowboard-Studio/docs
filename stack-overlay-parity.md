# Stack Overlay Parity

This document defines Flowboard stack behavior for overlay mode only.

Linear stack semantics (`axis: vertical|horizontal`, spacing/distribution, row/column behavior) are intentionally untouched.

## Overlay Detection

A stack is rendered as overlay when either condition is true:

- `properties.axis === "overlay"`
- Any child is positioned:
  - `type: "positioned"`, or
  - child `properties` has any of `top`, `right`, `bottom`, `left`

If neither condition is true, the stack uses the existing linear path.

## Overlay Semantics

- Paint order: first child is bottom-most, last child is top-most.
- Child kinds:
  - Positioned child: absolute constraints (`top/right/bottom/left/width/height`) against stack bounds.
  - Non-positioned child: aligned by `alignment`.
- `fit` applies to non-positioned children:
  - `loose` (default): child keeps intrinsic size.
  - `expand`: child stretches to stack bounds.
- `clipBehavior` controls overflow:
  - `none` (default): overflow visible.
  - `hardEdge`: clip to stack bounds.

## Mapping

- React package and Web preview:
  - overlay container uses `position: relative`
  - positioned children use `position: absolute` with exact side constraints
  - non-positioned children are aligned with alignment-to-CSS mapping
  - `clipBehavior` maps to CSS `overflow: visible|hidden`
- Flutter package:
  - overlay uses `Stack(alignment, fit, clipBehavior, children)`
  - positioned children use `Positioned(...)`
  - non-positioned children are regular `Stack` children
