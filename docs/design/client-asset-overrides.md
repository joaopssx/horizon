# Content mods that replace a vanilla item's model definition

Status: proposal. A content mod may ship `assets/minecraft/items/<item>.json` for the vanilla items its manifest
lists, so a crossbow can show the arrow it is loaded with. Data only: no code, no new file, network or clock access.

## What exists already (reused, not rebuilt)

- **Layering.** `mod_content::build` (horizon-client) builds the content root as an overlay of the prepared vanilla
  root: a mod's file is copied in by name over vanilla's (`Overlay::copy`). Resource packs are layered over that root
  (`game/packs.rs`, `packs::resource_root`). With no content mod `mod_content::resource_root` returns the vanilla root
  itself, so nothing changes for a player without mods.
- **Item definitions.** `item_render::load_definitions` reads every `assets/<ns>/items/*.json` from the root it is given;
  `horizon_assets::item_model` decodes them (26.3's `select`, `condition`, `range_dispatch`, tints...).
- **The only gap is the fence.** `horizon_mods::content::Content::check` refuses any `assets/minecraft/` file but
  `atlases/`. The mod's own models and textures (`assets/<id>/models/`, `textures/`) already load.

## The change

Manifest, one new optional key (unknown keys stay refused):

```toml
side = "client"
item_models = ["minecraft:crossbow", "minecraft:bow"]   # vanilla items whose definition this mod replaces
```

Rules, checked by `Content::check` (so `cargo horizon-mod build`/`check` and the game refuse the same things):

1. Each entry is a `minecraft:` item id; the list has no duplicates. `side = "server"` may not list any.
2. A listed id needs `assets/minecraft/items/<path>.json` in the mod, and that file needs its id listed. Every other
   `assets/minecraft/` file is still refused (paths keep `check_path`: no `..`, vanilla resource characters).
3. The file is at most 256 KiB and is a JSON object with a `model` object (serde_json's depth limit stops deep nesting).
4. In the game (`mod_content::accept_mod`): the file decodes with `ClientItem::from_json`, and the base root has the
   vanilla definition it replaces (a real vanilla item). Otherwise the mod is refused with the file and the error, on
   its row in Options > Mods, and the other mods load.

## Precedence and conflicts

vanilla < content mods < resource packs (the player's, then a server's). A resource pack the player picked still
wins, as packs win over vanilla. Two mods that are on and list the same item clash as two mods sharing a namespace do:
the first in load order loads, the other is left out with "mods A and B both replace the item model of
minecraft:crossbow; turn one of them off".

## How the player sees and turns it off

Like any content mod: off by default, switched in Options > Mods. Item definitions are assets, so `live_switch` lets the
switch apply at once through the resource reload; off, the root has no mod file and vanilla's definition is back.

## What a definition can select on (Horizon's 26.3 port, `item_model.rs`)

`select` on `minecraft:charge_type` (`none`, `arrow`, `rocket`); `range_dispatch` on `minecraft:crossbow/pull` and
`minecraft:use_duration`; `condition` on `minecraft:using_item`; `condition` on `minecraft:component` with an
`enchantments` predicate (Multishot); `select` on `minecraft:component` with `component =
"minecraft:charged_projectiles"`, whose cases are whole component values (the projectile list). No property exposes
"the loaded arrow's potion colour": a `minecraft:potion` tint reads the crossbow's own stack. A tipped tip is one case
per potion value with a `minecraft:constant` tint; a custom `custom_color` potion falls to the fallback.

## Files

`crates/horizon-mods/src/{manifest.rs,content.rs}` (field, fence, checks, tests); `crates/horizon-client/src/
mod_content.rs` and `mod_content_tests.rs` (decode, base check, clash, parity test); `docs/mods.md`; a test fixture
mod with one original texture. `docs/api/mod-index.md` and the index stay unchanged (no new index field).
Estimate: about 150 lines of code and 200 of tests.

## Not in this change

Other vanilla asset kinds (models, textures, sounds, lang); new item model properties; showing the replaced items on
the Mods page (a follow-up with translation keys).
