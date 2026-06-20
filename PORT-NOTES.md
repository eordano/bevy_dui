# bevy_dui: Bevy 0.18 -> 0.19 port notes

Branch: `0.19` (forked from `0.18`).
Target: `bevy = "0.19.0"` from crates.io.

Build:

```
cargo build
```

## Dependency changes (Cargo.toml)

- `bevy`: `0.18.1` -> `0.19.0` (crates.io), for both the normal and the dev dependency.
- `bevy_ecss`: unchanged path dependency `{ path = "../bevy_ecss-fork" }`. The
  `bevy_ecss-fork` must be on its `0.19` branch (also bumped to `bevy = 0.19.0`).
  See the dependency note at the bottom.

## Source changes

### `examples/component_ui.rs` — text engine (Cosmic Text -> Parley)

The only source break in `bevy_dui` itself. Bevy 0.19 migrated `bevy_text` from
Cosmic Text to Parley and restructured `TextFont`:

- `TextFont::font` changed from `Handle<Font>` to `FontSource` (variants
  `FontSource::Handle(Handle<Font>)` and `FontSource::Family(..)`).
- `TextFont::font_size` changed from `f32` to `FontSize` (variants `Px`, `Vw`,
  `Vh`, `VMin`, `VMax`, plus an `Rem`-style one).

The example constructed a `TextFont` literal, so it now reads:

```rust
font: FontSource::Handle(ctx.asset_server().load("fonts/FiraSans-Bold.ttf")),
font_size: FontSize::Px(20.),
```

`FontSource` and `FontSize` are in `bevy::prelude` (re-exported from
`bevy_text::prelude`), and the example already does `use bevy::prelude::*`, so no
new import was needed.

### `src/lib.rs` — no changes

The crate library required no edits. As in the previous ports, `bevy_dui`
manipulates almost all UI/text components reflectively
(`Box::<T>::default().into_reflect()`, `PartialReflect::apply`, `insert_reflect`,
`reflect_clone`), so concrete UI/text field-shape changes (including the
`TextFont`/`FontSize`/`FontSource` restructuring) are absorbed inside `bevy_ecss`'s
property impls rather than here. The font/text style properties live entirely in
`bevy_ecss::property::impls`.

## Breaks that did NOT surface in `bevy_dui` (and why)

The task anticipated several 0.18 -> 0.19 breaks. Each was checked against the
bevy 0.19.0 crate sources and confirmed to need no edit in `bevy_dui`:

- **Text engine (Cosmic Text -> Parley)** — surfaced only in the example (above).
  `src/lib.rs` touches `Text` (still `Text(pub String)`) and inserts text
  reflectively; the `TextFont` shape change is handled in `bevy_ecss`.
- **Resources are now Components / `#[derive(Resource)]` implies `Component`** —
  `DuiRegistry` is `#[derive(Resource)]` and is inserted/read via the normal
  `init_resource` / `Res` / `ResMut` APIs, all of which still work. `bevy_dui`
  never derives both `Component` and `Resource` on one type, never inserts
  duplicate resource copies, and uses no non-send-resource APIs, so the
  resource-as-component shift is transparent here.
- **`register_type` requirements for reflective inserts** — the existing
  `register_type::<{Inherited,View}Visibility>()` / `register_type::<Visibility>()`
  calls in `DuiPlugin::build` still apply and still compile. No `#[reflect(...)]`
  macro-syntax change was needed.
- **`BorderRadius` / `Node`** — already resolved in the 0.18 port (`BorderRadius`
  is a field on `Node`; `Node`'s default carries `BorderRadius::DEFAULT`). The new
  0.19 `Node::direction: InlineDirection` field defaults to `Ltr` and is covered by
  `Node`'s `Default`, which is what `bevy_dui` inserts reflectively — no change.
- **`BackgroundColor` / `BorderColor`** — `BackgroundColor(pub Color)` is unchanged
  and `BorderColor` still has the blanket `impl<T: Into<Color>> From<T>`, so the
  existing `BackgroundColor::from(Color::NONE)` / `BorderColor::from(Color::NONE)`
  reflective inserts compile unchanged.
- **`Mut::new` / change detection** — the `apply_prop!` macro calls
  `Mut::new(value, &mut Tick, &mut Tick, Tick, Tick, MaybeLocation<&mut &Location>)`.
  Although `bevy_ecs::change_detection` became a module directory in 0.19 and `Mut`
  now stores a `ComponentTicksMut`, the `Mut::new` signature is byte-for-byte the
  same, and `Tick` / `MaybeLocation` are still re-exported from
  `bevy::ecs::change_detection`. No change.
- **`AssetLoader` / `AssetPath`** — the `AssetLoader` trait signature
  (`load(&self, &mut dyn Reader, &Self::Settings, &mut LoadContext) -> impl
  ConditionalSendFuture`) is unchanged; `type Error: Into<BevyError>` accepts
  `anyhow::Error`. `bevy_dui` only uses `asset_server.load(path)` and
  `context.path()`, not the changed `AssetPath::resolve` / `get_full_extension`
  APIs, so nothing surfaced.
- **`MessageReader<AssetEvent>`** — the `add_duis` system reads asset events via
  `MessageReader<AssetEvent<DuiNodeList>>`; unchanged.

## Dependency: bevy_ecss 0.19

`bevy_dui` hard-depends on `bevy_ecss` internals, so it builds against
`bevy_ecss-fork` on its `0.19` branch (bevy 0.19.0). That branch is ported in
parallel and absorbs the heavy 0.18 -> 0.19 breaks (the Cosmic Text -> Parley
`FontSource`/`FontSize` font property changes in `property/impls.rs`, plus
`SystemParam`/`Query` API changes in `system.rs` such as the now-`Result`
`prepare_state` query state and the `StyleSheet` query helpers). This `bevy_dui`
0.19 port was verified green against that `bevy_ecss-fork@0.19` working tree.

NOTE at time of this port: the parallel `bevy_ecss-fork@0.19` source edits were
present in the working tree and compiled, but had **not yet been committed** on
that branch (HEAD was still the 0.18 port commit). The `bevy_dui` 0.19 build is
green against that tree; if `bevy_ecss-fork` is later reset/recommitted, re-run the
build to confirm the dependency is still satisfied.

## Status

Green. `cargo build` and `cargo build --all-targets` both succeed with zero errors
and zero warnings against `bevy = 0.19.0` and `bevy_ecss-fork@0.19`.
