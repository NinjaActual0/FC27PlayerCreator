# Gameplan: FC 27 Player Builder Plugin for FMT + Blender Export Workflow

Created: 2026-10-02

## Goal
Create an integrated player builder/face editor for EA Sports FC 27 in FMT that works with Cranium technology (generic meshes + morphs + recipes + BRT bundles). Also create a Blender plugin to allow custom head sculpting and export to FMT project files.

---

## PART 1: Hijack the In-Game Player Creator (CAP) Logic

The smartest approach is to leverage the game's own Create-A-Player (CAP) logic rather than rebuilding it from scratch. This ensures perfect compatibility with Cranium, morph constraints, and BRT bundle assignment.

### Why this route
- **Perfect accuracy**: Uses EA's exact morph definitions, ranges, and validation rules
- **BRT-safe**: The game's assembly logic knows how to correctly assign bundles/materials for Cranium heads
- **Saves massive time**: Avoids reverse-engineering every morph/constraint by hand
- **Future-proof**: If EA adds morphs, we just extract updated definitions

### Phase 1A: Find the Create-A-Player (CAP) System (1-2 days)

| Task | Details | Priority |
|---|---|---|
| [ ] Search FC27Plugin.dll | Decompile and search for: `CreatePlayer`, `PlayerCreator`, `CAP`, `CharacterCreator`, `Cranium`, `MorphSet`, `MorphTarget`, `VisualCreator` | Critical |
| [ ] Search game binaries | Look through FC27's main game exe/DLLs in the game install for CAP-related classes/UI | High |
| [ ] Find UI schema | Look for EBX/UI data for the Career Mode player creator - UI definitions usually list morph groups, slider names, ranges, and categories | High |
| [ ] Trace "Save" flow | When you save a created player in-game, what classes assemble the final recipe/visual data? That assembly function is what we want to reuse | Critical |

**Deliverable:** Document mapping where CAP logic lives (plugin vs game binaries)

### Phase 1B: Extract Canonical Definitions (2-3 days)

| Task | Details | Priority |
|---|---|---|
| [ ] Extract morph definitions | Get all morph targets, their display names, categories, min/max ranges, and step sizes exactly as the game defines them | Critical |
| [ ] Extract option sets | Hair styles, hair colors, eye colors, skin tones, complexions, facial hair - get the valid ID lists | Critical |
| [ ] Extract recipe assembly logic | Isolate the method(s) that take morph values + selections and produce the final visual/recipe data structure | Critical |
| [ ] Extract validation rules | Find any constraints (invalid combinations, dependent values) | Medium-High |

**Deliverable:** `FC27_CAP_Schema.json` containing all morphs, ranges, categories, and option lists

### Phase 1C: Recreate the CAP UI in FMT (1-1.5 weeks)

| Task | Details | Priority |
|---|---|---|
| [ ] Auto-generate UI from schema | Generate tabs/groups/sliders/dropdowns from schema to mirror in-game layout | High |
| [ ] Mirror in-game organization | Group morphs like the actual CAP for familiarity | Medium |
| [ ] Add presets & utilities | Save/load face presets, randomize, reset, copy/paste between players | Nice-to-have |
| [ ] Live validation | Enforce ranges/constraints pulled from game logic | High |

**Deliverable:** WPF UI that feels like in-game CAP, integrated into FMT

### Phase 1D: Reuse Assembly Logic & Write-Back (1 week)

| Task | Details | Priority |
|---|---|---|
| [ ] Port/reuse assembly logic | Extract minimal code to convert UI state → valid recipe/visual data. Adapt to FMT's types for FC27 | Critical |
| [ ] Write to roster/save safely | Serialize back to FC27's format, handling BRT correctly | Critical |
| [ ] In-game verification | Test morphs, textures, BRT bundles all load correctly | Critical |

**Timeline (Part 1):** ~2.5 - 3 weeks part-time

---

## PART 2: Blender Plugin → Export to FMT Project File

For sculptors who want full control. Requires bridging Blender's data with FC27's Cranium format and FMT's project format.

### Phase 2A: Reverse-Engineer FMT Project Format (1-2 days)

| Task | Details | Priority |
|---|---|---|
| [ ] Analyze .fbproject files | Examine existing FMT project files to determine format (JSON/XML/binary) | Critical |
| [ ] Understand project structure | What assets are referenced? How are modifications stored? | Critical |
| [ ] Look at FMT codebase | Search for project serialization (SaveProject/LoadProject) in FMT DLLs | High |

**Deliverable:** Document FMT project file schema

### Phase 2B: Understand Cranium Head Requirements (1-2 days)

| Task | Details | Priority |
|---|---|---|
| [ ] Extract base head mesh | Export FC27's generic Cranium head mesh(es) as reference (correct topology/UVs/vertex order) | Critical |
| [ ] Map morph target names | Use morph names from Part 1B - Blender shape keys must match exactly | Critical |
| [ ] Understand material setup | How Cranium head materials/textures/BRT work | High |

**Deliverable:** Reference head mesh + morph name mapping

### Phase 2C: Blender Addon - Core Exporter (1-1.5 weeks)

| Task | Details | Priority |
|---|---|---|
| [ ] Create Blender addon scaffold | Python addon for Blender 4.x/5.0 with UI panel | High |
| [ ] Mesh export with constraints | Validate topology matches reference, warn if different | Critical |
| [ ] Shape keys → Morphs | Export shape keys as morph deltas, map by name to FC27 morph targets | Critical |
| [ ] UV/Normals/Weights | Preserve UVs, normals, skin weights correctly | High |
| [ ] Texture export/packing | Handle base color/normal/etc in FMT/FC27 expected format | High |

**Deliverable:** Working Blender addon for exporting head mesh + morphs

### Phase 2D: Export to FMT Project Format (1 week)

| Task | Details | Priority |
|---|---|---|
| [ ] Generate valid .fbproject | Create project file that references exported assets properly | Critical |
| [ ] Asset integration | Ensure EBX/RES/Chunks are correctly referenced for FMT | Critical |
| [ ] BRT/bundle handling | Generate valid bundle references (learn from DuplicationPlugin lessons) | Critical |
| [ ] FMT import validation | Open generated .fbproject in FMT, verify loads cleanly | Critical |

**Deliverable:** Blender addon exports valid .fbproject files for FMT

### Phase 2E: Polish & Workflow (3-5 days)

| Task | Details | Priority |
|---|---|---|
| [ ] UI/UX | Validation warnings, reference mesh loader, morph mapping, presets | Nice-to-have |
| [ ] Documentation | Clear instructions on topology requirements and workflow | High |
| [ ] Error handling | Good feedback for common issues | High |

**Timeline (Part 2):** ~3 - 3.5 weeks part-time

---

## Recommended Order

**Do Part 1 first.** It gives you the ground truth (morph names, ranges, assembly logic, BRT behavior) which makes Part 2 much easier. Also delivers a usable tool faster.

Suggested sequence: 1A → 1B → 1C → 1D → 2A → 2B → 2C → 2D → 2E

---

## Key Files to Examine

| File | Purpose |
|---|---|
| `C:\Users\Ninja\Documents\Mods\FC27 Mods\FMT_PRO\Plugins\FC27Plugin.dll` | Main FC27 plugin - contains recipe/visual/CAP logic |
| `C:\Users\Ninja\Documents\Mods\FC27 Mods\FMT_PRO\SDK\FC27SDK.dll` | Type schemas for FC27 |
| `E:\Game stuff\mods\Repos\Repos\FMT.Madden26Plugin\Roster\PlayerVisualRecipe.cs` | Reference CFB27 implementation |
| `E:\Game stuff\mods\Repos\Repos\FMT.Madden26Plugin\Roster\CFB27RosterReader.cs` | Roster parsing patterns |
| `E:\Game stuff\mods\Repos\Repos\FMT.Madden26Plugin\Roster\FaceTemplateMatcher.cs` | Face matching logic |
| `E:\Game stuff\mods\Repos\Repos\DuplicationPlugin\DuplicationTool.cs` | BRT/bundle handling lessons (what to avoid) |

---

## Tools Needed

- **[dnSpy](https://github.com/dnSpy/dnSpy)** or **[ILSpy](https://github.com/icsharpcode/ILSpy)** - Decompiling
- **Visual Studio** - Building FMT plugin
- **Blender 4.x/5.0** - For the addon
- **FMT** - For testing

---

## Key Insights

- **CAP hijack is the game-changer**: Instead of reverse-engineering morphs/constraints, extract them directly from the game's creator logic. This is faster, more accurate, and handles BRT correctly.
- **Topology is king for Blender**: Cranium requires specific topology. Provide a reference head mesh and validate topology in the addon.
- **Data-driven design**: Extract everything to JSON schema (morphs, ranges, options). Makes both UI and Blender exporter flexible and maintainable.
- **Learn from DuplicationPlugin**: Bundle/BRT assignment was the pain point for M27. Always reuse the game's assembly logic for recipe generation rather than inventing our own.
