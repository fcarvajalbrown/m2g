# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**m2g** (Mixamo to GLB) — offline pipeline to convert Mixamo FBX characters and animations into separate GLB files using Blender's Python API (`bpy`). Written in Portuguese (README, commits).

## Running the Script

Requires macOS with Blender 5.x installed at `/Applications/Blender.app/Contents/MacOS/Blender`.

```bash
# Quick run via wrapper script
./convert.sh

# Direct invocation
/Applications/Blender.app/Contents/MacOS/Blender -b -P ./mixamo_batch.py -- \
  --character ./input/x-bot.fbx \
  --anims ./input/anims/idle.fbx ./input/anims/walking.fbx \
  --out ./output \
  --mode inplace \
  --inplace-ground-lock true \
  --export-mesh-in-anims false \
  --apply-scale true \
  --fix-root true \
  --limit-weights 4 \
  --abort-on-incompatible false
```

There is no test suite, linter, or package manager. Testing is manual — run with FBX files and inspect `output/report.json`.

## Architecture

The entire codebase is a single file: **`mixamo_batch.py`** (~1100 lines). Functions are organized by concern in this order:

1. **Args parsing** — custom `parse_args()` extracts flags from `sys.argv` after `--`
2. **Scene helpers** — `reset_scene()`, `select_none()`, `select_only()`
3. **Import/Export** — `import_fbx()`, `export_glb_selected()` (with retry dropping unsupported kwargs)
4. **Object discovery** — `find_armature()`, `find_meshes_deformed_by_armature()`
5. **Rig normalization** — `strip_bone_prefixes()` removes `mixamorig:` prefix; `ensure_root_bone()` creates a `Root` bone above `Hips`
6. **Weight limiting** — `limit_vertex_weights()` caps bone influences per vertex
7. **Animation compatibility** — `check_anim_compat()` validates bone sets (requires `common_ratio >= 0.98`)
8. **Action baking** — `bake_action_to_base()` copies F-curves from source rig to base rig, with scale compensation when armatures differ in size
9. **In-place processing** — `make_inplace_on_action()` removes world displacement via Root/Object bone drift removal, with Hips fallback and optional foot-bone anchoring
10. **F-curve iteration** — `iter_action_fcurves()` compatibility layer for both legacy Action API and Blender 5's slotted/layered API
11. **Main pipeline** — `main()` orchestrates the full workflow and writes `report.json`

## Key Design Decisions

- **Blender API dual compatibility**: `iter_action_fcurves()` handles both legacy and slotted Action APIs for forward/backward compatibility across Blender versions.
- **In-place animation** has multiple strategies: Root/Object bone drift removal (preferred), Hips trend-line detrending fallback, and optional foot-bone anchoring (`--inplace-anchor foot`).
- **Non-destructive processing**: temporary imported objects are cleaned up after each animation extraction.
- **Export retry logic**: `export_glb_selected()` catches unrecognized keyword errors and retries dropping the offending parameter (handles Blender version differences in glTF exporter).

## Output

- `<character>.glb` — mesh + skeleton, no animations
- `<anim>.glb` — skeleton + exactly 1 animation clip (exported with `ACTIVE_ACTIONS`)
- `report.json` — full processing report with compat checks, bake metrics, and export paths
