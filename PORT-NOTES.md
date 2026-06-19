# bevy_dui: Bevy 0.17 -> 0.18 port notes

Branch: `0.18` (forked from `0.17`).
Target: `bevy = "0.18.1"` from crates.io.

Build:

```
/home/dcl/linux-rigging/dcl-shell -c "cd /home/dcl/bevy_dui-fork && cargo build"
```

## Dependency changes (Cargo.toml)

- `bevy`: `0.17.3` -> `0.18.1` (crates.io), for both the normal and the dev dependency.
- `bevy_ecss`: unchanged path dependency `{ path = "../bevy_ecss-fork" }`. The
  `bevy_ecss-fork` must be on its `0.18` branch (also bumped to `bevy = 0.18.1`).
  See the dependency note at the bottom.

## Source changes (src/lib.rs)

Only one 0.17 -> 0.18 API break surfaced in `bevy_dui` itself:

- **`BorderRadius` is no longer a standalone component.** In 0.18 it became a field
  on the `Node` component (`Node::border_radius: BorderRadius`), and the
  `BorderRadius` type is now only `Reflect` (not `Component` / not
  `reflect(Component)`). `bevy_dui` previously inserted a default `BorderRadius`
  reflectively in `node_from_attrs` ("required for border to render"). That block was
  removed: `Node`'s `Default` already carries `BorderRadius::DEFAULT`, so a div's
  border radius travels with its `Node` and there is nothing extra to insert. Keeping
  the old block would have failed (inserting a non-`Component` reflect value as a
  component).

## Breaks that did NOT surface (and why)

The task anticipated several 0.17->0.18 breaks. None required edits in `bevy_dui`:

- **`LineHeight` removed from `TextFont` / now a separate required component** — not
  referenced anywhere in `bevy_dui`; font/text layout properties are owned by
  `bevy_ecss`'s property impls, handled in the `bevy_ecss` port.
- **Entity events immutable / `SetEntityEventTarget`** — `bevy_dui` does not use
  entity events or observers.
- **`RenderTarget` moved to a required component on the camera** — `bevy_dui` does not
  spawn cameras.
- **`ScrollPosition` / UI tweaks, `BackgroundColor`/`BorderColor`** — `BackgroundColor`
  is still `BackgroundColor(pub Color)` and `BorderColor::from(impl Into<Color>)` still
  exists (blanket `From`), so the existing reflective + `From` usage compiles
  unchanged. `ScrollPosition` is not referenced.
- **Reflect registration** — the explicit `register_type::<{Inherited,View}Visibility>()`
  / `register_type::<Visibility>()` calls added during the 0.17 port still apply; no
  `#[reflect(...)]` macro-syntax changes were needed in this crate.

As in the 0.16->0.17 port, `bevy_dui` manipulates almost all UI components
reflectively (`Box::<T>::default().into_reflect()`, `PartialReflect::apply`,
`insert_reflect`, `reflect_clone`), so concrete UI field-shape changes are absorbed
inside `bevy_ecss` rather than here.

## Dependency: bevy_ecss 0.18

`bevy_dui` hard-depends on `bevy_ecss` internals, so it builds against
`bevy_ecss-fork` on its `0.18` branch (bevy 0.18.1). That branch was ported in
parallel (it dropped 0.18-removed items such as `bevy::ecs::component::ComponentTicks`
and `Tick` moving to `bevy::ecs::change_detection`). This `bevy_dui` 0.18 port was
verified green against the committed `bevy_ecss-fork@0.18`.

## Status

Green. `cargo build` and `cargo build --all-targets` both succeed with zero errors and
zero warnings against `bevy = 0.18.1` and `bevy_ecss-fork@0.18`.
