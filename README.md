# Fix HD Inventory Icon Position (GAMMA)

Per-mod fixes for HD icons (`inv_grid_scale`, from HD_Inventory_Icons_Framework)
being drawn or placed wrong in the inventory:

- **464- Inventory Open Lag Reducer** — sorted icons no longer claim double-size tiles
- **110- SortingPlus** — favourite/junk marks sit on the icon again; items taken
  or dropped while looting land on the right rows
- **468- Inventory Antifreeze** — no more sort errors on items added to big
  inventories mid-sort
- **Looting Takes Time REDUX** — corpse loot grids pack tightly, no cell-wide gaps
- **HD Attachment Icons For GAMMA** — scoped weapon variants the pack doesn't
  cover now draw its straight gun-mounted scope icons instead of the diagonal
  inventory item icon
- **ilrathCXV's Meat Spoiling Timer in Tooltips** — the "hours until rotten"
  line reappears on raw/cooked meat; the HD framework's own copy of
  `meat_spoiling.script` had dropped it

Each fix disables itself if its mod isn't installed.

## What it fixes in detail

1. Icon tile size after a sorted re-init: the lag reducer's replacement
   `FindFreeCell` claimed unscaled (double-size) tiles for HD icons. Footprints
   are now divided by `inv_grid_scale` and resolved through
   `utils_xml.get_item_axis`, which also picks up `icon_override.ltx` entries.

2. NPC inventory / looting: taking or putting items while looting adds items
   one at a time (`ui_inventory.UIInventory:LMode_RefreshInventories` calls
   `UICellContainer:AddItem` directly) without a full sorted re-init. That path
   uses SortingPlus' own `FindFreeCell`, which places items via its internal
   `ab_k` kind cache. That cache is normally filled as a side effect of the
   SortingPlus sort comparators, but the lag reducer replaces those comparators,
   so the cache stayed empty and items landed on wrong rows. The factory now
   primes `zzz_rax_sortingplus_mcm.sort_info` once per unique section, so the
   cache stays populated. Caches also fill lazily, which keeps Inventory
   Antifreeze's deferred mode (it re-sorts on `AddItem`) from hitting nils.

3. SortingPlus favourite/junk mark icons: their positions were computed from
   the unscaled icon axis, drawing marks about twice too far right and down on
   HD icons. The functors are wrapped to use the rendered icon size. The junk
   mark sits at the right edge, the favourite mark at the left edge (original
   SortingPlus layout).

4. Looting Takes Time REDUX corpse layout: that mod precomputes a stable grid
   layout for corpse items (so items keep their spots while the search reveals
   them) and places them through its own `FindFreeCell` override. Its size
   cache used raw `inv_grid_width/height`, so HD items reserved double-size
   tiles and the loot window showed cell-wide gaps between items on first
   open. This mod ships a patched copy of `zzzz_loot_searching.script` whose
   `get_sort_info` returns rendered cell sizes; nothing else in it changed.

5. Attachment overlays on weapon icons: HD Attachment Icons For GAMMA ships
   straight gun-mounted scope icons as `<scope>_x` sections and rewires the
   weapon variants it covers to use them. Uncovered variants (e.g. the Obrez
   with a PU scope) still drew the scope's inventory item icon, which the HD
   pack draws at an angle, so the overlay looked rotated and misaligned.
   `Create_Layer` is replaced with a version that redirects a layer section
   to its `_x` variant when one exists (also stripping the vanilla
   `wpn_addon_scope_` prefix), shifting coords so the visual center stays
   put. Rotation-fix math and `inv_grid_scale` handling are preserved.

6. Meat spoiling tooltip: the "In-game Hours until Rotten" line comes from
   ilrathCXV's Meat Spoiling Timer via a `ui_item.build_desc_footer` override.
   HD_Inventory_Icons_Framework ships its own `meat_spoiling.script` (it adds an
   `inv_grid_scale` tweak to the frozen-meat icon layer) and wins the load-order
   conflict, but that copy dropped the footer override, so the tooltip vanished.
   Reordering can't fix it: the framework copy must stay on top for its icon
   fix, so this re-adds just the footer. The framework keeps its
   `expiration_table` file-local; we borrow the live reference through its
   public `save_state`, which only writes into the table it's handed. Footer
   layout, colour thresholds and the `cxv_*` strings are the originals (the
   strings still load, only the script was overridden).

## Files

| File | Role |
|------|------|
| `gamedata/scripts/seax_sortingplus_opt_sort_by_kind.script` | Overrides the copy from mod 464. Scale/override-aware footprints, SortingPlus cache priming, lazy caches, nil-guarded comparator. |
| `gamedata/scripts/zzz_aaa_hd_icon_mark_pos_fix.script` | Standalone. Wraps SortingPlus' `icon_junk`/`icon_favs` at `on_game_start` to correct mark positions. Named `zzz_aaa_*` so it wraps before SortingPlus registers the functors. |
| `gamedata/scripts/zzz_aaa_hd_attachment_layer_fix.script` | Standalone. Replaces `UICellItem:Create_Layer` at `on_game_start` to redirect attachment overlays to the HD pack's gun-mounted `_x` icons. |
| `gamedata/scripts/zzzz_loot_searching.script` | Patched copy of Looting Takes Time REDUX's script (must win over that mod in MO2). Only `get_sort_info` changed, to scale-correct the precomputed corpse grid. |
| `gamedata/scripts/zzz_aaa_meat_spoiling_tooltip_fix.script` | Standalone. Re-adds ilrathCXV's `ui_item.build_desc_footer` spoiling-timer line at chunk load (`zzz_` prefix loads it after `ui_item`/`meat_spoiling`), reading the live timer via `meat_spoiling.save_state`. No-ops if `meat_spoiling` isn't loaded. |
| `gamedata/configs/unlocalizers/unlocalizer_fix_hd_icon_pos.ltx` | Exposes SortingPlus' local `favorite_itms`/`junk_itms`/`item_order` to our sort factory. Identical to (and safely additive with) the config mod 464 ships; needed because Inventory Antifreeze activates our `seax_*` script by filename even when 464 itself is disabled. |

## Install

MO2: install as a normal mod named "Fix HD Inventory Icon Position", with
priority below (winning over) "464- Inventory Open Lag Reducer - sea-ex" so
this `seax_sortingplus_opt_sort_by_kind.script` is the one that loads.

For development, `scripts\link-mo2-mod.ps1` creates a junction that exposes the
repo as an MO2 mod, so changes are playable without copying files:

```powershell
.\scripts\link-mo2-mod.ps1 -ModsDir "D:\gamma0.9.5\GAMMA\mods"
```

## Debugging

Every script except the Looting Takes Time copy has a `local DBG = false` flag
near the top. Set it to `true` to log footprint calculations, placements, mark
positions, and attachment layer redirects to the xray log, prefixed `FIXHD|`.
The mark fix script also logs the claimed area of every class-level
`FindFreeCell` call and the inputs/results of every icon layer draw, which
identifies misplacements caused by other mods.

## Resilience

All cross-mod references are guarded. If SortingPlus, the lag reducer, the HD
framework, `magazine_binder`, or the exe-side unlocalizer (which exposes
SortingPlus' local `favorite_itms`/`junk_itms`/`item_order`) are missing, the
scripts no-op or degrade to a plain size/alphabetical sort instead of erroring.

One degradation bit users for real: Inventory Antifreeze soft-detects
`seax_sortingplus_opt_sort_by_kind` by filename, so shipping that script makes
468 route sorting through it even when 464 is disabled — and without 464 no
unlocalizers config loaded, so the guarded fallbacks silently dropped
favourite/kind ordering ("inventory sorts randomly"). This mod now ships its
own copy of the unlocalizers config so sorting works with or without 464, and
the sort factory logs a one-time `! FIXHD|` warning if the SortingPlus locals
are still unreachable.
