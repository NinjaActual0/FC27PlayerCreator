# FC 27 Cranium Schema — Research Notes

Reverse-engineered from `FC27SDK.dll` (FMT SDK) via `ilspycmd`. **No game launch required.**
Last updated: 2026-10-06

---

## Summary

EA FC 27 player faces are described by a **`CraniumRecipe`** asset — a flat list of numeric
blend weights keyed by **hashed morph names**, plus a set of pointer references for hair,
eye colour, skin tone and materials.

The critical finding: **the hash→name mapping ships as data**, in `CraniumBlendsInitData`
(parallel `BlendNames` / `BlendNameHashes` arrays). No hash-reversal is required.

All findings below are from SDK type schemas. Field **values** live in the game's
`.cas` archives and have not been read yet.

---

## 1. The Morph Weight System

### `CraniumShapeBlendPair` — one morph weight

```csharp
public class CraniumShapeBlendPair {
    public uint   ShapeNameHash;   // which morph
    public float  WeightValue;     // 0.0 – 1.0, how much
}
```

### `CraniumBlendsInitData` — THE HASH TABLE

```csharp
public class CraniumBlendsInitData : DataContainerPolicyAsset {
    public List<CString> BlendNames;       // human-readable morph names
    public List<uint>   BlendNameHashes;   // parallel array — same index
}
```

**This is the key insight.** Read this asset once, build `Dictionary<uint, string>`, and every
`ShapeNameHash` in any recipe becomes human-readable. No dictionary attack, no reversing EA's
hash function.

> ⚠️ **Open question:** whether `BlendNames` and `BlendNameHashes` are guaranteed index-aligned.
> They are described as parallel lists so alignment is the strong assumption, but this has not
> been verified against real data. **Verify before relying on it.**

---

## 2. `CraniumRecipe` — the player's face

```csharp
public class CraniumRecipe : DataContainerPolicyAsset {
    // ── morphs ──
    public List<CraniumShapeBlendPair> Blends;          // THE FACE

    // ── appearance pointers ──
    public PointerRef Hair;
    public PointerRef BrowHair;
    public PointerRef FacialHair;
    public PointerRef EyeColor;
    public PointerRef Skintone;
    public float      SkinToneValue;
    public PointerRef MaterialCollection;
    public List<PointerRef> Accessories;
    public List<uint>       SurfacePacks;

    // ── hair/decal colour variants ──
    public PointerRef HairColor;
    public PointerRef HairDecalColor;
    public PointerRef BrowHairColor;
    public PointerRef FacialHairColor;
    public PointerRef FacialHairDecalColor;
    public PointerRef BrowHairDecalColor;

    // ── metadata ──
    public CString   AssetName;
    public CString   DisplayName;
    public List<CString> Tags;
    public PointerRef RecipeId;
    public PointerRef VersionInfo;
    public float      CurrentAge;
    public bool       IsPhotofit;

    // ── topology + aging ──
    public CraniumBlendShapeType BlendShapeType;   // Topo2s | Topo3
    public List<CraniumSecondaryFormIntensity> SecondaryForms;
    public List<CraniumAgingDeltaValue>        AgingDeltaValues;

    // ── construction history (undo/audit trail) ──
    public List<CraniumConstructionHistory> PresetConstructionHistory;
    public List<CraniumConstructionHistory> DecalPresetConstructionHistory;
    public List<CraniumConstructionHistory> ColorPresetConstructionHistory;
}
```

### `CraniumBlendShapeType` — two head topologies

```csharp
public enum CraniumBlendShapeType {
    CraniumBlendShape_Topo2s,
    CraniumBlendShape_Topo3,
    CraniumBlendShape_Count
}
```

### Supporting types

```csharp
public class CraniumAgingDeltaValue    { /* age-driven morph deltas */ }
public class CraniumSecondaryFormIntensity { /* e.g. beard/makeup strength */ }
public class CraniumSecondaryForm      { /* definition of a secondary form */ }
public class CraniumConstructionHistory { /* preset application history */ }
```

---

## 3. The CAP UI Is Specified as Data

EA ships complete slider definitions. We do not need to guess ranges or reverse-engineer
the creator UI.

### `CraniumSculptSlider`

```csharp
public class CraniumSculptSlider {
    public CString  SliderName;
    public CString  LeftLabel;
    public CString  RightLabel;
    public float    MinRange;
    public float    MaxRange;
    public float    MultiplierValue;
    public float    YoungLimit;
    public CraniumAgeControlData AgeControls;
}
```

### `CraniumControlShapeData` — one facial control point

Carries **13 directional sliders**, which reveal EA's rig axis layout:

| Slider | Axis / meaning |
|--------|----------------|
| `D`    | Down           |
| `DU`   | Down-Up        |
| `U`    | Up             |
| `L`    | Left           |
| `R`    | Right          |
| `N`    | Nose           |
| `NW`   | Nose Width     |
| `M`    | Mouth          |
| `F`    | Forehead       |
| `RA`   | Right-Left asym. |
| `LM`   | Left-Mid asym.  |
| `BF`   | Brow/Forehead  |
| `SLSR` | Side (combined) |

```csharp
public class CraniumControlShapeData {
    public CString   ControllerName;
    public CString   SymmetricControllerName;   // paired control for symmetry
    public CString   DisplayName;
    public CString   MaterialName;
    public List<CString> RegionNames;           // grouping for UI
    public CString   ControlType;

    // 13 sliders: D, DU, U, L, R, N, NW, M, F, RA, LM, BF, SLSR

    public CraniumControlShapeSideFlags Side;
    public bool   Enabled;
    public bool   Invert;
    public bool   ForceSymmetry;
    public float  ConstraintMultiplier;
    public float  SymmetricConstraintLimit;
    public uint   PickingId;
    public CraniumSecondaryFormRegion SecondaryFormRegion;
    public CraniumAgeSecondaryControl  AgeSecondaryControls;
}
```

### `CraniumControlShapeMapping` — the top-level mapping asset

```csharp
public class CraniumControlShapeMapping : Asset {
    public List<CraniumControlShapeData> ControlShapes;
    public CraniumControlShapeNameMapping DisplayNameMapping;  // Regions[] + ControlTypes[]
    public PointerRef MorphGraphDataMappingAsset;
    public float GlobalConstraintMultiplier;
}

public class CraniumControlShapeNameMapping {
    public List<CraniumControlShapeRegion>      Regions;      // RegionName + DisplayName
    public List<CraniumControlShapeControlType> ControlTypes;  // TypeName  + DisplayName
}
```

**Implication:** a WPF face editor can be *generated* from `ControlShapes` — sliders,
ranges, labels, regions, symmetry rules and constraints all come from the data.

---

## 4. Blend Resolution / Morph Graph

### `FBMorphGraphDataMappingAsset`

```csharp
public class FBMorphGraphDataMappingAsset : Asset {
    public List<FBMorphGraphDataMapping> DataMapping;
    public List<FBMorphPrecomputedMapping> PrecomputedMappings;
    public bool EnablePropertyPins;
}

public class FBMorphGraphDataMapping {
    public uint DataFieldNameHash;                        // links back to recipe blends
    public List<FBMorphGraphDataMappingTarget> Targets;
}

public class FBMorphGraphDataMappingTarget {
    public uint   GraphFieldHash;
    public uint   GraphNodeNameHash;
    public float  MinValue;
    public float  MaxValue;
    public float  Multiplier;
}

public class FBMorphPrecomputedMapping {
    public uint TargetIndex;
    public uint DataFieldNameHash;
    public uint MappingIndex;
}
```

### ⚠️ IMPORTANT — unresolved semantic risk

`FBMorphRecipe.Blends` holds raw weights, but resolution goes through the morph graph, and
`PrecomputedMappings` suggests EA **precomputes** blend resolution at build time.

**Risk:** editing only `Blends` may produce *"value changed but the face did not move."*

**Open question:** are `PrecomputedMappings` derived-and-cacheable, or authoritative state that
must be invalidated/recomputed on edit? Must be determined before trusting weight-only edits.

Related: `FBMorphGraphData`, `FBMorphNodeGraph`, `FBMorphBlendsAsset`, `FBMorphResultNodeData`,
`FBMorphNodeData`, `FBMorphConnection`, `FBMorphCompositorNodeData`, `FBMorphBlendsNodeData`,
`FBMorphPrecomputedMapping`, `FBMorphEntityBlueprint`, `FBMorphResourceAsset`,
`FBMorphTargetMeshResourceAsset`, `FBMorphMeshResourceAssetProvider`, `FBMorphPresetValidatorData`.

---

## 5. Databases (slot / option lists)

```csharp
public class CraniumDatabase : Asset {
    public List<PointerRef> Heads;     // the head meshes (Topo2s / Topo3)
    public List<PointerRef> Presets;
    public List<CString>     SlotNames; // hair, eyes, brows, facial hair ...
}

public class CraniumSurfaceDatabase : Asset {
    public List<PointerRef> SurfacePackPresets;
    public PointerRef       SurfacePackCategory;
    public List<PointerRef> SecondaryFormsPresets;
    public PointerRef       SecondaryFormsCategory;
}

public class CraniumUIDatabase : Asset {
    public List<PointerRef> PresetUIs;
}
```

Also present: `CraniumColorPreset`, `CraniumDecalPresetInfo`, `CraniumEyeColorPreset`,
`CraniumSkinTonePreset`, `CraniumHairColorAgeAdjustment`, `CraniumHairGeoColorPreset`,
`CraniumHairDecalColorPreset`, `CraniumMaterialCollection`, `CraniumSurfacePack`,
`CraniumHair`, `CraniumSecondaryFormConfig`, `CraniumSurfacePackConfig`,
`CraniumPresetSlotConfig`, `CraniumDefaultPresetConfig`, `CraniumDefaultColors`,
`CraniumMaterialEditorTextureData`.

---

## 6. Runtime services

```csharp
public class CraniumBridgeEntityData {
    public PointerRef HotspotBrt;             // ← BundleRefTable
    public PointerRef HeadDecalLibrary;
    public PointerRef HeadAlaskaAsset;
    public PointerRef DefaultMaterialRecipe;
}
```

**`HotspotBrt` is significant.** It maps player attributes → head texture/decal **bundles**.

On the Madden side (M27 DuplicationPlugin work, Sept 2026) the conclusion was that head
textures bind through per-player **BRT bundles rather than EBX references** — that finding
cost weeks of dead-end EBX remapping. `HotspotBrt` appears to be exactly that mechanism, and it
is **driven by recipe fields**. Implication: drive the recipe, let the game assign bundles,
rather than hand-constructing bundle references.

Other services: `CraniumBackendServiceEntityData`, `FifaCraniumBackendServiceEntityData`,
`CraniumEventsEntityData`, `CraniumPickingEntityData`, `CraniumMaterialEditorEntityData`,
`CraniumAlaskaMaterialEditorEntityData`, `FCFBMorphEntityData` (→ `DataMappingAsset`),
`FBMorphEntityData`.

---

## 7. Randomization / aging / config

`CraniumRandomizationConfig`, `CraniumRandomizationItem`, `CraniumRandomizationCategory`,
`CraniumGlobalConfig`, `CraniumGameModeConfig`, `CraniumAgeCategoryConfig`,
`CraniumAgingDatabase`, `CraniumAgingFiltersConfig`, `CraniumDynamicAgingConfig`,
`CraniumAgeControl`, `CraniumAgeControlData`, `CraniumAgeSecondaryControl`,
`CraniumAgeDecadeSurfacePack`, `CraniumAgeDecadeSurfacePackGroup`,
`CraniumCategoryFiltersConfig`, `CraniumCategoryFilterConfig`, `CraniumCategoryFilterMap`,
`CraniumHairColorAgeAdjustment`, `CraniumPopupDefinitions`, `CraniumPopup`,
`CraniumPopupButton`, `CraniumInitDataContainer`, `CraniumThumbnailConfig`,
`CraniumPreset`, `CraniumPresetId`, `CraniumPresetUI`, `CraniumPresetLocalizationInfo`,
`CraniumTagData`, `CraniumRecipeId`.

`CraniumRandomizationConfig` + `...Item` + `...Category` are directly relevant to a
"randomize face" feature and worth reading later.

---

## 8. Negative findings

**`FC27Plugin.dll` contains NO player-creator logic.** It is 45 classes, purely:

```
FC27AssetCompiler, FC27AssetLoader, FC27TOCFile(+Writer),
Textures.{FC27TextureResourceReader,Writer},
Meshes.Readers.{...MeshSet,SkinnedMesh,RigidMesh,CompositeMesh},
Meshes.Writers.{...},
Cache.{FC27CacheReader,Writer,FC27CacheHelpers},
Settings.FC27AppSettingsLiveEditor, FC27CustomAssetEntryEnumerations,
FC27ModsCleanupFunctions
```

The earlier gameplan's "hijack the in-game CAP logic from `FC27Plugin.dll`" premise is
**wrong**. CAP assembly code lives in the game, not the plugin. Better news than it sounds —
the data contract *is* the specification, and the game enforces it.

`FMT.dll` does not exist as a separate file; the host is `FMT.exe` (441 MB, single-file).
`FMT.Ebx.dll` is small (48 classes) and holds only the attribute definitions
(`HashAttribute`, etc.). No string-hash helper is exposed anywhere in the SDK or FMT.Ebx —
consistent with the hash table being pure data.

---

## 9. Asset locations

| Path | Purpose |
|---|---|
| `…\FC27 Mods\FMT_PRO\SDK\FC27SDK.dll` | Type schemas — 5,449 classes |
| `…\FC27 Mods\FMT_PRO\Plugins\FC27Plugin.dll` | Plugin plumbing — 45 classes |
| `…\FC27 Mods\FMT_PRO\FrostbiteProfiles\FC26Profile.json` | (note: FC26 profile — FC27 profile not in that folder) |
| `C:\Program Files\EA Games\EA SPORTS FC 27\Data\Win32\superbundlelayout\installpackage_*\cas_*.cas` | Game data — ~1 GiB each, 7 packages |

Reading real `CraniumBlendsInitData` **values** requires extracting from the `.cas` archives
(either via FMT once it runs, or a hand-rolled CAS reader).

---

## 10. Gameplan revision

The original `GAMEPLAN.md` assumed we would locate and port EA's CAP assembly logic.
That assumption is dead (see §8). Revised approach:

1. ~~Find CAP logic in `FC27Plugin.dll`~~ — **done, it isn't there**
2. ✅ **Map the schema** — this document
3. **Extract real values** from `.cas` — needs FMT, or a CAS reader
4. **Verify the hash table** — confirm `BlendNames`/`BlendNameHashes` index alignment
5. **Resolve the precomputed-mapping question** (§4) — does weight-only editing work?
6. **Build a thin end-to-end slice** — read a player, change one blend, write back
7. **Generate the UI** from `ControlShapes`
8. **In-game verification** — blocked on FMT launching the game

Recommend step 4/5 before step 6 — both are cheap to settle and both could invalidate
the write path.

---

## Reproduction

```powershell
ilspycmd --disable-updatecheck -l c "…\SDK\FC27SDK.dll"          # list classes
ilspycmd --disable-updatecheck -t FMT.Ebx.CraniumRecipe "…\FC27SDK.dll"   # decompile type
ilspycmd --disable-updatecheck -p -o <outdir> "…\FC27SDK.dll"    # full project
```

`ilspycmd` is headless (no GUI) — installed via `dotnet tool install --global ilspycmd`.
Add `--disable-updatecheck` to avoid the network update check.

Note: `ilspycmd` writes UTF-16 output on Windows PowerShell redirect; strip NULs or use `-o`.