# bevy_dui: Bevy 0.16 -> 0.17 port notes

Branch: `0.17` (forked from `0.16`).
Target: `bevy = "0.17.3"` from crates.io.

Build (clean, zero warnings):

```
/home/dcl/linux-rigging/dcl-shell -c "cd /home/dcl/bevy_dui-fork && cargo build"
```

## Dependency changes (Cargo.toml)

- `bevy`: switched from the robtfm `release-0.16-dcl` git fork to `version = "0.17.3"`
  (crates.io), for both the normal and dev dependency.
- `bevy_ecss`: switched from `git = robtfm/bevy_ecss, branch = "0.16"` to a local
  path dependency `{ path = "../bevy_ecss-fork" }`.

  **Why a path dep / FLAG:** there is *no* 0.17-compatible `bevy_ecss` anywhere.
  Upstream `afonsolage/bevy_ecss` tops out at 0.7.0 (bevy 0.16) on crates.io, and
  robtfm's fork only has a `0.16` branch. `bevy_dui` hard-depends on `bevy_ecss`
  internals (`bevy_ecss::property::impls::*`, the `Property` trait, `PropertyValues`,
  `PropertyToken`, `StyleSheetAsset`, `Selector`, `Class`), so porting `bevy_dui` to
  0.17 required also porting `bevy_ecss` to 0.17. That port lives in
  `/home/dcl/bevy_ecss-fork` (branch `0.17`, version bumped to 0.8.0) and has its own
  PORT-NOTES.md. Before consuming this in bevy-explorer you must decide where the
  `bevy_ecss` 0.17 port should live (publish a branch / git dep) and repoint this
  Cargo.toml accordingly.

## Source changes (src/lib.rs)

Only one API break surfaced in `bevy_dui` itself:

- `EventReader<AssetEvent<DuiNodeList>>` -> `MessageReader<AssetEvent<DuiNodeList>>`
  in `add_duis`. In 0.17 the buffered-event API was renamed (`Event` -> `Message`,
  `EventReader` -> `MessageReader`, etc.). `AssetEvent` is now a `Message`.
  `MessageReader` is re-exported from the bevy prelude, so no import change was needed.

## Breaks that did NOT surface (and why)

The task anticipated a number of 0.16->0.17 UI breaks (`BorderColor` per-side fields,
`ScrollPosition` newtype, text type moves, camera moves to `bevy_camera`,
required-components changes). None of these required edits in `bevy_dui` because the
crate manipulates almost all UI components reflectively
(`Box::<T>::default().into_reflect()`, `PartialReflect::apply`, `insert_reflect`,
`reflect_clone`) rather than via direct field access. The concrete UI field-shape
changes (e.g. `BorderColor`) are absorbed inside `bevy_ecss`'s property
implementations, which is where they were handled during the `bevy_ecss` port. The
only non-reflective component touches in `bevy_dui` are `Text`, `ImageNode`
(`.image`, `.color`), `ZIndex(i32)`, `FocusPolicy`, `Node`, `BorderRadius`,
`BackgroundColor`, `BorderColor`, `Name`, `Interaction` — all of which kept
compatible constructors / `From`/`Default` impls in 0.17.

## Status

Green. `cargo build` succeeds with no errors and no warnings for both `bevy_dui`
and its `bevy_ecss-fork` path dependency.
