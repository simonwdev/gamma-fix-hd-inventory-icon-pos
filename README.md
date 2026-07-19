# Fix HD Inventory Icon Position (GAMMA)

Per-mod fixes for HD icons (`inv_grid_scale`, from HD_Inventory_Icons_Framework)
being drawn or placed wrong in the inventory:

- **464- Inventory Open Lag Reducer** — sorted icons no longer claim double-size tiles
- **110- SortingPlus** — favourite/junk marks sit on the icon again; items taken
  or dropped while looting land on the right rows
- **468- Inventory Antifreeze** — no more sort errors on items added to big
  inventories mid-sort
- **Looting Takes Time REDUX** — corpse loot grids pack tightly, no cell-wide gaps

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

## Files

| File | Role |
|------|------|
| `gamedata/scripts/seax_sortingplus_opt_sort_by_kind.script` | Overrides the copy from mod 464. Scale/override-aware footprints, SortingPlus cache priming, lazy caches, nil-guarded comparator. |
| `gamedata/scripts/zzz_aaa_hd_icon_mark_pos_fix.script` | Standalone. Wraps SortingPlus' `icon_junk`/`icon_favs` at `on_game_start` to correct mark positions. Named `zzz_aaa_*` so it wraps before SortingPlus registers the functors. |
| `gamedata/scripts/zzzz_loot_searching.script` | Patched copy of Looting Takes Time REDUX's script (must win over that mod in MO2). Only `get_sort_info` changed, to scale-correct the precomputed corpse grid. |

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

`seax_sortingplus_opt_sort_by_kind.script` and `zzz_aaa_hd_icon_mark_pos_fix.script`
each have a `local DBG = false` flag near the top. Set it to `true` to log every
footprint calculation, placement, and mark position to the xray log, prefixed
`FIXHD|`. The mark fix script also logs the claimed area of every class-level
`FindFreeCell` call, which identifies misplacements caused by other mods.

## Resilience

All cross-mod references are guarded. If SortingPlus, the lag reducer, the HD
framework, `magazine_binder`, or the exe-side unlocalizer (which exposes
SortingPlus' local `favorite_itms`/`junk_itms`/`item_order`) are missing, the
scripts no-op or degrade to a plain size/alphabetical sort instead of erroring.
