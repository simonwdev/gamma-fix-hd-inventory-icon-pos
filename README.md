# Fix HD Inventory Icon Position (GAMMA)

Per-mod fixes for HD icons (`inv_grid_scale`, from HD_Inventory_Icons_Framework)
being drawn or placed wrong in the inventory:

- **464- Inventory Open Lag Reducer**: sorted icons no longer claim double-size tiles
- **468- Inventory Antifreeze**: items added to a big inventory mid-sort no longer throw sort errors
- **110- SortingPlus**: favourite/junk marks sit on the icon again; items taken or dropped while looting land on the right rows; its highlight bookkeeping is back
- **Looting Takes Time REDUX**: corpse loot grids pack tightly again instead of leaving cell-wide gaps
- **HD Attachment Icons For GAMMA**: scoped weapon variants the pack doesn't cover now draw its straight gun-mounted scope icons instead of the diagonal inventory item icon
- **MartinLore's Stalker 2 icons**: the RPK-74 draws a gun again instead of a garbled crop of unrelated weapons, and the drum-mag RPK-16 is no longer squashed when the icon pack isn't installed
- **ilrathCXV's Meat Spoiling Timer in Tooltips**: the "hours until rotten" line is back on raw and cooked meat; meat and patch stacks expand in the picker again, so you can take the freshest piece
- **G.A.M.M.A. Artefacts Reinvention**: stacks of artefacts, junk artefacts, outfit attachments and mutant hides open in the picker again, so you can take a specific one instead of only the one on top (a self-inflicted regression, broken in v0.6.0)
- **UI Rework G.A.M.M.A. Style - Sota**: food tooltips read satiety as a percentage instead of a raw kcal figure; a stack shows the best-condition item on top; carry-weight bonuses keep their decimal, so a 1.58 kg backpack stops reading as 2 kg

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
   it in a `ceil(5/2) x ceil(2/2)` = 3x1 cell. The fix deletes the four stale
   coordinates (DLTX drops a `!key` line in the base+mods merge, so the
   parent's value survives the parent merge that follows), restoring the
   inheritance the override was breaking. With the pack installed that
   resolves to 10x4 at 52,12, which keeps the inventory footprint at 5x2
   cells - what the gun occupied before the HD pack.

   Deleting rather than assigning the HD coordinates matters. Neither
   `wpn_rpk` nor `wpn_rpk74` declares `icons_texture` or `inv_grid_scale` of
   its own; both reach `wpn_rpk74` only by inheritance from `![wpn_rpk]`.
   Writing them here (v0.5.0 through v0.6.0 did) was redundant with the icon
   pack installed and wrong without it. `icons_texture` forced the section
   onto an atlas that wasn't there, and `inv_grid_scale` leaked to
   `wpn_rpk74_16_drum`, the one other section that inherits `wpn_rpk74`. The
   drum redeclares its texture and all four coordinates but not the scale, so
   it took ours and drew its 250x100 icon in a 3x1 cell. `wpn_rpk74_16` was
   unaffected because it inherits `identity_immunities`, not `wpn_rpk74`.
   Without the icon pack `wpn_rpk74` now falls back to `wpn_rpk`'s vanilla
   icon (5x2 at 35,18) instead of its own (35,14) - a near-identical gun, and
   nothing garbled. A DLTX patch cannot be conditioned on another mod's
   presence, so that is the tradeoff.

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

   The shim also has to *chain* to whatever it displaced, which v0.6.0 and
   v0.7.0 did not. Artefacts are the same shape of problem as meat:
   `use_condition` is commented out on `[af_base_mlr]` in vanilla
   `items_artefacts.ltx`, so nothing about them declares it in config.
   G.A.M.M.A. Artefacts Reinvention solves it the same way ilrathCXV did, in
   `zz_item_artefact.script`: a `SYS_GetParam` shim that reports `use_condition`
   as true for five degradable kinds, installed at chunk load and gated on MCM
   `enable_condition_bar`, which defaults on. That is what gives those items
   both their condition bar and their picker.

   The name undersells the reach, so the regression was never artefact-only:

   | kind | what it is | was broken |
   |------|------------|------------|
   | `i_arty` | artefacts (`af_medusa`) | yes |
   | `i_arty_junk` | junk artefacts (`af_black`) | yes |
   | `i_attach` | outfit attachments (`af_kevlar`, `af_camelbak`) | yes |
   | `i_mutant_belt` | mutant hides and parts (`hide_chimera`) | yes |
   | `i_arty_cont` | artefact containers (`af_medusa_lead_box`) | no |

   Mutant hides matter as much as artefacts here, because the same script
   assigns their condition from the knife they were skinned with. Containers
   escaped because `mod_system_grok_llmc_fix.ltx` sets a real
   `use_condition = true` on every `af_*_lead_box` section: the flag is in
   config, so there was no shim for ours to displace.

   The shim installs into `ui_inventory` and `utils_ui` and never touches `_G`.
   Ours fell back to the global `SYS_GetParam`, so for the duration of
   `Picker_Toggle` that override was invisible and those items read
   `use_condition = false` -- `Picker_Toggle` logged `cell has no condition ->
   hide` and the strip below stayed empty. The condition bar kept drawing,
   because `Add_ProgressBar` reads `utils_ui.SYS_GetParam`, which we do not
   touch: the stack looked expandable and simply refused.

   Nothing else was in range. That shim diverges from the global only for
   `use_condition`, `use_condition` is read exactly once in the whole of
   `ui_inventory.script` (the picker gate), and no other mod here holds a
   persistent `ui_inventory.SYS_GetParam`. Weapons, outfits, headgear and ammo
   were never affected either way, since the gate rescues them on clsid.

   Stacking never saw the shim at all, which is why this was so visible:
   `FindSimilar` belongs to `rax_stacking_control` (GAMMA Mags Reloaded) and
   resolves `SYS_GetParam` through its own chunk to `_G`, so these items stack
   by section with no condition check. Three artefacts at 0.35/0.65/0.95 land in
   one cell, and the picker is the only way to tell them apart.

   The delegate is held in a file-local, saved and restored around the call the
   same way the swap itself is, and skips itself if it finds our own shim
   already installed, so a nested `Picker_Toggle` cannot leave a self-reference
   behind. ilrathCXV's original chains through `_inv_SYS_GetParam` for the same
   reason; our reimplementation kept the restore half of that pattern and
   dropped the delegate half.

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
    ahead of every rounding call in that iteration. If
    `utils_ui.stats_round_idp` is nil the script logs one `! FIXHD|` line and
    no-ops. That line does not diagnose why, because Lua cannot: the unlocalizer
    may be inert for want of modded exes, or it may have applied and captured
    nil because the winning `ui_inventory` has no `stats_round_idp` (only Sota
    defines it), or the winning `utils_ui.script` may be another mod's copy with
    no such declaration. Assigning nil creates no table key, so all three look
    identical from here.

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
| `gamedata/scripts/zzz_aaa_meat_stack_picker_fix.script` | Re-adds ilrathCXV's `Picker_Toggle` / `new_get_param` pair so meat and patch stacks expand in the picker. Chains to the displaced `ui_inventory.SYS_GetParam` rather than the global, which is what keeps Artefacts Reinvention's picker working for artefacts, outfit attachments and mutant hides. Skips itself if `meat_spoiling.new_get_param` exists, meaning ilrathCXV's copy won. |
| `gamedata/scripts/zzz_aaa_best_condition_stacking_fix.script` | Re-adds `FindChildIdByBestCondition`, the `AddChild` promotion, and `ResetToChild`. Skips itself if `UICellItem.FindChildIdByBestCondition` exists, meaning Sota's copy won. |
| `gamedata/scripts/zzz_aaa_sortingplus_highlight_fix.script` | Re-adds SortingPlus' `UIInventory:Highlight` / `UnHighlight_All` / `UICellItem:Update` overrides. No marker exists to detect SortingPlus' copy, so all three are written to be safe if applied twice. |
| `gamedata/scripts/zzz_aaa_weight_rounding_fix.script` | Wraps `utils_ui.get_stats_value` to record the current stat, and the unlocalized `utils_ui.stats_round_idp` to force 1 decimal place on the two carry-weight stats, both at `on_game_start`. Logs a one-time `! FIXHD\|` warning and no-ops if `utils_ui.stats_round_idp` is nil, without asserting which of the three possible causes it was. |
| `gamedata/configs/unlocalizers/unlocalizer_fix_hd_stats_rounding.ltx` | Promotes `utils_ui`'s file-local `stats_round_idp` out of file scope for the fix above. Kept separate from the SortingPlus unlocalizer so that one stays byte-identical to mod 464's copy; sections merge across every file in the folder. Requires modded exes. |
| `gamedata/configs/mod_system_zzzzzzzzzzzzzzzzzzzzzzzzzzz_fix_rpk74_hd_icon.ltx` | DLTX override deleting `wpn_rpk74`'s stale grid coordinates so it inherits the ones the icon pack sets on `wpn_rpk`. DLTX applies patches in filename order (`FS_FileSet` is sorted by name, `Xr_ini.cpp`), not MO2 order, so the long `z` run is what makes this land after the icon pack's `mod_system_zzzzzzzzzzzzzzz_s2rifles.ltx`. |
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
