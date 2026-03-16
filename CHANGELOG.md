# Changelog

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
