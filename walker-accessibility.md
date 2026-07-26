# Walker accessibility surface

Cross-cutting rules for **what an app must expose** so the semantic walker can find and tap UI elements. These are app-side obligations — the walker reads what the platform's accessibility tree contains.

Reference implementations: `ddb/agent/`, `idb/agent/`. Authoritative agent contract: [tctl/docs/agent-porting-guide.md](https://github.com/llm-actuators/tctl/blob/main/docs/agent-porting-guide.md).

## The general rule

The walker enumerates the platform's **accessibility tree**, not the visual view tree. If an element is rendered on screen but not exposed to accessibility, the walker can't see it. VoiceOver users have the same problem.

When a TC fails with "element not found" on an element you can see in the UI, the first question is: **is this element in the accessibility tree?** Probe with `idb ui` (iOS) or `ddb ui` (Android) before reaching for walker scope-widening or selector tweaks.

## SwiftUI patterns — iOS

### 1. Custom button wrappers eat `.accessibilityLabel`

`PhoneNumberLink`, `NKButton`, and similar custom Button-like SwiftUI views often run iOS's automatic-content-detection or wrap with `Button(...)` internally. The implicit auto-labeling **replaces** the subtree's accessibility content, ignoring `.accessibilityLabel`.

Wrong (label silently ignored):
```swift
Button(action: ...) {
    Text(formattedPhone)
}
.accessibilityLabel(Text(rawPhone))
```

Right — replace the subtree with `.accessibilityRepresentation`:
```swift
Button(action: ...) {
    Text(formattedPhone)
}
.accessibilityRepresentation {
    Button(rawPhone) { /* same action */ }
}
```

`.accessibilityRepresentation` substitutes a virtual subtree the auto-detector can't override. Use it whenever a custom button wraps complex content and you need a specific accessibility label.

### 2. `HStack + .onTapGesture` is not a Button

Walker matchers find the inner Text, but tapping the StaticText doesn't fire `onTapGesture`. The TC reports the tap "passed" (walker dispatched it) but no navigation happens.

Wrong:
```swift
HStack {
    Image(systemName: "...")
    Text("Answer")
}
.onTapGesture { actionAnswer() }
```

Right — make it a real Button:
```swift
Button(action: actionAnswer) {
    HStack {
        Image(systemName: "...")
        Text("Answer")
    }
}
```

Or, if Button styling is undesirable, expose the HStack as a Button-typed accessibility element:
```swift
HStack { ... }
.onTapGesture { actionAnswer() }
.accessibilityElement(children: .combine)
.accessibilityLabel(Text("Answer"))
.accessibilityAddTraits(.isButton)
```

### 3. `Text` inside conditional render gates may be missing entirely

If a Text element doesn't appear in the accessibility tree, check whether the render gate (`if let foo = bar { Text(...) }`) is actually true at probe time. A SwiftUI conditional that evaluates `false` produces no element — visible or not. Verify via the underlying data binding (e.g. `organization.contactPhone` is nil despite the mock providing it) before assuming a walker problem.

The walker can only expose what's actually in the tree. If the conditional is false, the fix is app-side data flow, not toolchain.

## Off-viewport carousel content

`LazyHStack`, horizontally-scrolling `UICollectionView`, and similar lazy-rendered containers only materialize their visible cells. Off-viewport siblings literally don't exist in the accessibility tree until they're scrolled into view.

The walker handles this by **programmatically scrolling** horizontal UIScrollView containers during the walk, capturing accessibility children at each offset, and restoring the original offset. See `idb/agent/Sources/SemanticAgent/SemanticViewWalker.swift` (`walkHorizontalCarousels`). The criterion: any `UIScrollView` with `contentSize.width > bounds.width + 1`.

Side effect: the carousel briefly flashes during walker calls. Cosmetic, not functional. ~50-200ms latency overhead per carousel.

## App-side a11y vs walker scope-widening

When a TC fails because the walker can't find an element, prefer the app-side a11y fix over a toolchain widening:

- App-side: 1-3 LOC of `.accessibilityLabel` / `.accessibilityIdentifier` / `.accessibilityRepresentation`. Surface-specific, low blast radius, lands in PR-sized changes.
- Toolchain widening: 50-200+ LOC of walker recursion, applies to every project consuming the binary, edge cases proliferate.

The app-side path is cheaper in nearly every case. The toolchain path is correct when:
- The element is in the accessibility tree but the walker isn't reading the right surface (TD-196 horizontal carousels).
- The platform exposes the element to VoiceOver but not via the API the walker currently uses.
- The fix requires changes across many apps consuming the binary, not just one.

Add a TD entry to `tctl/docs/tech-debt.md` when you propose a walker change; reach for app-side a11y first.

## Cross-references

- [agent-contract.md](agent-contract.md) — HTTP contract every in-app agent implements
- [device-control.md](device-control.md) — common verb surface across `ddb`/`idb`/`wdb`
- [tctl/docs/agent-porting-guide.md](https://github.com/llm-actuators/tctl/blob/main/docs/agent-porting-guide.md) — authoritative agent + walker contract
- [tctl/docs/tech-debt.md](https://github.com/llm-actuators/tctl/blob/main/docs/tech-debt.md) — TD ledger including TD-196 (LazyHStack), TD-197 (walker accessibility scope), TD-191 (viewport-clip)
