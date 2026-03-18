# Changelog

## [0.3.0] - 2026-03-18

### Fixed

- **`bindList` correctness (two bugs)**:

  1. **Stale child content after container rebuild.** When a list's `content` array was updated (item added or removed), child parts were re-applied from their build-time specs — overwriting any `updateParts` changes made after mount. `bindList` now fully remounts all child components on every structural change, so each child's `build()` runs with current-state props and initial content is always correct.

  2. **Listener firing on every action.** The internal key selector used `.map(keyFn)`, which always produces a new array reference. `Object.is` equality (the default) therefore always returned `false`, causing `bindList` to rebuild the container on every Redux action regardless of whether the list actually changed. The listener now uses element-wise array comparison and only fires when the key sequence genuinely differs.

- **`bindList` unmount leak.** Child components were not cleaned up when the parent component was unmounted. All `rendered` children are now unmounted as part of parent cleanup.

### Added

- **Optional `equals` parameter on `useSelector`** — mirrors the change in NAIStore 0.3.0. Callers can supply `(a: T, b: T) => boolean` to control when the listener fires. Useful for selectors that return derived arrays or objects where reference equality would cause spurious re-renders.

  ```ts
  ctx.useSelector(
    (state) => state.items.map((item) => item.id),
    (ids) => { /* only fires when the id list actually changes */ },
    (a, b) => a.length === b.length && a.every((k, i) => k === b[i]),
  );
  ```

- **Updated `StoreLike<S>` interface** — `subscribeSelector` now accepts the optional `equals` parameter so that any NAIStore-compatible store implementation exposes the same signature.

---

## [0.2.0] - 2026-03-16

### Added

- **`ctx.bindPart(partId, selector, mapper)`** — Collapses the two-path hydration pattern into one call. Reads current state to produce initial props (returned inline for use in `build()`), and subscribes for subsequent updates via `api.v1.ui.updateParts`. Eliminates the class of bug where initial render and subscription diverge.

- **`ctx.bindList(containerId, selector, keyFn, renderItem)`** — Eliminates manual list reconciliation boilerplate. Renders the initial item list from current state (returned as a `UIPart[]` for use in `build()`), then subscribes to key-array changes: mounts new items, unmounts removed items, and calls `updateParts` on the container with the new order.

- Updated `BindContext<S>` interface with the two new method signatures.

### Notes

- `useSelector` is **not deprecated** — it remains the right tool for non-UIPart side effects (syncing to `api.v1.memory`, `api.v1.an.set()`, external APIs).
- `bindPart` and `bindList` can be adopted incrementally; old patterns continue to work.

## [0.1.0] - Initial release

- `defineComponent`, `mount`, `mergeStyles`
- `StoreLike<S>`, `BindContext<S>`, `Component<P,S,St>`, `Style` types
- Auto-injected `this.style()` resolver when `styles` is defined
- Child lifecycle management via `ctx.render()`
