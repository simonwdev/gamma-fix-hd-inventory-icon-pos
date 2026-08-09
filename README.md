# Fix HD Inventory Icon Position (GAMMA)

Per-mod fixes for HD icons (`inv_grid_scale`, from HD_Inventory_Icons_Framework)
being drawn or placed wrong in the inventory:

- 464- Inventory Open Lag Reducer: sorted icons no longer claim double-size tiles
- 468- Inventory Antifreeze: items added to a big inventory mid-sort no longer throw sort errors
- 110- SortingPlus: favourite/junk marks sit on the icon again; items taken or dropped while looting land on the right rows; its highlight bookkeeping is back
- Looting Takes Time REDUX: corpse loot grids pack tightly again instead of leaving cell-wide gaps
- HD Attachment Icons For GAMMA: scoped weapon variants the pack doesn't cover now draw its straight gun-mounted scope icons instead of the diagonal inventory item icon
- MartinLore's Stalker 2 icons: the RPK-74 draws its own gun again instead of a garbled crop of unrelated weapons
- ilrathCXV's Meat Spoiling Timer in Tooltips: the "hours until rotten" line is back on raw and cooked meat; meat and patch stacks expand in the picker again, so you can take the freshest piece
- UI Rework G.A.M.M.A. Style - Sota: food tooltips read satiety as a percentage instead of a raw kcal figure; a stack shows the best-condition item on top; carry-weight bonuses keep their decimal, so a 1.58 kg backpack stops reading as 2 kg

Each fix disables itself if its mod isn't installed.

## Load order

Towards the end, high priority. Only two files here can lose a conflict, and both
need to win. `seax_sortingplus_opt_sort_by_kind.script` has to beat `464-
Inventory Open Lag Reducer`, and `zzzz_loot_searching.script` has to beat
`Looting_takes_time_REDUX`. REDUX is the later of the two in a stock GAMMA list,
somewhere around position 830, so anything past it satisfies both. Near the end
of the list is the easy answer.

Nothing else constrains it. The `zzz_aaa_*` scripts, the unlocalizers and the
DLTX config use filenames no other mod ships, so they cannot lose a conflict at
any priority. This mod does not need to win against HD_Inventory_Icons_Framework
or Sota's UI Rework. It never overrides their files at all, and patches their
behaviour at runtime instead.

Two other orderings get confused with MO2 priority. Script execution order is
alphabetical by filename whatever the priority is, which is what the `zzz_aaa_`
and `zzzz_` prefixes are for. DLTX also applies `mod_system_*` patches in
filename order (`FS_FileSet` sorts by name, `Xr_ini.cpp`), which is why the
RPK-74 config carries such a long `z` run.

## The pattern behind most of these

HD_Inventory_Icons_Framework is unmaintained. Seven of the twelve scripts it
overrides are old forks of files whose authors have kept working on them since.
The framework only needed those files for `inv_grid_scale`, but overriding a file
replaces it whole, so anything the real author shipped after the fork is missing
from the running game. Nothing throws an error. Features just stop existing,
which is why they surface one user report at a time.

Reordering doesn't fix this in general, because the framework's copies have to
stay on top for their icon work. Shipping the newer copies from here is no
better: those upstreams are still active, and freezing them in this mod would set
the same trap one level up. So the fixes re-add the dropped behaviour at runtime,
and each one bails out if it finds the real author's copy won the load order
anyway.

## What it fixes in detail

1. Icon tile size after a sorted re-init: the lag reducer's replacement
   `FindFreeCell` claimed unscaled (double-size) tiles for HD icons. This copy
   divides footprints by `inv_grid_scale` and resolves them through
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
   HD icons. This wraps the functors to use the rendered icon size. The junk
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
   to its `_x` variant when one exists, strips the vanilla
   `wpn_addon_scope_` prefix, and shifts the coordinates to keep the visual
   center in place. It keeps the rotation-fix math and the `inv_grid_scale`
   handling.

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

7. RPK-74 icon: MartinLore's Stalker 2 icons moves the RPK family onto the
   `ui\ui_stalker2_rifles` HD atlas by patching the parent section
   (`![wpn_rpk]`, 10x4 at 52,12, `inv_grid_scale = 2`). `[wpn_rpk74]:wpn_rpk`
   inherits that texture and scale, but redeclares its own
   `inv_grid_x/y/width/height`, which are still the legacy `ui_icon_equipment`
   coordinates (5x2 at 35,14). The pack patches `wpn_rpk74_16` and
   `wpn_rpk74_16_drum` explicitly but never plain `wpn_rpk74`, so nothing
   corrects them. The engine therefore samples a 250x100 window at (1750,700)
   out of the 4096x2048 atlas, a region belonging to other weapons, and draws
   it in a `ceil(5/2) x ceil(2/2)` = 3x1 cell. Giving `wpn_rpk74` the parent's
   HD coordinates restores the inheritance the override was breaking. It uses
   `wpn_rpk`'s 10x4 rather than `wpn_rpk74_16`'s 12x4 because that keeps the
   inventory footprint at 5x2 cells, which is what the gun occupied before the
   HD pack.

8. Food satiety shown in kcal instead of %: Sota's UI Rework does this in two
   steps. `sota_stats_tweak_gamma.script` writes `eat_satiety` into the booster
   stats table as `magnitude=1000` / `unit="st_kcal"`, and Sota's own
   `utils_ui.script` converts it back to a percentage at render time inside
   `UIInfoItem:Update`. The HD framework's `utils_ui.script` wins the load-order
   conflict and is an older fork (2025-03-27 against Sota's 2025-09-22) that
   predates the conversion, so step one's kcal reaches the screen. We re-apply
   the same two fields to the table entry after
   `ish_item_stats.override_stats_table` runs. Patching the entry rather than the
   renderer is deliberate. Sota mutates the same shared entry anyway,
   `eat_satiety` has `track=false` so it always takes the text branch where those
   two fields decide the printed unit, and the renderer itself is unreachable
   (`idp_stats_tt` is a function-local).

9. Meat and patch stacks not expanding in the picker: Anomaly only offers the
   per-item picker when a section declares `use_condition`, which meat and
   patches do not. ilrathCXV's `meat_spoiling.script` swaps `ui_inventory`'s
   `SYS_GetParam` for the duration of `Picker_Toggle` so `use_condition` reads
   true for those sections only. The framework's fork of that file dropped it,
   along with the spoiling footer in item 6. This re-adds it, saving and
   restoring the previous `ui_inventory.SYS_GetParam` rather than clearing it to
   nil.

10. Stacks not showing the best-condition item: Sota added
    `UICellItem:FindChildIdByBestCondition` plus promotion logic in `AddChild`
    (a better incoming item takes the top slot, demoting the old one) and
    `ResetToChild` (when the top leaves, the best remaining is promoted). The
    framework's fork predates all three, so a cell shows whichever copy arrived
    first and the condition bar under a stack means nothing. `ResetToChild` is a
    full copy of Sota's body rather than a wrapper, because it picks its child
    mid-function from `pairs(self.childs)` with no seam to hook. That is
    acceptable only because the file being overridden belongs to the unmaintained
    framework.

11. SortingPlus highlight overrides: its copy of `zzz_rax_sortingplus_mcm.script`
    adds a `highlight_set` flag so `UnHighlight_All` can skip its sweep over
    every cell when nothing is highlighted, and clears a cell's highlight in
    `UICellItem:Update` so it cannot outlive a recycled cell. The framework's
    fork has none of the three. This one came out of auditing the forks rather
    than from a reported symptom, so see the file header for what to watch.

12. Carry-weight stats rounded to whole kilos: Sota's `utils_ui.script` forces
    the tooltip decimal-place setting to 1 for `additional_inventory_weight` and
    `boost_max_weight`; the framework's fork predates that, so both fall back to
    the MCM default of 0. `stats_round_idp` already protects values under 1 kg,
    so only 1 kg and up flatten. That is 31 of base Anomaly's 201 non-zero
    values, off by up to 0.5 kg (1.58 renders as 2), plus whatever GAMMA's
    artefact and backpack mods add. The setting itself is a function-local inside
    `UIInfoItem:Update` and unreachable, so this is the one fix that uses the
    engine's unlocalizer. `stats_round_idp` is promoted out of file scope (it is
    a column-0 file-scope local, which is all the `^local` rewrite can reach) and
    wrapped. Knowing which stat is being rounded takes a second wrapper on
    `get_stats_value`, which runs once at the top of each stat-loop iteration,
    ahead of every rounding call in that iteration. Without modded exes the
    unlocalizer is inert, `utils_ui.stats_round_idp` stays nil, and the script
    logs one `! FIXHD|` line and no-ops.

## Files

Three of these replace a file another mod ships, and have to win the MO2
conflict: the `seax_*` script, `zzzz_loot_searching.script`, and
`unlocalizer_fix_hd_icon_pos.ltx`. Everything else uses a filename no other mod
uses.

| File | Role |
|------|------|
| `gamedata/scripts/seax_sortingplus_opt_sort_by_kind.script` | Replaces the copy from mod 464. Scale/override-aware footprints, SortingPlus cache priming, lazy caches, nil-guarded comparator. |
| `gamedata/scripts/zzz_aaa_hd_icon_mark_pos_fix.script` | Wraps SortingPlus' `icon_junk`/`icon_favs` at `on_game_start` to correct mark positions. Named `zzz_aaa_*` so it wraps before SortingPlus registers the functors. |
| `gamedata/scripts/zzz_aaa_hd_attachment_layer_fix.script` | Replaces `UICellItem:Create_Layer` at `on_game_start` to redirect attachment overlays to the HD pack's gun-mounted `_x` icons. |
| `gamedata/scripts/zzzz_loot_searching.script` | Patched copy of Looting Takes Time REDUX's script. Only `get_sort_info` changed, to scale-correct the precomputed corpse grid. |
| `gamedata/scripts/zzz_aaa_meat_spoiling_tooltip_fix.script` | Re-adds ilrathCXV's `ui_item.build_desc_footer` spoiling-timer line at chunk load (the `zzz_` prefix loads it after `ui_item`/`meat_spoiling`), reading the live timer via `meat_spoiling.save_state`. No-ops if `meat_spoiling` isn't loaded. |
| `gamedata/scripts/zzz_aaa_food_satiety_percent_fix.script` | Wraps `ish_item_stats.override_stats_table` at chunk load, after Sota's own wrapper, to set `eat_satiety` back to `magnitude=100` / `unit="st_perc"`. Only rewrites the kcal form, so it no-ops if another mod owns the entry. |
| `gamedata/scripts/zzz_aaa_meat_stack_picker_fix.script` | Re-adds ilrathCXV's `Picker_Toggle` / `new_get_param` pair so meat and patch stacks expand in the picker. Skips itself if `meat_spoiling.new_get_param` exists, meaning ilrathCXV's copy won. |
| `gamedata/scripts/zzz_aaa_best_condition_stacking_fix.script` | Re-adds `FindChildIdByBestCondition`, the `AddChild` promotion, and `ResetToChild`. Skips itself if `UICellItem.FindChildIdByBestCondition` exists, meaning Sota's copy won. |
| `gamedata/scripts/zzz_aaa_sortingplus_highlight_fix.script` | Re-adds SortingPlus' `UIInventory:Highlight` / `UnHighlight_All` / `UICellItem:Update` overrides. No marker exists to detect SortingPlus' copy, so all three are written to be safe if applied twice. |
| `gamedata/scripts/zzz_aaa_weight_rounding_fix.script` | Wraps `utils_ui.get_stats_value` to record the current stat, and the unlocalized `utils_ui.stats_round_idp` to force 1 decimal place on the two carry-weight stats, both at `on_game_start`. Logs a one-time `! FIXHD\|` warning and no-ops if the unlocalizer didn't apply. |
| `gamedata/configs/unlocalizers/unlocalizer_fix_hd_stats_rounding.ltx` | Promotes `utils_ui`'s file-local `stats_round_idp` out of file scope for the fix above. Kept separate from the SortingPlus unlocalizer so that one stays byte-identical to mod 464's copy; sections merge across every file in the folder. Requires modded exes. |
| `gamedata/configs/mod_system_zzzzzzzzzzzzzzzzzzzzzzzzzzz_fix_rpk74_hd_icon.ltx` | DLTX override giving `wpn_rpk74` the HD atlas coordinates it should have inherited. DLTX applies patches in filename order (`FS_FileSet` is sorted by name, `Xr_ini.cpp`), not MO2 order, so the long `z` run is what makes this land after the icon pack's `mod_system_zzzzzzzzzzzzzzz_s2rifles.ltx`. |
| `gamedata/configs/unlocalizers/unlocalizer_fix_hd_icon_pos.ltx` | Exposes SortingPlus' local `favorite_itms`/`junk_itms`/`item_order` to our sort factory. Identical to (and safely additive with) the config mod 464 ships; needed because Inventory Antifreeze activates our `seax_*` script by filename even when 464 itself is disabled. |

## Install

MO2: install as a normal mod named "Fix HD Inventory Icon Position", placed
towards the end of the list so it beats both mods whose scripts it replaces.
Those are "464- Inventory Open Lag Reducer - sea-ex" (for
`seax_sortingplus_opt_sort_by_kind.script`) and
"Looting_takes_time_REDUX_by_Priler_update_" (for `zzzz_loot_searching.script`).
REDUX is the later of the two, so anything past it satisfies both. The load order
section above covers why nothing else here cares about priority.

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
`FindFreeCell` call and the inputs and results of every icon layer draw, which
identifies misplacements caused by other mods.

## Resilience

All cross-mod references are guarded. If SortingPlus, the lag reducer, the HD
framework, `magazine_binder`, or the exe-side unlocalizer (which exposes
SortingPlus' local `favorite_itms`/`junk_itms`/`item_order`) are missing, the
scripts no-op or degrade to a plain size/alphabetical sort instead of erroring.

One degradation bit users for real: Inventory Antifreeze soft-detects
`seax_sortingplus_opt_sort_by_kind` by filename, so shipping that script makes
468 route sorting through it even when 464 is disabled. Without 464 no
unlocalizers config loaded, so the guarded fallbacks silently dropped
favourite/kind ordering ("inventory sorts randomly"). This mod ships its own copy
of the unlocalizers config so sorting works with or without 464, and the sort
factory logs a one-time `! FIXHD|` warning if the SortingPlus locals are still
unreachable.
