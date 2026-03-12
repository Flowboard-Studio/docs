# Onboarding JSON Runtime Specification

Scope:

- Derived from the current repository code only.
- "React Native" and "Expo" mean `flowboard-pckg-react`.
- "Flutter" means `flowboard-pckg-flutter`.
- "iOS / Swift" means the UIKit renderer in `flowboard-ios`.
- "Android / Kotlin" means the view renderer in `flowboard-android`.
- "Web Preview" means the dashboard/editor preview in `flowboard-dashboard/components/editor/DraggableComponent.tsx`.
- The referenced `Regles API Codex.pdf` is not present in this workspace snapshot, so no rules were taken from it.

## Table of contents

- [1. Global rules](#1-global-rules)
- [2. Component reference](#2-component-reference)
  - [stack](#stack)
  - [row](#row)
  - [column](#column)
  - [layout](#layout)
  - [container](#container)
  - [positioned](#positioned)
  - [padding](#padding)
  - [align](#align)
  - [expanded](#expanded)
  - [spacer](#spacer)
  - [text](#text)
  - [rich_text](#rich_text)
  - [button](#button)
  - [text_input](#text_input)
  - [selection_list](#selection_list)
  - [wheel_picker](#wheel_picker)
  - [image](#image)
  - [lottie](#lottie)
  - [grid](#grid)
  - [fake_progress_bar / fake_loader](#fake_progress_bar--fake_loader)
  - [slider](#slider)
  - [pageview_indicator](#pageview_indicator)
  - [icon](#icon)
  - [radar_chart](#radar_chart)
- [3. Verified source files](#3-verified-source-files)

## 1. Global rules

### 1.1 Canonical runtime model

- App SDKs consume runtime onboarding JSON, not the draft/editor model.
- A runtime onboarding object must contain a non-empty `screens` array.
- Each runtime screen must include `id`.
- A runtime screen is either:
  - a normal screen with `children`
  - a `type: "custom"` screen with a `properties` object
- Runtime validation currently requires at least one `finish` action somewhere in the flow. It only warns, not fails, if the last screen has no `finish`.

Runtime builder and canonicalizer rules:

- Draft `fake_loader` exports to runtime `fake_progress_bar`.
- Draft `layout` exports to runtime `row`, `column`, or `grid` from `properties.direction`.
- Draft `text_input`, `selection_list`, and `wheel_picker` export `properties.formId` as runtime component `id`, then remove `formId`.
- Runtime-to-draft canonicalization maps:
  - `row` -> `stack` with `axis: "horizontal"`
  - `column` -> `stack` with `axis: "vertical"`
  - `grid` -> draft `stack`
  - `fake_progress_bar` -> draft `fake_loader`
- Legacy `slide` is accepted by app renderers and normalized to `stack`.

### 1.2 Authoring contract vs runtime renderer

- The v1 AI contract allows only:
  - `stack`, `spacer`, `text`, `rich_text`, `button`, `text_input`, `selection_list`, `wheel_picker`, `image`, `lottie`, `slider`, `pageview_indicator`, `icon`, `fake_loader`
- The current runtime renderers also accept additional literal runtime types:
  - `row`, `column`, `layout`, `container`, `grid`, `padding`, `align`, `expanded`, `positioned`, `radar_chart`
- The v1 AI contract explicitly disallows:
  - `container`, `row`, `column`, `grid`, `padding`, `align`, `expanded`, `positioned`, `radar_chart`
- Draft validation strips unknown props for contract-backed components. Runtime SDKs do not do generic prop validation; they read known keys and ignore the rest.

### 1.3 Component tree structure

- Recursive component locations are:
  - `screen.children[]`
  - `component.children[]`
  - `component.child`
  - `component.options[].child`
- `padding`, `align`, `expanded`, and `positioned` are the only current runtime components with a meaningful single `child`.
- `selection_list` and `wheel_picker` use `options`.
- `slider` uses `children`.
- Runtime validators walk all of the locations above when checking actions and required input IDs.

Important tree differences:

- React Native and Flutter normalize stack-like nodes (`stack`, `row`, `column`, `layout`, `container`, `grid`) before rendering. During that normalization, `child` is folded into `children`.
- iOS and Android do not fold `child` into stack-like `children`. For literal runtime `stack`/`row`/`column`/`layout`/`container`, only `children` participate in layout.
- React Native, Flutter, iOS, and Android all normalize non-`stack` slider pages into generated `stack` pages before rendering.

### 1.4 Actions and runtime flow behavior

Runtime validators accept these actions:

- `next`
- `previous`
- `finish`
- `deeplink`
- `weblink`
- `rate_app`
- `request_permission`
- `jump_to`
- `jumpTo`

Validation rules:

- `jump_to` / `jumpTo` require `actionData.screenId`.
- `request_permission` requires `actionPermission`.
- `deeplink` / `weblink` require `properties.url`.
- Runtime `text_input`, `selection_list`, and `wheel_picker` require top-level `id`.

Host flow behavior:

- `next` validates `required: true` inputs and optional `regex` before advancing.
- `previous` goes back one screen.
- `finish` ends the flow and clears persisted progress.
- `deeplink` / `weblink` open `payload.url`, with app flows also falling back to `screen.url`.
- `rate_app` requests an in-app review, then advances or completes.
- `request_permission` only advances after permission is granted.
- `jump_to` / `jumpTo` jump to the screen with matching `screenId`.

Supported permission keys in the current contract and host flows:

- `push_notifications`
- `notifications`
- `contacts`
- `location`
- `camera`
- `microphone`
- `photos`
- `photo_library`

Component-level action reality:

- `button` uses its explicit `action`.
- `fake_progress_bar` uses its explicit `action` on completion.
- `selection_list` and `wheel_picker` do not honor their explicit `action` field in current app renderers. They only trigger `next` when `autoGoNext === true`.

### 1.5 Size model: fill / fit / fixed

Shared concepts:

- `fill`, `fit`, and `fixed` are the intended size modes.
- Fill aliases seen in app code:
  - `fill`
  - `infinity`
  - `infinite`
  - `double.infinity`
  - `+infinity`
  - `"100%"` in React Native, iOS, Android, and some Flutter fill checks
- Fixed pixel hints are used mainly through:
  - `width`
  - `height`
  - `size.widthPx`
  - `size.heightPx`

Actual SDK behavior:

- React Native and Flutter implement the clearest `fill` / `fit` / `fixed` behavior for `stack`, `text`, and `slider`.
- iOS partially mirrors the size model with Auto Layout priorities and slider parity helpers.
- Android is the loosest:
  - generic wrapper sizing only respects explicit numeric dimensions
  - many fill behaviors are left to the parent `LinearLayout` / `FrameLayout`

Dimension parsing differences:

- React Native:
  - numeric values and numeric strings are accepted
  - percent strings are accepted in layout helpers
- Flutter:
  - generic `_parseDimension()` accepts numbers and the string `"infinity"`
  - string numbers and percent strings are ignored there
  - some higher-level fill checks still recognize `"100%"`
- iOS and Android:
  - numeric values, numeric strings, and fill aliases are accepted by the expression resolver

### 1.6 Width and height behavior

- Stack-like defaults are axis-sensitive and SDK-sensitive.
- React Native / Flutter stack-like defaults:
  - vertical stack: width fill, height fit
  - horizontal stack: width fit, height fill
  - overlay stack: width fill, height fill
- iOS stack-like defaults are more approximate:
  - `stack` / `column` / `layout` default to width fill and height fit
  - `row` defaults to width fit and height fill
- Android does not have a comparable generic fill model.

Leaf component differences:

- `text` supports width modes on React Native, Flutter, and iOS. Android only has a simple `size.width == "fill"` wrapper.
- `button` defaults to full-width on React Native and Flutter. iOS generally fills width inside vertical stack layouts. Android leaves button width mostly intrinsic unless an explicit wrapper size is present.
- `text_input` stretches by default on React Native, Flutter, and iOS. Android has no generic stretch wrapper.

### 1.7 Margin and padding behavior

Insets parsing:

- Number -> all sides.
- Object forms:
  - all SDKs accept `top`, `right`, `bottom`, `left`
  - all app SDKs accept `horizontal`, `vertical`
  - Flutter and Web Preview also accept `x`, `y`
  - React Native only accepts `x`, `y` through stack-margin normalization, not generic `parseInsets()`
  - iOS and Android do not accept `x`, `y`

Margin source:

- `stack` uses `properties.layout.margin` first, then `properties.margin`
- non-`stack` components use `properties.margin`

Padding source:

- React Native and Flutter:
  - generic wrapper padding is applied to most non-`stack`, non-`container`, non-`padding` components through `properties.padding`
  - stack-like `layout.padding` is currently not carried through normalization, so literal runtime `stack` / `row` / `column` / `layout` / `container` do not honor it there
- iOS and Android:
  - stack-like renderers honor `properties.layout.padding` and fall back to `properties.padding`
  - there is no generic "wrap every component in padding" behavior
- `padding` component works on all app SDKs.

Screen padding:

- React Native reads `screen.padding`.
- Flutter, iOS, and Android fall back from `screen.padding` to `screen.margin`.

### 1.8 Position and alignment rules

Overlay detection:

- A stack is treated as overlay when:
  - `properties.axis === "overlay"`, or
  - any child is positioned, meaning:
    - `type: "positioned"`, or
    - child `properties` contains any of `top`, `right`, `bottom`, `left`

Overlay behavior:

- First child paints first; last child is top-most.
- Positioned children use explicit constraints.
- Non-positioned overlay children use alignment.
- `overlayAlignment` falls back to `alignment`.

Linear stack behavior:

- `alignment` controls cross-axis alignment.
- `distribution` controls main-axis layout.
- `childSpacing` inserts manual spacers only where the renderer still uses them.

Distribution differences:

- React Native and Flutter suppress manual spacing when `distribution` is `spaceBetween`, `spaceAround`, or `spaceEvenly`.
- iOS still sets `UIStackView.spacing`.
- Android still inserts explicit `Space` views between children.

### 1.9 Text rendering and expression rules

Interpolation:

- React Native, Flutter, iOS, and Android all resolve `{{...}}` expressions.
- Supported expression features:
  - direct variable lookup: `{{age}}`
  - math expressions: `{{weight / 2.2}}`
  - default values: `{{name|Guest}}`
  - date formatting: `{{DATE(yyyy-MM-dd)}}`

Numeric-string difference:

- React Native, iOS, and Android format integer math results as `x.0` when numeric strings were coerced into numbers.
- Flutter returns the evaluator result directly and does not add the extra `.0`.

Text style support:

- React Native and Flutter support:
  - `fontFamily`
  - `fontSize`
  - `fontWeight`
  - `fontStyle`
  - `letterSpacing`
  - line-height multiplier from `height`
  - `decoration`
  - gradients via `foreground`
- iOS supports only:
  - `fontSize`
  - basic `fontWeight`
  - `color`
  - `textAlign`
- Android supports only:
  - `fontSize`
  - basic `fontWeight`
  - `color`
  - `textAlign`

### 1.10 Image rendering rules

- Screen backgrounds support `color`, `gradient`, `image`, and `asset` on all app SDKs.
- Component images depend on `image` and `stack` behavior:
  - `image` component is the portable way to render an image asset or remote image
  - stack/component background images are not fully portable

`fit` defaults:

- React Native `image`: `contain`
- Flutter `image`: `contain`
- iOS `image`: `cover` (`scaleAspectFill`)
- Android `image`: `cover` (`CENTER_CROP`)

Source differences:

- React Native and Flutter require an explicit `source` branch:
  - `network` -> `url`
  - `asset` -> `path`
- iOS uses `url` when `source == "network"`, otherwise `path`, and also accepts file URLs.
- Android `FlowboardRemoteImageView` only loads URLs starting with `http`.

`blurhash`:

- Present in the AI contract and dashboard storage flows.
- No current app runtime renderer uses it.

### 1.11 Color format handling

All app SDKs accept:

- `rgba(...)`
- `rgb(...)`
- `#RRGGBB`
- `#AARRGGBB`
- `0xAARRGGBB`
- numeric ARGB integers

Additional differences:

- React Native and iOS and Android also accept `0xRRGGBB`.
- React Native stack/text helpers accept percent-like layout dimensions elsewhere, but color parsing itself does not.
- Dashboard preview accepts additional editor-friendly web forms and converts Flutter-style hex for CSS.

Important alpha rule:

- `#AARRGGBB` is treated as ARGB, not CSS `#RRGGBBAA`.
- Generic component opacity is not implemented. Alpha comes from:
  - the color value itself
  - button stroke/effect `opacity`

### 1.12 Corner radius, border, shadow, opacity

Portable support is limited:

- `appearance.cornerRadius` / `borderRadius`
  - fully used by React Native and Flutter on `stack`, `button`, and many leaf widgets
  - partially used by iOS on `button`, `text_input`, and some wrappers
  - lightly used on Android
- `border`
  - React Native and Flutter `stack` read `border.width` and `border.color`
  - iOS and Android stack-like renderers do not
- `shadow`
  - React Native and Flutter `stack` support only one shadow object with `color`, `x`, `y`, `blur`
  - iOS and Android stack-like renderers ignore stack shadow
  - `button.effects` have much better support than generic stack shadow
- `opacity`
  - no generic component `opacity` prop exists
  - button stroke/effect opacity is supported on React Native, Flutter, and iOS
  - Android ignores button stroke/effect opacity

### 1.13 Default behavior when a prop is omitted

General pattern:

- Boolean props are usually opt-in only (`=== true`).
- Missing objects fall back to empty objects and then to component defaults.
- Missing colors usually become transparent or a hard-coded component fallback.
- Missing unsupported props do nothing at runtime.

Common defaults:

- `button.borderRadius` -> `10`
- `button.label` -> `"Button"`
- `button.color` -> blue fallback
- `fake_progress_bar.duration` -> `3000` on React Native / Flutter, `3` seconds on iOS, `3` seconds interpreted as `3000` ms on Android
- `fake_progress_bar.style` -> `"linear"`
- `pageview_indicator.style` -> `"dots"`
- `pageview_indicator.dotSize` -> `8`
- `selection_list.spacing` / `wheel_picker.spacing` -> `8`
- `wheel_picker.visibleItems` -> `5`, then coerced to an odd number on React Native / Flutter

### 1.14 Unsupported prop behavior

- Runtime renderers ignore props they do not read.
- Some props are parsed but have no visible effect on some SDKs. Examples:
  - `pageview_indicator.style` only falls back to dots everywhere
  - `pageview_indicator.alignment` is ignored by app renderers
  - `selection_list.backgroundColor` is effectively Web Preview only
  - `placeholderStyle` is effectively Web Preview only
  - `text_input.strokeStyle = dashed|dotted` is only visibly honored on React Native / Web Preview
  - `stack.fill.type = image|asset` only has visible support in Web Preview

### 1.15 Cross-SDK differences that matter most

- Web Preview is not a runtime SDK. It still reflects some draft/editor behaviors that app runtimes never execute.
- React Native and Flutter normalize many literal layout types into `stack` before rendering.
- iOS and Android have materially smaller text, icon, and chart feature sets than React Native / Flutter / Web Preview.
- `selection_list` and `wheel_picker` explicit `action` fields are currently ignored by app runtimes.
- `radar_chart` is not parity-safe:
  - React Native / Flutter / Web Preview read `datasets[].data`
  - iOS / Android currently read `datasets[].values`
  - iOS / Android currently use `strokeColor`, not `borderColor`
- Android slider and fake progress bar are materially simpler than React Native / Flutter / iOS.

### 1.16 Screen-level rendering rules

Common screen props read by app SDKs:

- `backgroundColor`
- `background`
- `padding`
- `margin`
- `safeArea`
- `pageScroll`
- `scrollDirection`
- `scrollable`
- `mainAxisAlignment`
- `crossAxisAlignment`
- `showProgress`
- `progressColor`
- `progressThickness`
- `progressRadius`
- `progressStyle`

Screen behavior:

- `background` supports `color`, `gradient`, `image`, and `asset` on all app SDKs.
- If no background is provided, app renderers fall back to a white screen.
- `pageScroll` is normalized to:
  - `vertical`
  - `none`
- `scrollable: true` backfills `pageScroll = "vertical"` in the draft validator and is still read by app renderers as a fallback.
- `showProgress` adds a screen-level progress bar ahead of the normal component tree.

Screen differences:

- React Native, Flutter, and iOS honor `safeArea`.
- Android currently ignores `safeArea`.
- React Native reads `screen.padding` only.
- Flutter, iOS, and Android fall back from `screen.padding` to `screen.margin`.

## 2. Component reference

## stack

Description: Canonical runtime layout container.

Rendering intent: Vertical stack, horizontal stack, or overlay stack.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `children` | array | no | `[]` | Primary content list. |
| `axis` | string | no | `vertical` | `horizontal`, `vertical`, or `overlay`. Overlay can also be inferred from positioned children. |
| `mode` | string | no | none | Legacy `overlay` / `linear` hint read by React Native / Flutter / Web Preview stack-like normalization. |
| `alignment` | string | no | `start` | Cross-axis alignment for linear stacks; overlay fallback alignment for non-positioned children. |
| `distribution` | string | no | `start` | Main-axis layout for linear stacks. |
| `overlayAlignment` | string | no | falls back to `alignment` | Used only for overlay children. |
| `childSpacing` | number | no | `0` | Manual child spacing. |
| `size.width` / `size.height` | string | no | axis-dependent | Intended modes are `fill`, `fit`, `fixed`. |
| `size.widthPx` / `size.heightPx` | number | no | none | Used with fixed sizing in React Native, Flutter, and iOS. |
| `width` / `height` | number or string | no | none | Explicit dimensions; parsing varies by SDK. |
| `fit` | string | no | `loose` | Overlay container fit on React Native / Flutter. Limited elsewhere. |
| `clipBehavior` | string | no | `none` | `hardEdge` clips overlay content on React Native, Flutter, and iOS. |
| `fill` | object | no | transparent solid fill | Portable forms are solid and gradient. |
| `background` / `backgroundColor` | legacy | no | none | Legacy aliases still read at runtime. |
| `border.width` / `border.color` | number / color | no | `0` / `#E5E7EB` | Used by React Native / Flutter stack rendering. |
| `borderColor` / `borderWidth` | legacy | no | none | Legacy aliases used by React Native / Flutter / Web Preview. |
| `appearance.cornerRadius` / `borderRadius` | number | no | `0` | Corner radius. |
| `shadow.color` / `shadow.x` / `shadow.y` / `shadow.blur` | mixed | no | platform fallback | Used by React Native / Flutter only for stack. |
| `layout.margin` / `margin` | number or insets object | no | `0` | Stack margin source prefers `layout.margin`. |
| `layout.padding` / `padding` | number or insets object | no | `0` | Honored by iOS / Android stack-like renderers. React Native / Flutter stack normalization drops it. |

Display behavior:

- React Native / Flutter:
  - render solid or gradient fill
  - render border, corner radius, and one shadow object
  - support overlay and linear modes
- iOS / Android:
  - render only solid background color on stack-like nodes
  - do not render stack border or stack shadow

Layout behavior:

- Overlay mode uses absolute positioning for positioned children and alignment wrappers for others.
- Linear mode uses `alignment`, `distribution`, and `childSpacing`.
- React Native / Flutter normalize `row`, `column`, `layout`, `container`, and literal `grid` into `stack` before rendering.

Interaction behavior:

- None.

Known SDK differences:

- React Native / Flutter stack-like normalization still reads legacy `mainAxisAlignment`, `crossAxisAlignment`, `gap`, and `spacing`.
- React Native / Flutter ignore `layout.padding` on literal runtime stack-like nodes.
- Web Preview supports stack background images; app SDKs do not for `stack.fill` / `stack.background`.
- iOS clips overlay stacks when `clipBehavior == "hardEdge"`. Android overlay stack does not expose a comparable clip setting.

Minimal JSON example:

```json
{
  "type": "stack",
  "properties": {
    "axis": "vertical"
  },
  "children": []
}
```

Full JSON example:

```json
{
  "type": "stack",
  "properties": {
    "axis": "overlay",
    "overlayAlignment": "bottom",
    "fit": "expand",
    "clipBehavior": "hardEdge",
    "size": {
      "width": "fill",
      "height": "fixed",
      "heightPx": 240
    },
    "fill": {
      "type": "gradient",
      "colors": ["0xFF0F172A", "0xFF1D4ED8"],
      "begin": "topLeft",
      "end": "bottomRight"
    },
    "border": {
      "width": 1,
      "color": "0x33FFFFFF"
    },
    "appearance": {
      "cornerRadius": 24
    },
    "shadow": {
      "color": "0x40000000",
      "x": 0,
      "y": 12,
      "blur": 24
    },
    "layout": {
      "margin": {
        "horizontal": 20,
        "top": 12
      }
    }
  },
  "children": [
    {
      "type": "text",
      "properties": {
        "text": "Overlay title"
      }
    },
    {
      "type": "positioned",
      "properties": {
        "right": 16,
        "bottom": 16
      },
      "child": {
        "type": "button",
        "properties": {
          "label": "Continue"
        },
        "action": "next"
      }
    }
  ]
}
```

## row

Description: Stack-like alias that defaults to horizontal layout.

Rendering intent: Horizontal linear layout when no explicit `axis` overrides it.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `children` | array | no | `[]` | Same content rules as `stack`. |
| `axis` | string | no | `horizontal` when omitted | Explicit `axis` still wins if provided. |
| `alignment`, `distribution`, `childSpacing`, `size`, `width`, `height`, `fill`, `border`, `shadow`, `layout.margin` | inherited | no | stack-like defaults | React Native / Flutter read these through stack normalization. iOS / Android route `row` through stack-like rendering. |

Display behavior:

- Behaves like a horizontal `stack`.

Layout behavior:

- Main axis is horizontal unless `properties.axis` overrides it.

Interaction behavior:

- None.

Known SDK differences:

- React Native / Flutter still read legacy `mainAxisAlignment`, `crossAxisAlignment`, `gap`, and `spacing` during normalization.
- iOS / Android do not read `mainAxisAlignment` on literal runtime `row`; use `distribution` instead.

Minimal JSON example:

```json
{
  "type": "row",
  "children": []
}
```

Full JSON example:

```json
{
  "type": "row",
  "properties": {
    "distribution": "spaceBetween",
    "alignment": "center",
    "childSpacing": 12,
    "margin": {
      "horizontal": 20
    }
  },
  "children": [
    {
      "type": "text",
      "properties": {
        "text": "Left"
      }
    },
    {
      "type": "text",
      "properties": {
        "text": "Right"
      }
    }
  ]
}
```

## column

Description: Stack-like alias that defaults to vertical layout.

Rendering intent: Vertical linear layout when no explicit `axis` overrides it.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `children` | array | no | `[]` | Same content rules as `stack`. |
| `axis` | string | no | `vertical` when omitted | Explicit `axis` still wins if provided. |
| `alignment`, `distribution`, `childSpacing`, `size`, `width`, `height`, `fill`, `border`, `shadow`, `layout.margin` | inherited | no | stack-like defaults | Same stack-like rules as `stack`. |

Display behavior:

- Behaves like a vertical `stack`.

Layout behavior:

- Main axis is vertical unless `properties.axis` overrides it.

Interaction behavior:

- None.

Known SDK differences:

- Same stack-like differences as `stack`.

Minimal JSON example:

```json
{
  "type": "column",
  "children": []
}
```

Full JSON example:

```json
{
  "type": "column",
  "properties": {
    "alignment": "stretch",
    "childSpacing": 10
  },
  "children": [
    {
      "type": "text",
      "properties": {
        "text": "Headline"
      }
    },
    {
      "type": "button",
      "properties": {
        "label": "Next"
      },
      "action": "next"
    }
  ]
}
```

## layout

Description: Legacy layout wrapper from the draft/editor model.

Rendering intent: Draft export converts it to `row`, `column`, or `grid`. Runtime renderers still accept literal `layout`, but not consistently.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `direction` | string | no | `vertical` | React Native / Flutter only distinguish `horizontal` vs other. iOS / Android ignore it when `axis` is omitted. |
| `children` | array | no | `[]` | Runtime builder normally exports `layout` away before app runtimes see it. |
| stack-like layout props | inherited | no | stack-like defaults | Same stack-like rules as `stack`. |

Display behavior:

- Prefer to think of literal runtime `layout` as a compatibility path, not the canonical runtime type.

Layout behavior:

- React Native / Flutter:
  - `direction == "horizontal"` -> horizontal stack
  - any other direction -> vertical stack
- iOS / Android:
  - literal `layout` without explicit `axis` defaults to vertical stack

Interaction behavior:

- None.

Known SDK differences:

- Draft export is the only place where `direction == "grid"` becomes a real runtime `grid`.
- Dashboard Web Preview has dedicated draft `layout` support, including CSS grid.

Minimal JSON example:

```json
{
  "type": "layout",
  "properties": {
    "direction": "horizontal"
  },
  "children": []
}
```

Full JSON example:

```json
{
  "type": "layout",
  "properties": {
    "direction": "horizontal",
    "mainAxisAlignment": "spaceBetween",
    "crossAxisAlignment": "center",
    "gap": 12
  },
  "children": [
    {
      "type": "text",
      "properties": {
        "text": "A"
      }
    },
    {
      "type": "text",
      "properties": {
        "text": "B"
      }
    }
  ]
}
```

## container

Description: Legacy wrapper container with fragmented runtime semantics.

Rendering intent: Preview/editor container and stack-like runtime compatibility wrapper.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `children` | array | no | `[]` | Safest runtime content form across app SDKs. |
| `child` | object | no | none | Folded into `children` by React Native / Flutter normalization. Ignored by iOS / Android stack-like path. |
| `background` / `backgroundColor` | mixed | no | transparent | React Native / Flutter treat it as stack fill. Web Preview has dedicated container rendering. |
| `borderRadius` | number | no | `0` | Mapped to stack corner radius on React Native / Flutter. |
| `borderColor` / `borderWidth` | mixed | no | none | Legacy border aliases. |
| `padding` | number or insets object | no | `0` | Web Preview dedicated container uses it. React Native / Flutter runtime normalization drops it. iOS / Android stack-like path can honor stack-style padding only through `children`. |
| `width` / `height` | number or string | no | none | Explicit size where the renderer supports it. |

Display behavior:

- React Native / Flutter normalize literal runtime `container` into `stack` with overlay axis.
- iOS / Android route `container` through stack-like rendering with overlay axis.
- Web Preview has a dedicated single-child container preview with padding and background decoration.

Layout behavior:

- Behaves most like an overlay stack, not a generic CSS-style box.

Interaction behavior:

- None.

Known SDK differences:

- `padding` is not parity-safe on runtime `container`.
- `child` is not parity-safe on runtime `container`; use `children`.

Minimal JSON example:

```json
{
  "type": "container",
  "children": []
}
```

Full JSON example:

```json
{
  "type": "container",
  "properties": {
    "backgroundColor": "0xFFF8FAFC",
    "borderRadius": 16,
    "borderColor": "0xFFE2E8F0",
    "borderWidth": 1
  },
  "children": [
    {
      "type": "text",
      "properties": {
        "text": "Container content"
      }
    }
  ]
}
```

## positioned

Description: Explicit absolute-position wrapper.

Rendering intent: Place one child at fixed edges inside an overlay stack.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `child` | object | no | none | Wrapped content. |
| `top`, `right`, `bottom`, `left` | number or string | no | none | Absolute offsets. |
| `width`, `height` | number or string | no | none | Explicit child frame size. |

Display behavior:

- Renders only when its parent behaves like an overlay container.

Layout behavior:

- React Native / Flutter / iOS / Android all also treat ordinary children with `top` / `right` / `bottom` / `left` as positioned, even without `type: "positioned"`.

Interaction behavior:

- None.

Known SDK differences:

- Percent positioning is supported by React Native / Web Preview. Native app SDKs use numeric dimensions only.

Minimal JSON example:

```json
{
  "type": "positioned",
  "properties": {
    "top": 0,
    "right": 0
  }
}
```

Full JSON example:

```json
{
  "type": "positioned",
  "properties": {
    "right": 16,
    "bottom": 16,
    "width": 120,
    "height": 48
  },
  "child": {
    "type": "button",
    "properties": {
      "label": "Skip"
    },
    "action": "finish"
  }
}
```

## padding

Description: Explicit padding wrapper.

Rendering intent: Apply insets around one child.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `child` | object | no | none | Wrapped content. |
| `padding` | number or insets object | no | `0` | Primary padding source. |
| `margin` | number or insets object | no | `0` | iOS / Android also fall back to it inside the padding component. |

Display behavior:

- Adds visual space around its child.

Layout behavior:

- Uses `child`.

Interaction behavior:

- None.

Known SDK differences:

- React Native / Flutter read only `padding`.
- iOS / Android read `padding ?? margin`.

Minimal JSON example:

```json
{
  "type": "padding",
  "properties": {
    "padding": 12
  }
}
```

Full JSON example:

```json
{
  "type": "padding",
  "properties": {
    "padding": {
      "horizontal": 20,
      "top": 12,
      "bottom": 16
    }
  },
  "child": {
    "type": "text",
    "properties": {
      "text": "Padded text"
    }
  }
}
```

## align

Description: Explicit alignment wrapper.

Rendering intent: Align one child inside the available box.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `child` | object | no | none | Wrapped content. |
| `alignment` | string | no | `center` | Uses the same alignment vocabulary as overlay alignment helpers. |

Display behavior:

- Positions its child inside its own bounds.

Layout behavior:

- Uses `child`.

Interaction behavior:

- None.

Known SDK differences:

- Android and iOS use view gravity / constraints.
- React Native and Flutter use alignment wrappers.

Minimal JSON example:

```json
{
  "type": "align",
  "properties": {
    "alignment": "center"
  }
}
```

Full JSON example:

```json
{
  "type": "align",
  "properties": {
    "alignment": "bottomEnd"
  },
  "child": {
    "type": "icon",
    "properties": {
      "icon": "check",
      "size": 20
    }
  }
}
```

## expanded

Description: Flex-expansion wrapper.

Rendering intent: Ask the parent linear layout to give the child remaining main-axis space.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `child` | object | no | none | Wrapped content. |

Display behavior:

- No visual style of its own.

Layout behavior:

- React Native wraps the child in `flex: 1` when the parent allows flex expansion.
- Flutter returns `Expanded(child: ...)` only when the parent main axis is bounded.
- iOS just lowers hugging / compression on the child.
- Android simply returns the child; there is no dedicated `LinearLayout` weight path here.

Interaction behavior:

- None.

Known SDK differences:

- This is not parity-safe across all SDKs.

Minimal JSON example:

```json
{
  "type": "expanded"
}
```

Full JSON example:

```json
{
  "type": "expanded",
  "child": {
    "type": "stack",
    "properties": {
      "axis": "vertical",
      "size": {
        "width": "fill",
        "height": "fill"
      }
    },
    "children": []
  }
}
```

## spacer

Description: Empty spacing block.

Rendering intent: Reserve fixed width and/or height.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `width` | number or string | no | none | Explicit width. |
| `height` | number or string | no | none | Explicit height. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- Renders as empty space.

Layout behavior:

- Takes its explicit width / height where supported.

Interaction behavior:

- None.

Known SDK differences:

- iOS and Android default to `height = 20` when `height` is omitted.
- React Native and Flutter do not add that fallback.

Minimal JSON example:

```json
{
  "type": "spacer",
  "properties": {
    "height": 16
  }
}
```

Full JSON example:

```json
{
  "type": "spacer",
  "properties": {
    "width": 24,
    "height": 24,
    "margin": {
      "top": 8,
      "bottom": 8
    }
  }
}
```

## text

Description: Plain text label.

Rendering intent: Render one resolved string.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `text` | string | yes | none | Supports expression resolution. |
| `fontFamily` | string | no | platform default | React Native / Flutter only. |
| `fontSize` | number | no | `14` React Native / Flutter, `16` iOS / Android | |
| `fontWeight` | string or number | no | regular | Basic weight support on all app SDKs. |
| `fontStyle` | string | no | normal | React Native / Flutter only. |
| `letterSpacing` | number | no | none | React Native / Flutter only. |
| `height` | number or string | no | `1` multiplier | React Native / Flutter interpret as line-height multiplier, not pixels. |
| `decoration` | string | no | none | React Native / Flutter only. |
| `textAlign` | string | no | start / left | All app SDKs use it. |
| `color` | color | no | black / label | All app SDKs use it. |
| `foreground` | object | no | none | Gradient text on React Native / Flutter / Web Preview. Ignored on iOS / Android. |
| `size.width`, `size.widthPx`, `width` | mixed | no | fit | Width-mode handling on React Native / Flutter / iOS. Android only supports a simple fill wrapper. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- All app SDKs resolve `{{...}}` expressions before rendering.
- React Native / Flutter can render gradient text through `foreground`.

Layout behavior:

- React Native / Flutter / iOS support text width modes.
- Android only has a special-case fill wrapper when `size.width == "fill"`.

Interaction behavior:

- None.

Known SDK differences:

- iOS and Android ignore `fontFamily`, `fontStyle`, `letterSpacing`, `height`, `decoration`, and `foreground`.

Minimal JSON example:

```json
{
  "type": "text",
  "properties": {
    "text": "Hello"
  }
}
```

Full JSON example:

```json
{
  "type": "text",
  "properties": {
    "text": "Hello {{name|Guest}}",
    "fontFamily": "Montserrat",
    "fontSize": 24,
    "fontWeight": "700",
    "letterSpacing": 0.5,
    "height": 1.2,
    "textAlign": "center",
    "foreground": {
      "type": "gradient",
      "colors": ["0xFF2563EB", "0xFF06B6D4"],
      "begin": "topLeft",
      "end": "bottomRight"
    },
    "size": {
      "width": "fill"
    }
  }
}
```

## rich_text

Description: Concatenated text spans with per-span style data.

Rendering intent: Render multiple styled text segments as one rich label.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `spans` | array | yes | none | Each span contributes ordered text. |
| `spans[].text` | string | yes | empty string | Supports expression resolution. |
| `spans[].color` | color | no | black / label | All app SDKs use span colors. |
| `spans[].fontSize` | number | no | platform default | All app SDKs use it. |
| `spans[].fontWeight` | string or number | no | regular | React Native / Flutter / iOS. Android rich text only applies size and color. |
| `spans[].fontFamily`, `spans[].fontStyle`, `spans[].letterSpacing`, `spans[].height`, `spans[].decoration`, `spans[].foreground` | mixed | no | none | React Native / Flutter only. |
| `textAlign` | string | no | start / left | All app SDKs use it. |
| `foreground` | object | no | none | Whole-label gradient on React Native / Flutter / Web Preview only. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native and Flutter support per-span gradients.
- React Native and Flutter support top-level whole-label gradients.

Layout behavior:

- Same general text width handling as `text`.

Interaction behavior:

- None.

Known SDK differences:

- React Native top-level `foreground` flattens the spans into one combined gradient string, so per-span typography is lost there.
- Flutter keeps the spans under a `ShaderMask`, so per-span typography survives.
- iOS and Android ignore all gradient support.

Minimal JSON example:

```json
{
  "type": "rich_text",
  "properties": {
    "spans": [
      {
        "text": "Hello"
      }
    ]
  }
}
```

Full JSON example:

```json
{
  "type": "rich_text",
  "properties": {
    "textAlign": "center",
    "spans": [
      {
        "text": "Build ",
        "fontSize": 20,
        "fontWeight": "700",
        "color": "0xFF0F172A"
      },
      {
        "text": "faster",
        "fontSize": 20,
        "fontWeight": "700",
        "foreground": {
          "type": "gradient",
          "colors": ["0xFF2563EB", "0xFF06B6D4"]
        }
      }
    ]
  }
}
```

## button

Description: Pressable CTA.

Rendering intent: Render a tappable button and dispatch an explicit action.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `label` | string | yes | `"Button"` | Visible title. |
| `action` | string | no | none | The explicit action this component dispatches. |
| `actionData.screenId` | string | conditional | none | Needed for `jump_to` / `jumpTo`. |
| `actionPermission` | string | conditional | none | Needed for `request_permission`. |
| `url` | string | conditional | none | Needed for `deeplink` / `weblink`. |
| `color` | color | no | blue fallback | Solid background fallback. |
| `background` | object | no | none | Gradient background on React Native / Flutter / iOS. |
| `textColor` | color | no | white | Button label color. |
| `borderRadius` | number | no | `10` | Corner radius. |
| `width` / `height` | number or string | no | full-width + `50` on React Native / Flutter | Android does not add the explicit `50` fallback. |
| `size` | object | no | none | Only partially honored. Do not rely on it for parity. |
| `stroke.enabled`, `stroke.width`, `stroke.color`, `stroke.position`, `stroke.opacity` | mixed | no | centered 1px-style defaults where supported | Full support on React Native / Flutter / iOS. Android only uses width + color. |
| `effects[]` | array | no | `[]` | Drop-shadow effects. React Native / Flutter / iOS read effect `x`, `y`, `blur`, `spread`, `color`, and `opacity`. |
| `shadow` | legacy object | no | none | Migrated to button effects in React Native / Flutter / iOS. Android only uses the first effect for elevation. |
| `fontFamily`, `fontSize`, `fontWeight`, `fontStyle`, `letterSpacing` | mixed | no | platform default | React Native / Flutter / iOS. Android ignores them. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native / Flutter / iOS support solid and gradient backgrounds.
- Android always uses a solid `GradientDrawable`.

Layout behavior:

- Explicit width / height are portable.
- Size-model support is incomplete for button compared with stack / slider.

Interaction behavior:

- Press dispatches the component `action` with merged `props`, `actionData`, and `actionPermission`.

Known SDK differences:

- Android ignores gradient backgrounds, stroke position, stroke opacity, extra shadow layers, and most font props.
- iOS, React Native, and Flutter support multiple drop shadows.

Minimal JSON example:

```json
{
  "type": "button",
  "properties": {
    "label": "Continue"
  },
  "action": "next"
}
```

Full JSON example:

```json
{
  "type": "button",
  "properties": {
    "label": "Open offer",
    "background": {
      "type": "gradient",
      "colors": ["0xFF2563EB", "0xFF1D4ED8"]
    },
    "textColor": "0xFFFFFFFF",
    "borderRadius": 18,
    "height": 56,
    "stroke": {
      "enabled": true,
      "width": 2,
      "color": "0x33FFFFFF",
      "position": "inside",
      "opacity": 1
    },
    "effects": [
      {
        "type": "dropShadow",
        "enabled": true,
        "x": 0,
        "y": 12,
        "blur": 24,
        "spread": 0,
        "color": "0xFF000000",
        "opacity": 0.22
      }
    ],
    "url": "https://example.com/offer"
  },
  "action": "weblink"
}
```

## text_input

Description: Single-line or multiline text entry field.

Rendering intent: Collect one string value into runtime `formData[id]`.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | string | yes in runtime JSON | none | Runtime field key. |
| `label` | string | no | none | Visible label. |
| `placeholder` | string | no | none | Hint text. |
| `keyboardType` | string | no | default keyboard | `email`, `number`, `phone`, `multiline` are the current meaningful values. |
| `autoCapitalize` | string | no | platform default | React Native / Flutter only. |
| `mask` | string | no | none | React Native / Flutter only. |
| `backgroundColor` | color | no | light gray fallback | Input fill color. |
| `textColor` | color | no | black | Input text color. |
| `borderRadius` / `strokeRadius` | number | no | `8` | Input corner radius. |
| `strokeEnabled` | boolean | no | `strokeWidth > 0` | Border enable switch. |
| `strokeColor` | color | no | black | Border color. |
| `strokeWidth` | number | no | `0` | Border width. |
| `strokeStyle` | string | no | `solid` | Visible only on React Native / Web Preview. |
| `padding` | number | no | `12` | Inner content padding. |
| `required` | boolean | no | `false` | Validated by flow containers, not by the widget itself. |
| `regex` | string | no | none | Validated by flow containers, not by the widget itself. |
| `labelStyle` | object | no | none | React Native / Flutter only. |
| `hintStyle` | object | no | none | React Native / Flutter only. |
| `placeholderStyle` | object | no | none | Effectively Web Preview only. |
| `obscureText` | boolean | no | `false` | React Native / Flutter only. |
| `maxLength` | number | no | none | React Native / Flutter only. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native / Flutter render styled text fields with label, placeholder, fill, border, and text styling.
- iOS renders `UITextField` or `UITextView` with basic styling.
- Android renders a simple `EditText` in a `LinearLayout`.

Layout behavior:

- React Native / Flutter / iOS usually stretch to full width by default.
- Android has no equivalent generic fill wrapper.

Interaction behavior:

- Every text change updates `formData[id]`.
- Validation happens only when the flow handles `next`.

Known SDK differences:

- React Native and Flutter always set `autoFocus: true`.
- iOS treats `keyboardType == "multiline"` as a multiline control.
- Android ignores `keyboardType`, `autoCapitalize`, `mask`, `obscureText`, `maxLength` style objects, and regex at render time.
- Flutter does not visibly support dashed or dotted border styles.

Minimal JSON example:

```json
{
  "type": "text_input",
  "id": "email",
  "properties": {
    "placeholder": "Email"
  }
}
```

Full JSON example:

```json
{
  "type": "text_input",
  "id": "phone",
  "properties": {
    "label": "Phone",
    "placeholder": "06 00 00 00 00",
    "keyboardType": "phone",
    "autoCapitalize": "none",
    "mask": "## ## ## ## ##",
    "backgroundColor": "0xFFF8FAFC",
    "textColor": "0xFF0F172A",
    "strokeEnabled": true,
    "strokeColor": "0xFFCBD5E1",
    "strokeWidth": 1,
    "strokeRadius": 14,
    "padding": 14,
    "required": true,
    "regex": "^[0-9 ]+$",
    "labelStyle": {
      "fontSize": 12,
      "color": "0xFF64748B"
    },
    "hintStyle": {
      "fontSize": 16,
      "color": "0xFF94A3B8"
    }
  }
}
```

## selection_list

Description: Choice list that writes one or more selected values into `formData[id]`.

Rendering intent: Tap one or more options rendered from `options[]`.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | string | yes in runtime JSON | none | Runtime field key. |
| `options[]` | array | yes for useful output | `[]` | Portable runtime shape is `{ "value": "...", "child": { ... } }`. |
| `options[].value` | string | yes | generated upstream if omitted | Stored selection value. |
| `options[].child` | object | effectively yes for parity | none | React Native / Flutter rely on it. iOS / Android can fall back to the value label. |
| `options[].selectedStyle` / `options[].unselectedStyle` | object | no | none | React Native / Flutter / Web Preview only. |
| `layout` | string | no | `column` | `row` and `column` everywhere; `grid` only on React Native / Flutter / Web Preview. |
| `spacing` | number | no | `8` | Gap between options. |
| `gridCrossAxisCount` | number | no | `2` | React Native / Flutter / Web Preview only. |
| `gridAspectRatio` | number | no | `1` | React Native / Flutter / Web Preview only. |
| `multiSelect` | boolean | no | `false` | React Native / Flutter / Web Preview only. |
| `autoGoNext` | boolean | no | `false` | Triggers `next` after a single selection. |
| `selectedStyle` / `unselectedStyle` | object | no | empty object | Option chrome styles. |
| `backgroundColor` | color | no | none | Effectively Web Preview only. |
| `required` | boolean | no | `false` | Validated by flow containers. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native / Flutter render each option child inside a pressable styled wrapper.
- iOS / Android create buttons or button-like views.

Layout behavior:

- `row` uses horizontal wrapping.
- `grid` is only implemented in React Native / Flutter / Web Preview.

Interaction behavior:

- Single select writes one string.
- React Native / Flutter multi-select writes a comma-joined string.
- `autoGoNext` dispatches `next` after the value is stored.

Known SDK differences:

- Explicit component `action` is ignored by current app renderers.
- iOS and Android are effectively single-select only.
- `backgroundColor` on the selection-list container is preview-only.

Minimal JSON example:

```json
{
  "type": "selection_list",
  "id": "goal",
  "options": [
    {
      "value": "gain",
      "child": {
        "type": "text",
        "properties": {
          "text": "Gain"
        }
      }
    }
  ]
}
```

Full JSON example:

```json
{
  "type": "selection_list",
  "id": "activity_level",
  "properties": {
    "layout": "grid",
    "gridCrossAxisCount": 2,
    "gridAspectRatio": 1.2,
    "spacing": 12,
    "multiSelect": true,
    "selectedStyle": {
      "backgroundColor": "0xFFE0F2FE",
      "borderColor": "0xFF0284C7",
      "borderWidth": 2,
      "borderRadius": 16
    },
    "unselectedStyle": {
      "backgroundColor": "0xFFF8FAFC",
      "borderColor": "0xFFE2E8F0",
      "borderWidth": 1,
      "borderRadius": 16
    },
    "required": true
  },
  "options": [
    {
      "value": "walking",
      "child": {
        "type": "text",
        "properties": {
          "text": "Walking"
        }
      }
    },
    {
      "value": "running",
      "child": {
        "type": "text",
        "properties": {
          "text": "Running"
        }
      }
    }
  ]
}
```

## wheel_picker

Description: Picker-style choice component with generated or explicit options.

Rendering intent: Render a wheel or a selection layout and write the selected value into `formData[id]`.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | string | yes in runtime JSON | none | Runtime field key. |
| `options[]` | array | no if generated | `[]` | Explicit options. |
| `options[].value` | string | yes for explicit options | none | Stored value. |
| `options[].text` | string | no | falls back to `value` or child text | Used by wheel renderers. |
| `layout` | string | no | `wheel` in React Native / Flutter, native picker on iOS / Android | `row`, `column`, `grid`, `wheel` are only meaningful on React Native / Flutter / Web Preview. |
| `spacing` | number | no | `8` | Gap between non-wheel items and wheel rows. |
| `wheelItemHeight` | number | no | `48` | React Native / Flutter / Web Preview only. |
| `visibleItems` | number | no | `5` then odd-coerced | React Native / Flutter / Web Preview only. |
| `startItemIndex` / `startIndex` / `start_index` | number | no | `1` | React Native / Flutter wheel only, and only when there is no initial value. |
| `itemCount` | number or expression | no | `0` | When `> 0`, options are generated. |
| `start` / `itemStart` | number or expression | no | `0` | Generated option start. |
| `step` / `itemStep` | number or expression | no | `1` | Generated option step. |
| `itemTemplate` | object | no | none | Template for generated options. |
| `wheelTextStyle` / `wheelSelectedTextStyle` | object | no | empty object | React Native / Flutter / Web Preview only. |
| `wheelSelectedBackgroundColor`, `wheelSelectedBorderColor`, `wheelUnselectedBackgroundColor`, `wheelUnselectedBorderColor` | color | no | component fallback | React Native / Flutter / Web Preview only. |
| `wheelCenterBackgroundColor` / `wheelCenterBorderColor` | color | no | platform fallback | React Native / Flutter / Web Preview only. |
| `wheelEdgeFadeOpacity` / `wheelEdgeFadeColor` | mixed | no | `0` / white | React Native / Flutter / Web Preview only. |
| `wheelHorizontalInset`, `wheelItemVerticalInset` | number | no | Flutter fallbacks | Flutter-only wheel layout chrome props. |
| `selectedStyle` / `unselectedStyle` | object | no | empty object | Used for non-wheel layouts, and as a fallback source for some wheel chrome. |
| `multiSelect` | boolean | no | `false` | Read by React Native / Flutter / Web Preview in non-wheel layouts even though the v1 contract strips it from draft authoring. |
| `autoGoNext` | boolean | no | `false` | Triggers `next` after selection. |
| `required` | boolean | no | `false` | Validated by flow containers. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native / Flutter can render:
  - wheel
  - row
  - column
  - grid
- iOS always renders `UIPickerView`.
- Android always renders `NumberPicker`.

Layout behavior:

- Generated options use `itemCount`, `start`, `step`, and `itemTemplate`.

Interaction behavior:

- Wheel mode is single-select everywhere.
- React Native / Flutter non-wheel layouts can multi-select and store a comma-joined string.
- `autoGoNext` dispatches `next`.

Known SDK differences:

- Explicit component `action` is ignored by current app renderers.
- iOS and Android ignore `layout`, `multiSelect`, `startItemIndex`, and most wheel styling props.
- Generated option template context:
  - React Native / Flutter / Android include `index`, `item`, `start`, `step`
  - iOS includes `index`, `item`, `start`, but not `step`

Minimal JSON example:

```json
{
  "type": "wheel_picker",
  "id": "age",
  "properties": {
    "itemCount": 3,
    "start": 18,
    "step": 1
  }
}
```

Full JSON example:

```json
{
  "type": "wheel_picker",
  "id": "weight",
  "properties": {
    "layout": "wheel",
    "itemCount": 5,
    "start": 60,
    "step": 5,
    "itemTemplate": {
      "value": "{{item}}",
      "child": {
        "type": "text",
        "properties": {
          "text": "{{item}} kg"
        }
      }
    },
    "startItemIndex": 2,
    "wheelItemHeight": 52,
    "visibleItems": 5,
    "wheelTextStyle": {
      "fontSize": 18,
      "color": "0xFF475569"
    },
    "wheelSelectedTextStyle": {
      "fontSize": 20,
      "fontWeight": "700",
      "color": "0xFF0F172A"
    },
    "wheelCenterBackgroundColor": "0x24FFFFFF",
    "wheelCenterBorderColor": "0xFFCBD5E1"
  }
}
```

## image

Description: Image view for network or asset sources.

Rendering intent: Render one raster image.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `source` | string | effectively yes for React Native / Flutter | none | `network` or `asset`. |
| `url` | string | conditional | none | Used for `source == "network"`. |
| `path` | string | conditional | none | Used for `source == "asset"`. |
| `width` / `height` | number or string | no | none | Explicit size. |
| `fit` | string | no | SDK default | Common values: `contain`, `cover`, `fill`, `none`, `scaleDown`. |
| `blurhash` | string | no | none | No runtime effect in current app SDKs. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native uses an animated `Image` wrapper and tries to infer aspect ratio for remote images.
- Flutter uses `Image.network` / `Image.asset`.
- iOS uses `FlowboardRemoteImageView`.
- Android uses `FlowboardRemoteImageView`.

Layout behavior:

- Explicit width / height are portable.
- React Native has the most aspect-ratio fallback logic when width or height is omitted.

Interaction behavior:

- None.

Known SDK differences:

- iOS can load file URLs and remote URLs.
- Android only loads URLs beginning with `http`.
- React Native and Flutter return `null` / empty output if the required source data is missing.

Minimal JSON example:

```json
{
  "type": "image",
  "properties": {
    "source": "network",
    "url": "https://example.com/image.png"
  }
}
```

Full JSON example:

```json
{
  "type": "image",
  "properties": {
    "source": "asset",
    "path": "assets/onboarding/hero.png",
    "width": 240,
    "height": 180,
    "fit": "contain",
    "blurhash": "LEHV6nWB2yk8pyo0adR*.7kCMdnj"
  }
}
```

## lottie

Description: Lottie animation slot.

Rendering intent: Play one animation from a remote URL or local asset/path.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `source` | string | effectively yes for React Native / Flutter | none | `network` or `asset`. |
| `url` | string | conditional | none | Used for network sources. |
| `path` | string | conditional | none | Used for local sources. |
| `width` / `height` | number or string | no | none | Explicit size. |
| `fit` | string | no | contain on React Native / Flutter | Android does not map it. |
| `loop` | boolean | no | `false` | Repeat animation. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native and Flutter render real Lottie animations.
- Android renders `LottieAnimationView`.
- iOS currently renders a placeholder label that says `Lottie`.

Layout behavior:

- Explicit width / height are portable.

Interaction behavior:

- No user interaction.

Known SDK differences:

- iOS has no real Lottie playback in the current renderer.
- Android accepts remote URLs and local animation names / paths.

Minimal JSON example:

```json
{
  "type": "lottie",
  "properties": {
    "source": "network",
    "url": "https://example.com/anim.json"
  }
}
```

Full JSON example:

```json
{
  "type": "lottie",
  "properties": {
    "source": "asset",
    "path": "animations/confetti.json",
    "width": 160,
    "height": 160,
    "fit": "contain",
    "loop": true
  }
}
```

## grid

Description: Multi-column layout component with poor runtime parity.

Rendering intent: Equal-width tiled layout.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `children` | array | no | `[]` | Grid items. |
| `crossAxisCount` | number | no | `2` | Used by iOS / Android dedicated grid rendering. |
| `spacing` | number | no | `8` | Used by iOS / Android dedicated grid rendering. |
| `mainAxisSpacing` | number | no | `spacing` | Dedicated React Native / Flutter grid code exists but is bypassed by stack normalization. |
| `crossAxisSpacing` | number | no | `spacing` | Same note as above. |
| `childAspectRatio` | number | no | `1` | Same note as above. |

Display behavior:

- iOS renders rows of equally sized cells with one spacing value.
- Android renders a `GridLayout`.

Layout behavior:

- React Native and Flutter normalize literal runtime `grid` into `stack` before rendering, so the dedicated runtime grid behavior is not reached there.

Interaction behavior:

- None.

Known SDK differences:

- Web Preview supports draft `layout.direction = "grid"` with CSS grid behavior.
- Literal runtime `grid` is not parity-safe across app SDKs.

Minimal JSON example:

```json
{
  "type": "grid",
  "children": []
}
```

Full JSON example:

```json
{
  "type": "grid",
  "properties": {
    "crossAxisCount": 2,
    "spacing": 12,
    "childAspectRatio": 1.1
  },
  "children": [
    {
      "type": "text",
      "properties": {
        "text": "One"
      }
    },
    {
      "type": "text",
      "properties": {
        "text": "Two"
      }
    }
  ]
}
```

## fake_progress_bar / fake_loader

Description: Animated fake loader.

Rendering intent: Animate a progress UI, then fire the component `action` when complete.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `type` | string | yes | none | Runtime uses `fake_progress_bar`. Draft alias is `fake_loader`. |
| `progressColor` | color | no | blue fallback | Progress fill color. |
| `backgroundColor` | color | no | gray fallback | Track color. |
| `thickness` | number | no | `4` | Bar thickness or circular stroke width. |
| `borderRadius` | number | no | `0` | Linear bar corner radius. |
| `duration` | number | no | SDK-specific | React Native / Flutter treat it as milliseconds. iOS and Android code use seconds-scale defaults. |
| `startDelay` | number | no | `0` | Same unit mismatch as `duration`. |
| `style` | string | no | `linear` | `circular` is implemented on React Native / Flutter / iOS. Android ignores it. |
| `size` | number | no | `50` | Circular diameter. |
| `showProgressText` | boolean | no | `false` | Show progress label. |
| `progressTextFormat` | string | no | `{{progress}}%` | React Native / Flutter only replace the `{{progress}}` token. Android / iOS use text resolution paths. |
| `progressTextStyle` | object | no | empty object | React Native / Flutter only. |
| `action` | string | no | none | Fired on completion. |
| `actionData.screenId` | string | conditional | none | Needed for `jump_to` / `jumpTo`. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native and Flutter support linear and circular variants.
- iOS supports linear and circular, but circular text is not rendered.
- Android always renders a linear bar.

Layout behavior:

- Linear mode stretches to full width on React Native / Flutter / Android.
- Circular mode uses `size`.

Interaction behavior:

- Starts automatically on mount / attach.
- Fires the component `action` on completion.

Known SDK differences:

- Android multiplies `duration` and `startDelay` by `1000`, so authored numeric values behave like seconds there.
- iOS linear progress text jumps straight to the resolved `progress = 100` at animation time.
- iOS and Android ignore `progressTextStyle`.

Minimal JSON example:

```json
{
  "type": "fake_progress_bar",
  "action": "next"
}
```

Full JSON example:

```json
{
  "type": "fake_progress_bar",
  "properties": {
    "style": "circular",
    "size": 64,
    "progressColor": "0xFF2563EB",
    "backgroundColor": "0xFFE2E8F0",
    "thickness": 6,
    "duration": 3000,
    "startDelay": 250,
    "showProgressText": true,
    "progressTextFormat": "{{progress}}%"
  },
  "action": "next"
}
```

## slider

Description: Paged horizontal or vertical carousel.

Rendering intent: Page through stack children and optionally link an indicator.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `children` | array | yes for useful output | `[]` | Runtime contract wants `stack` pages. App renderers normalize legacy non-stack children into stack pages. |
| `direction` | string | no | `horizontal` | `vertical` is also supported. |
| `size.width` / `size.height` | string | no | `fill` / `fit` parity helpers on React Native / Flutter / iOS | Android does not match these rules. |
| `width` / `height` | number or string | no | none | Explicit outer slider size. |
| `pageAlignment` | string | no | `center` | React Native / Flutter / iOS. Android does not implement start/end parity. |
| `pageSpacing` | number | no | `16` | Gap between pages. |
| `pagePeek` | number | no | `16` | Visible edge peek. |
| `interaction.loop` | boolean | no | `false` | React Native / Flutter only. |
| `interaction.autoAdvance` | boolean | no | `false` | React Native / Flutter only. |
| `interaction.autoAdvanceIntervalMs` | number | no | `4000` | React Native / Flutter only. |
| `appearance.fill` | mixed | no | none | React Native only, used as background color extraction. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |
| `_internalId` / `id` | string | no | generated upstream | Used as the slider registry key. |

Display behavior:

- Children are rendered page by page.
- Indicators subscribe to the slider registry, keyed by `_internalId` first, then `id`.

Layout behavior:

- React Native / Flutter / iOS implement the page-peek and fit-size parity rules.
- Android uses a simpler `ViewPager2` layout with symmetric padding.
- Child page `stack.properties.size.width` / `height` participate in page sizing on React Native / Flutter / iOS. Android does not implement the same page sizing rules.

Interaction behavior:

- Swiping changes the current page.
- React Native / Flutter optionally auto-advance and loop.

Known SDK differences:

- Android has no current loop or auto-advance support.
- Android defaults the slider height to about `160` when no explicit height is present.
- `appearance.fill` is only read by the React Native renderer.

Minimal JSON example:

```json
{
  "type": "slider",
  "children": [
    {
      "type": "stack",
      "properties": {
        "axis": "vertical"
      },
      "children": []
    }
  ]
}
```

Full JSON example:

```json
{
  "type": "slider",
  "_internalId": "hero_slider",
  "properties": {
    "direction": "horizontal",
    "size": {
      "width": "fill",
      "height": "fit"
    },
    "pageAlignment": "center",
    "pageSpacing": 12,
    "pagePeek": 24,
    "interaction": {
      "loop": true,
      "autoAdvance": true,
      "autoAdvanceIntervalMs": 4000
    }
  },
  "children": [
    {
      "type": "stack",
      "properties": {
        "axis": "vertical",
        "size": {
          "width": "fill",
          "height": "fit"
        }
      },
      "children": [
        {
          "type": "text",
          "properties": {
            "text": "Page 1"
          }
        }
      ]
    },
    {
      "type": "stack",
      "properties": {
        "axis": "vertical",
        "size": {
          "width": "fill",
          "height": "fit"
        }
      },
      "children": [
        {
          "type": "text",
          "properties": {
            "text": "Page 2"
          }
        }
      ]
    }
  ]
}
```

## pageview_indicator

Description: Dot indicator linked to a slider.

Rendering intent: Show the active page for one slider registry key.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `linkedTo` | string | yes for useful output | none | Must match the slider registry key. In exported runtime JSON, that is usually the slider `_internalId` carried from the draft component ID. |
| `activeColor` | color | no | blue fallback | Active dot color. |
| `inactiveColor` | color | no | gray fallback | Inactive dot color. |
| `dotSize` | number | no | `8` | Dot diameter. |
| `spacing` | number | no | `8` | Gap between dots. |
| `style` | string | no | `dots` | Only dots are currently rendered. |
| `alignment` | string | no | none | Allowed by contract, ignored by app renderers. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- Renders dots only.
- Hidden when no slider is linked or when the slider has `count <= 1`.

Layout behavior:

- Uses the slider registry state, not direct child inspection.

Interaction behavior:

- None.

Known SDK differences:

- Dashboard Web Preview renders a "No linked slider found" message when the link is missing.
- App SDKs render nothing in that case.

Minimal JSON example:

```json
{
  "type": "pageview_indicator",
  "properties": {
    "linkedTo": "hero_slider"
  }
}
```

Full JSON example:

```json
{
  "type": "pageview_indicator",
  "properties": {
    "linkedTo": "hero_slider",
    "activeColor": "0xFF2563EB",
    "inactiveColor": "0xFFE2E8F0",
    "dotSize": 10,
    "spacing": 10,
    "style": "dots"
  }
}
```

## icon

Description: Font Awesome style icon slot.

Rendering intent: Render one named icon.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `icon` | string | yes in the v1 contract | none | `name` is also accepted as a legacy alias. |
| `name` | string | no | none | Legacy alias for `icon`. |
| `style` | string | no | `solid` | Font Awesome family / variant on React Native / Flutter / Web Preview. |
| `size` | number | no | `24` | Icon size. |
| `color` | color | no | black | Icon color. |
| `margin` | number or insets object | no | `0` | Standard wrapper margin. |

Display behavior:

- React Native / Flutter / Web Preview attempt to resolve a real icon glyph.
- iOS and Android currently render the icon name or code as plain text.

Layout behavior:

- Intrinsic-size leaf component.

Interaction behavior:

- None.

Known SDK differences:

- React Native falls back to `?`.
- Flutter falls back to `help_outline`.
- iOS / Android are not true icon renderers yet.

Minimal JSON example:

```json
{
  "type": "icon",
  "properties": {
    "icon": "check"
  }
}
```

Full JSON example:

```json
{
  "type": "icon",
  "properties": {
    "icon": "bolt",
    "style": "solid",
    "size": 28,
    "color": "0xFF2563EB"
  }
}
```

## radar_chart

Description: Radar/spider chart.

Rendering intent: Render multiple numeric datasets across named axes.

| Property | Type | Req | Default | Notes |
| --- | --- | --- | --- | --- |
| `axes[]` | array | yes | none | At least 3 axes are required for visible output. |
| `axes[].label` | string | yes | none | Axis label. |
| `axes[].maxValue` | number | no | auto-expanded from datasets | React Native / Flutter auto-normalize axis maximums upward. |
| `datasets[]` | array | yes | none | Current parity problem area. |
| `datasets[].label` | string | no | `Dataset N` | Dataset legend label. |
| `datasets[].data` | array | no | none | Used by React Native / Flutter / Web Preview. |
| `datasets[].values` | array | no | none | Used by current iOS / Android implementations instead of `data`. |
| `datasets[].borderColor` | color | no | blue fallback | Used by React Native / Flutter / Web Preview. |
| `datasets[].strokeColor` | color | no | blue fallback | Used by current iOS / Android implementations instead of `borderColor`. |
| `datasets[].fillColor` | color | no | none | Fill polygon color. |
| `datasets[].borderWidth` | number | no | `2` | Stroke width. |
| `datasets[].borderStyle`, `datasets[].dashArray`, `datasets[].dashLength` | mixed | no | solid | React Native / Flutter / Web Preview only. |
| `datasets[].showPoints`, `datasets[].pointRadius`, `datasets[].pointColor` | mixed | no | visible, `4`, border color | React Native / Flutter / Web Preview only. |
| `backgroundColor` | color | no | transparent | React Native / Flutter / Web Preview. |
| `width` / `height` | number or string | no | width auto, height `300` React Native, `220` iOS / Android | |
| `padding` | insets object | no | `0` | React Native / Flutter / Web Preview. |
| `gridLevels`, `gridColor`, `gridStrokeWidth`, `axisColor`, `axisStrokeWidth` | mixed | no | platform defaults | React Native / Flutter / Web Preview only. |
| `axisLabelStyle`, `labelOffset`, `autoFitLabels` | mixed | no | platform defaults | React Native / Flutter / Web Preview only. |
| `showLegend`, `legendPosition`, `legendSpacing`, `legendStyle` | mixed | no | visible bottom legend | React Native / Flutter / Web Preview only. |
| `shape` | string | no | `polygon` | React Native / Flutter / Web Preview only. |
| `animate`, `animationDurationMs` | mixed | no | `true`, `800` | React Native / Flutter / Web Preview only. |

Display behavior:

- React Native / Flutter / Web Preview have the richer custom chart implementation.
- iOS / Android draw a basic polygon chart with much less styling.

Layout behavior:

- Needs at least 3 axes.

Interaction behavior:

- None.

Known SDK differences:

- The current exported runtime format uses `datasets[].data` and `borderColor`, but current iOS / Android renderers read `datasets[].values` and `strokeColor`.
- This means canonical runtime-exported radar datasets are currently not parity-safe on iOS / Android without redundant compatibility fields.

Minimal JSON example:

```json
{
  "type": "radar_chart",
  "properties": {
    "axes": [
      { "label": "A" },
      { "label": "B" },
      { "label": "C" }
    ],
    "datasets": [
      {
        "label": "Score",
        "data": [30, 60, 90]
      }
    ]
  }
}
```

Full JSON example:

```json
{
  "type": "radar_chart",
  "properties": {
    "height": 280,
    "axes": [
      { "label": "Energy", "maxValue": 100 },
      { "label": "Strength", "maxValue": 100 },
      { "label": "Sleep", "maxValue": 100 },
      { "label": "Mood", "maxValue": 100 }
    ],
    "datasets": [
      {
        "label": "Current",
        "data": [72, 64, 80, 68],
        "values": [72, 64, 80, 68],
        "borderColor": "0xFF2563EB",
        "strokeColor": "0xFF2563EB",
        "fillColor": "0x332563EB",
        "borderWidth": 2,
        "showPoints": true,
        "pointRadius": 4,
        "pointColor": "0xFF2563EB"
      }
    ],
    "showLegend": true,
    "legendPosition": "bottom",
    "shape": "polygon"
  }
}
```

## 3. Verified source files

Primary validation and runtime mapping:

- `flowboard-rest-api/services/ai/lib/contracts/supported-components.v1.json`
- `flowboard-rest-api/services/ai/lib/validation/runtimeValidator.js`
- `flowboard-rest-api/services/ai/lib/validation/draftProjectValidator.js`
- `flowboard-rest-api/services/ai/lib/runtime/draftFlowJsonBuilder.js`
- `flowboard-rest-api/services/ai/lib/catalog/canonicalizeRuntimeScreen.js`

React Native / Expo:

- `flowboard-pckg-react/src/components/FlowboardRenderer.tsx`
- `flowboard-pckg-react/src/components/FlowboardFlow.tsx`
- `flowboard-pckg-react/src/utils/flowboardUtils.ts`
- `flowboard-pckg-react/src/components/layout/stackOverlayModel.ts`

Flutter:

- `flowboard-pckg-flutter/lib/src/flowboard_ui.dart`
- `flowboard-pckg-flutter/lib/src/flowboard_flow.dart`

iOS / Swift:

- `flowboard-ios/Sources/FlowboardUIKit/FlowboardRendererView.swift`
- `flowboard-ios/Sources/FlowboardUIKit/FlowboardRendererParity.swift`
- `flowboard-ios/Sources/FlowboardUIKit/FlowboardFlowViewController.swift`
- `flowboard-ios/Sources/FlowboardCore/FlowboardExpressionResolver.swift`

Android / Kotlin:

- `flowboard-android/flowboard-view/src/main/kotlin/co/flowboard/view/FlowboardRendererView.kt`
- `flowboard-android/flowboard-view/src/main/kotlin/co/flowboard/view/FlowboardFlowView.kt`
- `flowboard-android/flowboard-core/src/main/kotlin/co/flowboard/core/FlowboardExpressionResolver.kt`

Web Preview:

- `flowboard-dashboard/components/editor/DraggableComponent.tsx`
- `flowboard-dashboard/components/editor/stackOverlayModel.ts`
- `docs/stack-overlay-parity.md`
