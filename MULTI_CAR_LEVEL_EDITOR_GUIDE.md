# Multi-Car Pattern Level Editor — Full Design & Implementation Guide

> **Purpose:** Replace the single-size grid editor with a pattern-painting workflow that supports **Small**, **Medium**, **Big**, and **Bus** vehicles, auto-faces cars outward, keeps `CarMover` + `Carout` gameplay intact, and does **not** break existing levels.
>
> **Important:** This document is the deliverable. Copy each script section into your project in the order listed below. No automatic changes were applied to your repo.

---

## Table of Contents

1. [What You Have Today](#1-what-you-have-today)
2. [What This Upgrade Adds](#2-what-this-upgrade-adds)
3. [Architecture Overview](#3-architecture-overview)
4. [How Car Sizes Work on the Grid](#4-how-car-sizes-work-on-the-grid)
5. [How to Assign Car Sizes (Prefabs)](#5-how-to-assign-car-sizes-prefabs)
6. [Implementation Order](#6-implementation-order)
7. [File 1 — CarSizeType.cs](#7-file-1--carsizetypecs)
8. [File 2 — LevelData.cs (updated)](#8-file-2--leveldatacs-updated)
9. [File 3 — CarFootprintHelper.cs](#9-file-3--carfootprinthelpercs)
10. [File 4 — CarMover.cs (small addition)](#10-file-4--carmovercs-small-addition)
11. [File 5 — Carout.cs (multi-cell support)](#11-file-5--caroutcs-multi-cell-support)
12. [File 6 — SpawnCars.cs (updated spawning)](#12-file-6--spawncarscs-updated-spawning)
13. [File 7 — LevelEditorWindow.cs (new editor)](#13-file-7--leveleditorwindowcs-new-editor)
14. [Unity Inspector Setup](#14-unity-inspector-setup)
15. [Workflow: Create a Level From a Pattern](#15-workflow-create-a-level-from-a-pattern)
16. [Outward Facing & Face-to-Face Blocking](#16-outward-facing--face-to-face-blocking)
17. [Backward Compatibility Checklist](#17-backward-compatibility-checklist)
18. [Testing Checklist](#18-testing-checklist)
19. [Troubleshooting](#19-troubleshooting)

---

## 1. What You Have Today

| Component | Current behavior |
|-----------|------------------|
| `LevelData` | Stores `gridColumns`, `gridRows`, and `List<CarSpawnInfo>` with **position + rotation only** |
| `LevelEditorWindow` | Click cell → adds 1-cell car; manual rotation buttons |
| `SpawnCars` | Reads spawns from `LevelData`, but picks a **random prefab** from `carPrefabs` — size is ignored |
| `Carout` | Tracks **one grid cell** (`currentGridIndex`); blockage uses 8-way ray on grid |
| `CarMover` | Pathfinding + parking; expects `Carout` on same GameObject |
| Car prefabs | Already have different **passenger capacity** (e.g. Red=4, Green/Blue=2) but all occupy **one grid slot** |

**Your pain points (confirmed in code):**
- Editor only places one kind of 1×1 car marker.
- Runtime ignores editor car type and randomizes prefab.
- No pattern paint → no smart multi-size layout.

---

## 2. What This Upgrade Adds

| Feature | Description |
|---------|-------------|
| **Pattern paint mode** | Paint cells on/off; one click generates full car layout |
| **4 car sizes** | Small (1 cell), Medium (1 cell), Big (2 cells), Bus (3 cells) |
| **Smart auto-placement** | Pattern → detects lines of 3/2/1 cells → assigns Bus/Big/Small/Medium |
| **Auto face outward** | Every car rotation points toward nearest grid border (puzzle-friendly) |
| **Face-to-face blocking** | Two cars pointing at each other stay blocked → `DoTackle()` (existing flow) |
| **Better editor visuals** | Color-coded cells, size labels, arrows, validation warnings |
| **Prefab by size** | `SpawnCars` spawns the correct prefab per `CarSizeType` |
| **Legacy safe** | Old levels without `carSize` default to **Medium**; empty `carSpawns` still uses `carsToSpawn` fallback |

---

## 3. Architecture Overview

```
┌─────────────────────┐     paint / generate      ┌──────────────────┐
│  LevelEditorWindow  │ ─────────────────────────►│    LevelData     │
│  (Editor only)      │     saves CarSpawnInfo    │  ScriptableObject│
└─────────────────────┘                           └────────┬─────────┘
                                                           │
                     LoadLevel → SpawnCarAsPerLevelNeed    │
                                                           ▼
                                                  ┌──────────────────┐
                                                  │    SpawnCars     │
                                                  │  grid + spawn    │
                                                  └────────┬─────────┘
                                                           │ Instantiate
                                                           ▼
                              ┌────────────────────────────────────────────┐
                              │  Car prefab (CarMover + Carout + collider) │
                              │  carSize set on prefab & copied at spawn     │
                              └────────────────────────────────────────────┘
```

**Shared logic:** `CarFootprintHelper` computes which grid cells a vehicle occupies from **head cell + rotation + size**. Both editor and runtime use the same helper so visuals match gameplay.

---

## 4. How Car Sizes Work on the Grid

| Size | Grid length | Passenger suggestion | Cardinal rotations only for multi-cell |
|------|-------------|----------------------|----------------------------------------|
| **Medium** | 1 cell | 4 (your current Red car) | — |
| **Small** | 1 cell | 2 (your Green/Blue cars) | — |
| **Big** | 2 cells | 6 | 0°, 90°, 180°, 270° |
| **Bus** | 3 cells | 8–10 | 0°, 90°, 180°, 270° |

**Anchor rule (important):** `gridPosition` in `CarSpawnInfo` is the **head** cell (front of car, where the arrow points). Body extends **backward** along the opposite of the forward direction.

Example — Bus at head `(4, 3)` facing **Right (90°)** occupies:
- `(4, 3)` head
- `(3, 3)` body
- `(2, 3)` tail

**Scale visuals at spawn (optional but recommended):**

| Size | Suggested `localScale` multiplier |
|------|-----------------------------------|
| Small | `0.55` |
| Medium | `0.70` (your current spawn scale) |
| Big | `0.85` |
| Bus | `1.00` |

---

## 5. How to Assign Car Sizes (Prefabs)

### Step A — Add `carSize` field to each car prefab

After updating `CarMover.cs` (Section 10), open each prefab under `Assets/Prafab/Car/`:

| Prefab | Set `carSize` | Set `thisCarCapacity` |
|--------|---------------|------------------------|
| GreenCar / BlueCar / WhiteCar / YellowCar | **Small** | 2 |
| RedCar (and other 4-seat cars) | **Medium** | 4 |
| BigCar prefab (duplicate RedCar, stretch mesh) | **Big** | 6 |
| Bus prefab (use SimplePoly bus model) | **Bus** | 8 |

Each prefab **must** keep:
- Tag: `car`
- Components: `CarMover`, `Carout`, collider
- `defaultCarType` / `carType` color enum unchanged

### Step B — Wire prefabs in `SpawnCars`

In the Inspector on **SpawnCarsManager**, fill the new **Cars By Size** list:

```
Medium → RedCar prefab
Small  → GreenCar prefab (or rotate colors at runtime later)
Big    → BigCar prefab
Bus    → Bus prefab
```

Fallback: if a size slot is empty, spawner uses `carPrefabs` list (old behavior).

### Step C — Create Big & Bus prefabs (one-time)

1. Duplicate `RedCar.prefab` → rename `BigCar.prefab`
   - Scale mesh child slightly longer on Z
   - BoxCollider size Z × 2 footprint feel
   - `carSize = Big`, `thisCarCapacity = 6`

2. Duplicate → `BusCar.prefab`
   - Assign SimplePoly bus mesh OR scale RedCar longer
   - `carSize = Bus`, `thisCarCapacity = 8`

---

## 6. Implementation Order

Apply files **in this exact order** to avoid compile errors:

1. `CarSizeType.cs` *(new)*
2. `LevelData.cs` *(replace)*
3. `CarFootprintHelper.cs` *(new)*
4. `CarMover.cs` *(add one field)*
5. `Carout.cs` *(replace CheckForBlockage + occupation helpers)*
6. `SpawnCars.cs` *(replace spawn + blockage methods)*
7. `LevelEditorWindow.cs` *(replace editor window)*

Then configure prefabs + SpawnCars Inspector (Section 14).

---

## 7. File 1 — CarSizeType.cs

**Path:** `Assets/Script/Level/CarSizeType.cs`

```csharp
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Medium = 0 so old LevelData assets default to current 1-cell cars.
/// </summary>
public enum CarSizeType
{
    Medium = 0,
    Small = 1,
    Big = 2,
    Bus = 3
}

public static class CarSizeUtility
{
    public static int GetLength(CarSizeType size)
    {
        switch (size)
        {
            case CarSizeType.Small:
            case CarSizeType.Medium:
                return 1;
            case CarSizeType.Big:
                return 2;
            case CarSizeType.Bus:
                return 3;
            default:
                return 1;
        }
    }

    public static float GetVisualScale(CarSizeType size)
    {
        switch (size)
        {
            case CarSizeType.Small:  return 0.55f;
            case CarSizeType.Medium: return 0.70f;
            case CarSizeType.Big:    return 0.85f;
            case CarSizeType.Bus:    return 1.00f;
            default:                 return 0.70f;
        }
    }

    public static string GetEditorLabel(CarSizeType size)
    {
        switch (size)
        {
            case CarSizeType.Small:  return "S";
            case CarSizeType.Medium: return "M";
            case CarSizeType.Big:    return "B";
            case CarSizeType.Bus:    return "BU";
            default:                 return "?";
        }
    }
}
```

---

## 8. File 2 — LevelData.cs (updated)

**Path:** `Assets/Script/Level/LevelData.cs` — replace entire file.

```csharp
using System.Collections.Generic;
using UnityEngine;

[System.Serializable]
public class CarSpawnInfo
{
    public Vector2Int gridPosition; // HEAD cell (front of car)
    public float rotationY;         // 0=Up, 90=Right, 180=Down, 270=Left (+ diagonals for Small/Medium)
    public CarSizeType carSize = CarSizeType.Medium;
}

[CreateAssetMenu(fileName = "NewLevel", menuName = "ScriptableObjects/LevelData")]
public class LevelData : ScriptableObject
{
    public int levelNumber;

    [Header("Custom Grid Layout")]
    public int gridColumns = 5;
    public int gridRows = 5;

    [Header("Spawn Locations")]
    public List<CarSpawnInfo> carSpawns = new List<CarSpawnInfo>();

    [Header("Pattern Paint (Editor)")]
    [Tooltip("Painted cells used by the editor to auto-generate carSpawns.")]
    public List<Vector2Int> patternCells = new List<Vector2Int>();

    [Header("Fallback (Legacy Level Data)")]
    public int carsToSpawn;

    public bool HasCustomSpawns()
    {
        return carSpawns != null && carSpawns.Count > 0;
    }

    public bool HasPattern()
    {
        return patternCells != null && patternCells.Count > 0;
    }
}
```

---

## 9. File 3 — CarFootprintHelper.cs

**Path:** `Assets/Script/Level/CarFootprintHelper.cs`

```csharp
using System.Collections.Generic;
using UnityEngine;

public static class CarFootprintHelper
{
    /// <summary>
    /// Forward step on the grid from Y rotation (matches SpawnCars 8-way logic).
    /// </summary>
    public static Vector2Int GetForwardStep(float rotationY)
    {
        float angle = Mathf.Repeat(Mathf.Round(rotationY / 45f) * 45f, 360f);
        int rounded = Mathf.RoundToInt(angle);

        switch (rounded)
        {
            case 0:   return new Vector2Int(0, 1);
            case 45:  return new Vector2Int(1, 1);
            case 90:  return new Vector2Int(1, 0);
            case 135: return new Vector2Int(1, -1);
            case 180: return new Vector2Int(0, -1);
            case 225: return new Vector2Int(-1, -1);
            case 270: return new Vector2Int(-1, 0);
            case 315: return new Vector2Int(-1, 1);
            default:  return new Vector2Int(0, 1);
        }
    }

    public static bool IsCardinal(float rotationY)
    {
        float angle = Mathf.Repeat(Mathf.Round(rotationY / 45f) * 45f, 360f);
        int r = Mathf.RoundToInt(angle);
        return r == 0 || r == 90 || r == 180 || r == 270;
    }

    /// <summary>
    /// All cells occupied by a vehicle. First cell = head (gridPosition).
    /// Body extends backward (opposite of forward).
    /// </summary>
    public static List<Vector2Int> GetFootprint(Vector2Int head, float rotationY, CarSizeType size)
    {
        var result = new List<Vector2Int>();
        int length = CarSizeUtility.GetLength(size);

        Vector2Int forward = GetForwardStep(rotationY);
        Vector2Int backward = new Vector2Int(-forward.x, -forward.y);

        Vector2Int current = head;
        for (int i = 0; i < length; i++)
        {
            result.Add(current);
            if (i < length - 1)
                current += backward;
        }

        return result;
    }

    /// <summary>
    /// Rotation (Y) that faces toward the nearest grid border from this cell.
    /// </summary>
    public static float GetOutwardRotation(Vector2Int cell, int columns, int rows)
    {
        int distLeft   = cell.x;
        int distRight  = (columns - 1) - cell.x;
        int distBottom = cell.y;
        int distTop    = (rows - 1) - cell.y;

        int min = Mathf.Min(distLeft, distRight, distBottom, distTop);

        if (min == distLeft)   return 270f; // face left toward x=0
        if (min == distRight)  return 90f;  // face right
        if (min == distBottom) return 180f; // face down toward y=0
        return 0f;                          // face up
    }

    public static bool IsInsideGrid(Vector2Int cell, int columns, int rows)
    {
        return cell.x >= 0 && cell.x < columns && cell.y >= 0 && cell.y < rows;
    }

    public static bool FootprintInsideGrid(Vector2Int head, float rotationY, CarSizeType size, int columns, int rows)
    {
        foreach (var c in GetFootprint(head, rotationY, size))
        {
            if (!IsInsideGrid(c, columns, rows))
                return false;
        }
        return true;
    }
}
```

---

## 10. File 4 — CarMover.cs (small addition)

**Path:** `Assets/Script/CarMover.cs`

Add this field inside the class (near `carType`):

```csharp
[Header("Vehicle Size")]
public CarSizeType carSize = CarSizeType.Medium;
```

No other changes required to `CarMover` — pathfinding still uses grid slots; multi-cell occupation is handled in `Carout` + `SpawnCars`.

---

## 11. File 5 — Carout.cs (multi-cell support)

**Path:** `Assets/Script/Carout.cs`

Replace the entire `CheckForBlockage()` method and add the helper methods below inside the class.

**Add field (near top of class):**

```csharp
public CarSizeType carSize = CarSizeType.Medium;
```

**Add these methods:**

```csharp
public List<Vector2Int> GetOccupiedCells()
{
    return CarFootprintHelper.GetFootprint(currentGridIndex, transform.eulerAngles.y, carSize);
}

public void ReleaseAllOccupiedCells()
{
    if (SpawnCars.Instance == null) return;
    foreach (Vector2Int cell in GetOccupiedCells())
        SpawnCars.Instance.SetSlotOccupation(cell.x, cell.y, false);
}

public void OccupyAllCells()
{
    if (SpawnCars.Instance == null) return;
    foreach (Vector2Int cell in GetOccupiedCells())
        SpawnCars.Instance.SetSlotOccupation(cell.x, cell.y, true);
}
```

**Replace `CheckForBlockage()` with:**

```csharp
public bool CheckForBlockage()
{
    if (SpawnCars.Instance == null) return false;

    var selfCells = new HashSet<Vector2Int>(GetOccupiedCells());

    isBlocked = SpawnCars.Instance.CheckBlockageForCar(
        currentGridIndex,
        transform.eulerAngles.y,
        carSize,
        selfCells,
        out Vector2Int newHeadIndex);

    if (!isBlocked)
    {
        ReleaseAllOccupiedCells();
        currentGridIndex = newHeadIndex;
        OccupyAllCells();
    }
    else
    {
        blockingGridIndex = newHeadIndex;
    }

    return isBlocked;
}
```

`DoTackle()` stays unchanged — it already uses `blockingGridIndex`.

---

## 12. File 6 — SpawnCars.cs (updated spawning)

**Path:** `Assets/Script/SpawnCars.cs`

### 12A — Add serializable entry + list (inside class)

```csharp
[System.Serializable]
public class CarSizePrefabEntry
{
    public CarSizeType size = CarSizeType.Medium;
    public CarMover prefab;
}

[Header("Cars By Size (recommended)")]
public List<CarSizePrefabEntry> carsBySize = new List<CarSizePrefabEntry>();
```

### 12B — Replace `CheckBlockageForCar` signature and body

```csharp
public bool CheckBlockageForCar(
    Vector2Int headIndex,
    Vector3 eulerAngles,
    CarSizeType size,
    HashSet<Vector2Int> selfCells,
    out Vector2Int targetHeadIndex)
{
    float angle = Mathf.Repeat(Mathf.Round(eulerAngles.y / 45f) * 45f, 360f);
    Vector2Int stepDir = CarFootprintHelper.GetForwardStep(angle);

    Vector2Int nextCheck = headIndex + stepDir;
    targetHeadIndex = headIndex;

    while (true)
    {
        // Blocked by another vehicle?
        if (IsSlotBlockedFor(headIndex, nextCheck, selfCells))
        {
            targetHeadIndex = nextCheck;
            return true;
        }

        // Would the full footprint fit at the new head position?
        if (!CarFootprintHelper.FootprintInsideGrid(nextCheck, angle, size, maxColumns, maxRows))
        {
            return false; // clear path to exit
        }

        // All body cells must be free (except self)
        if (!FootprintIsFree(nextCheck, angle, size, selfCells))
        {
            targetHeadIndex = nextCheck;
            return true;
        }

        headIndex = nextCheck;
        targetHeadIndex = headIndex;
        nextCheck = headIndex + stepDir;
    }
}

private bool IsSlotBlockedFor(Vector2Int currentHead, Vector2Int nextCell, HashSet<Vector2Int> selfCells)
{
    if (selfCells != null && selfCells.Contains(nextCell))
        return false;

    return IsSlotBlocked(nextCell.x, nextCell.y);
}

private bool FootprintIsFree(Vector2Int head, float rotationY, CarSizeType size, HashSet<Vector2Int> selfCells)
{
    foreach (Vector2Int cell in CarFootprintHelper.GetFootprint(head, rotationY, size))
    {
        if (selfCells != null && selfCells.Contains(cell))
            continue;

        if (IsSlotBlocked(cell.x, cell.y))
            return false;
    }
    return true;
}
```

### 12C — Add prefab resolver

```csharp
private CarMover ResolvePrefab(CarSizeType size, CarSpawnInfo spawnFallback = null)
{
    for (int i = 0; i < carsBySize.Count; i++)
    {
        if (carsBySize[i].prefab != null && carsBySize[i].size == size)
            return carsBySize[i].prefab;
    }

    // Legacy random list
    if (carPrefabs != null && carPrefabs.Count > 0)
        return carPrefabs[Random.Range(0, carPrefabs.Count)];

    return null;
}
```

### 12D — Add single spawn helper

```csharp
private void SpawnOneCarAt(CarSpawnInfo spawn, GridSlot slot)
{
    CarSizeType size = spawn.carSize;
    float rotation = spawn.rotationY;

    // Big/Bus must use cardinal directions so footprint aligns with grid
    if (size == CarSizeType.Big || size == CarSizeType.Bus)
    {
        if (!CarFootprintHelper.IsCardinal(rotation))
            rotation = CarFootprintHelper.GetOutwardRotation(spawn.gridPosition, maxColumns, maxRows);
    }

    if (!CarFootprintHelper.FootprintInsideGrid(spawn.gridPosition, rotation, size, maxColumns, maxRows))
    {
        Debug.LogWarning($"Spawn skipped — {size} car at {spawn.gridPosition} does not fit. Rotating outward.");
        rotation = CarFootprintHelper.GetOutwardRotation(spawn.gridPosition, maxColumns, maxRows);
    }

    Quaternion carRotation = Quaternion.Euler(0f, rotation, 0f);

    GameObject slotIndicator = Instantiate(carSpawnPrefab, slot.WorldPosition, carRotation);
    spawnedPositions.Add(slotIndicator);

    CarMover chosenPrefab = ResolvePrefab(size, spawn);
    if (chosenPrefab == null) return;

    CarMover activeCar = ObjectPool.Instance.GetCarFromPool(chosenPrefab);

    if (activeCar != null)
    {
        activeCar.transform.position = slot.WorldPosition;
        activeCar.transform.rotation = carRotation;
        activeCar.ResetCapacity();
        activeCar.ResetEnum();
        activeCar.gameObject.SetActive(true);
        activeCar.arrow.enabled = true;
        activeCar.totalPassengerTxt.enabled = false;
    }
    else
    {
        activeCar = Instantiate(chosenPrefab, slot.WorldPosition, carRotation);
    }

    activeCar.carSize = size;
    activeCar.transform.localScale = Vector3.one * CarSizeUtility.GetVisualScale(size);

    Carout caroutComp = activeCar.GetComponent<Carout>();
    if (caroutComp != null)
    {
        caroutComp.carSize = size;
        caroutComp.currentGridIndex = slot.GridIndex;
        caroutComp.OccupyAllCells();
    }

    AnimateCarSpawn(activeCar, spawnPassengers.TotalCarsSpawn.Count);
    spawnPassengers.TotalCarsSpawn.Add(activeCar);
}
```

### 12E — Replace custom-spawn block inside `SpawnGridAndCars()`

Find the block:

```csharp
if (currentLevelData.carSpawns != null && currentLevelData.carSpawns.Count > 0)
```

Replace the `foreach (var spawn in currentLevelData.carSpawns)` body with:

```csharp
foreach (var spawn in currentLevelData.carSpawns)
{
    GridSlot slot = GetSlotAt(spawn.gridPosition.x, spawn.gridPosition.y);
    if (slot == null) continue;

    // Skip if head cell already consumed by a previous multi-cell spawn
    if (slot.IsOccupied) continue;

    SpawnOneCarAt(spawn, slot);
}
```

Remove the old per-spawn duplicate code (random prefab pick, manual Carout assign, etc.) — `SpawnOneCarAt` handles it.

**Legacy fallback branch** (empty `carSpawns`) — leave exactly as-is so levels 1–8 and infinite random generation still work.

---

## 13. File 7 — LevelEditorWindow.cs (new editor)

**Path:** `Assets/Script/Level/Editor/LevelEditorWindow.cs` — replace entire file.

```csharp
#if UNITY_EDITOR
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;

public class LevelEditorWindow : EditorWindow
{
    private enum EditorMode
    {
        PatternPaint,
        ManualSpawn,
        ViewSpawns
    }

    private enum PaintTool
    {
        PaintOn,
        Erase
    }

    private LevelData targetLevel;
    private Vector2Int selectedCell = new Vector2Int(-1, -1);
    private Vector2 scrollPosition;

    private EditorMode mode = EditorMode.PatternPaint;
    private PaintTool paintTool = PaintTool.PaintOn;
    private CarSizeType manualSize = CarSizeType.Medium;

    private bool showValidation = true;

    [MenuItem("Tools/Grid Level Editor")]
    public static void ShowWindow()
    {
        GetWindow<LevelEditorWindow>("Grid Level Editor");
    }

    private void OnGUI()
    {
        GUILayout.Label("Multi-Car Pattern Level Editor", EditorStyles.boldLabel);
        EditorGUILayout.Space();

        EditorGUI.BeginChangeCheck();
        targetLevel = (LevelData)EditorGUILayout.ObjectField("Level Data", targetLevel, typeof(LevelData), false);
        if (EditorGUI.EndChangeCheck())
            selectedCell = new Vector2Int(-1, -1);

        if (targetLevel == null)
        {
            EditorGUILayout.HelpBox("Select or create a LevelData asset.", MessageType.Info);
            if (GUILayout.Button("Create New Level Data", GUILayout.Height(30)))
                CreateNewLevelAsset();
            return;
        }

        DrawHeaderSettings();
        EditorGUILayout.Space();

        mode = (EditorMode)EditorGUILayout.EnumPopup("Editor Mode", mode);
        EditorGUILayout.Space();

        if (mode == EditorMode.PatternPaint)
            DrawPatternTools();
        else if (mode == EditorMode.ManualSpawn)
            DrawManualTools();

        EditorGUILayout.Space();
        DrawGrid();
        EditorGUILayout.Space();

        if (selectedCell.x >= 0)
            DrawSelectionPanel();

        EditorGUILayout.Space();
        DrawActionButtons();
    }

    private void DrawHeaderSettings()
    {
        EditorGUILayout.BeginVertical("box");
        targetLevel.levelNumber = EditorGUILayout.IntField("Level Number", targetLevel.levelNumber);

        EditorGUI.BeginChangeCheck();
        int cols = EditorGUILayout.IntField("Grid Columns", targetLevel.gridColumns);
        int rows = EditorGUILayout.IntField("Grid Rows", targetLevel.gridRows);
        if (EditorGUI.EndChangeCheck())
        {
            targetLevel.gridColumns = Mathf.Clamp(cols, 3, 15);
            targetLevel.gridRows = Mathf.Clamp(rows, 3, 15);
            PruneOutOfBoundsPatternAndSpawns();
            EditorUtility.SetDirty(targetLevel);
        }

        EditorGUILayout.LabelField("Pattern Cells", targetLevel.patternCells.Count.ToString());
        EditorGUILayout.LabelField("Car Spawns", targetLevel.carSpawns.Count.ToString());
        EditorGUILayout.EndVertical();
    }

    private void DrawPatternTools()
    {
        EditorGUILayout.BeginVertical("box");
        GUILayout.Label("Pattern Paint", EditorStyles.boldLabel);
        paintTool = (PaintTool)EditorGUILayout.EnumPopup("Brush", paintTool);
        EditorGUILayout.HelpBox(
            "Paint ON cells where you want cars. Then click 'Generate Cars From Pattern'.\n" +
            "The generator assigns Bus/Big/Small/Medium and faces each car outward.",
            MessageType.Info);
        EditorGUILayout.EndVertical();
    }

    private void DrawManualTools()
    {
        EditorGUILayout.BeginVertical("box");
        GUILayout.Label("Manual Spawn", EditorStyles.boldLabel);
        manualSize = (CarSizeType)EditorGUILayout.EnumPopup("Car Size", manualSize);
        EditorGUILayout.HelpBox("Click a cell to add/remove a car with the selected size.", MessageType.None);
        EditorGUILayout.EndVertical();
    }

    private void DrawGrid()
    {
        GUILayout.Label("Grid (row 0 = bottom)", EditorStyles.boldLabel);
        scrollPosition = EditorGUILayout.BeginScrollView(scrollPosition);

        float buttonSize = 58f;
        HashSet<Vector2Int> patternSet = new HashSet<Vector2Int>(targetLevel.patternCells);

        EditorGUILayout.BeginVertical();
        for (int r = targetLevel.gridRows - 1; r >= 0; r--)
        {
            EditorGUILayout.BeginHorizontal();
            GUILayout.FlexibleSpace();

            for (int c = 0; c < targetLevel.gridColumns; c++)
            {
                Vector2Int coord = new Vector2Int(c, r);
                CarSpawnInfo spawn = FindSpawnAt(coord);
                bool inPattern = patternSet.Contains(coord);

                Color originalBg = GUI.backgroundColor;
                string btnText = ".";

                if (spawn != null)
                {
                    string arrow = GetArrowString(spawn.rotationY);
                    btnText = CarSizeUtility.GetEditorLabel(spawn.carSize) + "\n" + arrow;
                    GUI.backgroundColor = GetSizeColor(spawn.carSize, selectedCell == coord);
                }
                else if (inPattern)
                {
                    btnText = "P";
                    GUI.backgroundColor = selectedCell == coord ? Color.yellow : new Color(0.5f, 0.5f, 0.5f);
                }
                else
                {
                    btnText = $"{c},{r}";
                    GUI.backgroundColor = selectedCell == coord ? Color.yellow : Color.white;
                }

                if (GUILayout.Button(btnText, GUILayout.Width(buttonSize), GUILayout.Height(buttonSize)))
                {
                    selectedCell = coord;
                    HandleCellClick(coord, spawn, inPattern);
                }

                GUI.backgroundColor = originalBg;
            }

            GUILayout.FlexibleSpace();
            EditorGUILayout.EndHorizontal();
        }

        EditorGUILayout.EndVertical();
        EditorGUILayout.EndScrollView();

        if (showValidation)
            DrawValidationMessages();
    }

    private void HandleCellClick(Vector2Int coord, CarSpawnInfo spawn, bool inPattern)
    {
        if (mode == EditorMode.PatternPaint)
        {
            if (paintTool == PaintTool.PaintOn)
            {
                if (!inPattern)
                    targetLevel.patternCells.Add(coord);
            }
            else
            {
                targetLevel.patternCells.Remove(coord);
            }
            EditorUtility.SetDirty(targetLevel);
            return;
        }

        if (mode == EditorMode.ManualSpawn)
        {
            if (spawn == null)
            {
                float rot = CarFootprintHelper.GetOutwardRotation(coord, targetLevel.gridColumns, targetLevel.gridRows);
                targetLevel.carSpawns.Add(new CarSpawnInfo
                {
                    gridPosition = coord,
                    rotationY = rot,
                    carSize = manualSize
                });
            }
            else
            {
                targetLevel.carSpawns.Remove(spawn);
            }
            EditorUtility.SetDirty(targetLevel);
        }
    }

    private void DrawSelectionPanel()
    {
        EditorGUILayout.BeginVertical("box");
        GUILayout.Label($"Selected: ({selectedCell.x}, {selectedCell.y})", EditorStyles.boldLabel);

        CarSpawnInfo spawn = FindSpawnAt(selectedCell);
        if (spawn != null)
        {
            EditorGUI.BeginChangeCheck();
            spawn.carSize = (CarSizeType)EditorGUILayout.EnumPopup("Size", spawn.carSize);
            spawn.rotationY = EditorGUILayout.Slider("Rotation Y", spawn.rotationY, 0f, 360f);
            if (EditorGUI.EndChangeCheck())
                EditorUtility.SetDirty(targetLevel);

            EditorGUILayout.BeginHorizontal();
            if (GUILayout.Button("Face Outward"))
            {
                spawn.rotationY = CarFootprintHelper.GetOutwardRotation(
                    spawn.gridPosition, targetLevel.gridColumns, targetLevel.gridRows);
                EditorUtility.SetDirty(targetLevel);
            }
            if (GUILayout.Button("Delete Car"))
            {
                targetLevel.carSpawns.Remove(spawn);
                EditorUtility.SetDirty(targetLevel);
            }
            EditorGUILayout.EndHorizontal();
        }
        else
        {
            bool inPattern = targetLevel.patternCells.Contains(selectedCell);
            EditorGUILayout.LabelField("Pattern Cell", inPattern ? "YES" : "NO");
        }

        EditorGUILayout.EndVertical();
    }

    private void DrawActionButtons()
    {
        EditorGUILayout.BeginVertical("box");

        GUI.backgroundColor = new Color(0.4f, 0.75f, 1f);
        if (GUILayout.Button("Generate Cars From Pattern", GUILayout.Height(32)))
        {
            PatternToCarGenerator.Generate(targetLevel);
            EditorUtility.SetDirty(targetLevel);
        }

        if (GUILayout.Button("Auto Face All Outward", GUILayout.Height(28)))
        {
            foreach (var s in targetLevel.carSpawns)
            {
                s.rotationY = CarFootprintHelper.GetOutwardRotation(
                    s.gridPosition, targetLevel.gridColumns, targetLevel.gridRows);
            }
            EditorUtility.SetDirty(targetLevel);
        }

        if (GUILayout.Button("Clear All Spawns", GUILayout.Height(24)))
        {
            targetLevel.carSpawns.Clear();
            EditorUtility.SetDirty(targetLevel);
        }

        if (GUILayout.Button("Clear Pattern", GUILayout.Height(24)))
        {
            targetLevel.patternCells.Clear();
            EditorUtility.SetDirty(targetLevel);
        }

        GUI.backgroundColor = new Color(0.4f, 0.85f, 0.45f);
        if (GUILayout.Button("Save Level Asset", GUILayout.Height(34)))
        {
            EditorUtility.SetDirty(targetLevel);
            AssetDatabase.SaveAssets();
            Debug.Log($"Saved {targetLevel.name}");
        }

        GUI.backgroundColor = Color.white;
        showValidation = EditorGUILayout.Toggle("Show Validation", showValidation);
        EditorGUILayout.EndVertical();
    }

    private void DrawValidationMessages()
    {
        var issues = LevelValidator.Validate(targetLevel);
        if (issues.Count == 0) return;

        EditorGUILayout.HelpBox(string.Join("\n", issues), MessageType.Warning);
    }

    private CarSpawnInfo FindSpawnAt(Vector2Int coord)
    {
        // Head cell match
        return targetLevel.carSpawns.Find(s => s.gridPosition == coord);
    }

    private Color GetSizeColor(CarSizeType size, bool selected)
    {
        Color c;
        switch (size)
        {
            case CarSizeType.Small:  c = new Color(0.3f, 0.85f, 0.45f); break;
            case CarSizeType.Big:    c = new Color(1f, 0.6f, 0.2f); break;
            case CarSizeType.Bus:    c = new Color(0.65f, 0.35f, 0.95f); break;
            default:                 c = new Color(0.3f, 0.6f, 0.95f); break;
        }
        if (selected) c = Color.Lerp(c, Color.white, 0.35f);
        return c;
    }

    private string GetArrowString(float rotationY)
    {
        float a = Mathf.Repeat(Mathf.Round(rotationY / 45f) * 45f, 360f);
        if (Mathf.Approximately(a, 0f)) return "↑";
        if (Mathf.Approximately(a, 90f)) return "→";
        if (Mathf.Approximately(a, 180f)) return "↓";
        if (Mathf.Approximately(a, 270f)) return "←";
        if (Mathf.Approximately(a, 45f)) return "↗";
        if (Mathf.Approximately(a, 135f)) return "↘";
        if (Mathf.Approximately(a, 225f)) return "↙";
        if (Mathf.Approximately(a, 315f)) return "↖";
        return a + "°";
    }

    private void PruneOutOfBoundsPatternAndSpawns()
    {
        targetLevel.patternCells.RemoveAll(p =>
            p.x < 0 || p.x >= targetLevel.gridColumns || p.y < 0 || p.y >= targetLevel.gridRows);

        targetLevel.carSpawns.RemoveAll(s =>
            s.gridPosition.x < 0 || s.gridPosition.x >= targetLevel.gridColumns ||
            s.gridPosition.y < 0 || s.gridPosition.y >= targetLevel.gridRows);
    }

    private void CreateNewLevelAsset()
    {
        string path = EditorUtility.SaveFilePanelInProject(
            "Create Level Data", "NewLevel", "asset", "Choose save location");

        if (string.IsNullOrEmpty(path)) return;

        LevelData newLevel = ScriptableObject.CreateInstance<LevelData>();
        AssetDatabase.CreateAsset(newLevel, path);
        AssetDatabase.SaveAssets();
        targetLevel = newLevel;
        selectedCell = new Vector2Int(-1, -1);
    }
}

/// <summary>
/// Editor-only: converts painted pattern into sized car spawns.
/// </summary>
public static class PatternToCarGenerator
{
    public static void Generate(LevelData level)
    {
        level.carSpawns.Clear();

        if (level.patternCells == null || level.patternCells.Count == 0)
        {
            Debug.LogWarning("Pattern is empty — paint cells first.");
            return;
        }

        bool[,] grid = new bool[level.gridColumns, level.gridRows];
        bool[,] used = new bool[level.gridColumns, level.gridRows];

        foreach (var p in level.patternCells)
        {
            if (CarFootprintHelper.IsInsideGrid(p, level.gridColumns, level.gridRows))
                grid[p.x, p.y] = true;
        }

        // 1) Try to place BUS along horizontal and vertical runs of length >= 3
        PlaceRuns(level, grid, used, 3, CarSizeType.Bus, true);
        PlaceRuns(level, grid, used, 3, CarSizeType.Bus, false);

        // 2) BIG runs of length 2
        PlaceRuns(level, grid, used, 2, CarSizeType.Big, true);
        PlaceRuns(level, grid, used, 2, CarSizeType.Big, false);

        // 3) Remaining isolated cells
        for (int x = 0; x < level.gridColumns; x++)
        {
            for (int y = 0; y < level.gridRows; y++)
            {
                if (!grid[x, y] || used[x, y]) continue;

                Vector2Int head = new Vector2Int(x, y);
                float rot = CarFootprintHelper.GetOutwardRotation(head, level.gridColumns, level.gridRows);

                // Border cells → Medium, deep interior single cells → Small
                CarSizeType size = IsBorderCell(head, level) ? CarSizeType.Medium : CarSizeType.Small;

                level.carSpawns.Add(new CarSpawnInfo
                {
                    gridPosition = head,
                    rotationY = rot,
                    carSize = size
                });
                used[x, y] = true;
            }
        }

        ResolveOverlaps(level);
        Debug.Log($"Generated {level.carSpawns.Count} cars from pattern.");
    }

    private static void PlaceRuns(LevelData level, bool[,] grid, bool[,] used, int length, CarSizeType size, bool horizontal)
    {
        if (horizontal)
        {
            for (int y = 0; y < level.gridRows; y++)
            {
                for (int x = 0; x <= level.gridColumns - length; x++)
                {
                    if (!IsRunFree(grid, used, x, y, length, true)) continue;

                    // Head at the end closest to right border (faces outward easier)
                    Vector2Int head = new Vector2Int(x + length - 1, y);
                    float rot = 90f; // face right; Auto Face can refine

                    if (TryAddSpawn(level, head, rot, size, used))
                        MarkRun(used, x, y, length, true);
                }
            }
        }
        else
        {
            for (int x = 0; x < level.gridColumns; x++)
            {
                for (int y = 0; y <= level.gridRows - length; y++)
                {
                    if (!IsRunFree(grid, used, x, y, length, false)) continue;

                    Vector2Int head = new Vector2Int(x, y + length - 1);
                    float rot = 0f; // face up

                    if (TryAddSpawn(level, head, rot, size, used))
                        MarkRun(used, x, y, length, false);
                }
            }
        }
    }

    private static bool TryAddSpawn(LevelData level, Vector2Int head, float rot, CarSizeType size, bool[,] used)
    {
        if (!CarFootprintHelper.FootprintInsideGrid(head, rot, size, level.gridColumns, level.gridRows))
            return false;

        rot = CarFootprintHelper.GetOutwardRotation(head, level.gridColumns, level.gridRows);

        // Force cardinal for multi-cell
        if (size == CarSizeType.Big || size == CarSizeType.Bus)
        {
            if (!CarFootprintHelper.IsCardinal(rot))
                rot = SnapToBestCardinal(head, level);
        }

        if (!CarFootprintHelper.FootprintInsideGrid(head, rot, size, level.gridColumns, level.gridRows))
            return false;

        level.carSpawns.Add(new CarSpawnInfo { gridPosition = head, rotationY = rot, carSize = size });

        foreach (var cell in CarFootprintHelper.GetFootprint(head, rot, size))
            used[cell.x, cell.y] = true;

        return true;
    }

    private static float SnapToBestCardinal(Vector2Int head, LevelData level)
    {
        float[] cardinals = { 0f, 90f, 180f, 270f };
        float best = 0f;
        int bestDist = int.MaxValue;

        foreach (float rot in cardinals)
        {
            Vector2Int step = CarFootprintHelper.GetForwardStep(rot);
            int dist = DistanceToBorderAlong(head, step, level.gridColumns, level.gridRows);
            if (dist < bestDist)
            {
                bestDist = dist;
                best = rot;
            }
        }
        return best;
    }

    private static int DistanceToBorderAlong(Vector2Int head, Vector2Int forward, int cols, int rows)
    {
        if (forward.x > 0) return (cols - 1) - head.x;
        if (forward.x < 0) return head.x;
        if (forward.y > 0) return (rows - 1) - head.y;
        if (forward.y < 0) return head.y;
        return 999;
    }

    private static bool IsRunFree(bool[,] grid, bool[,] used, int x, int y, int length, bool horizontal)
    {
        for (int i = 0; i < length; i++)
        {
            int cx = horizontal ? x + i : x;
            int cy = horizontal ? y : y + i;
            if (!grid[cx, cy] || used[cx, cy]) return false;
        }
        return true;
    }

    private static void MarkRun(bool[,] used, int x, int y, int length, bool horizontal)
    {
        for (int i = 0; i < length; i++)
        {
            int cx = horizontal ? x + i : x;
            int cy = horizontal ? y : y + i;
            used[cx, cy] = true;
        }
    }

    private static bool IsBorderCell(Vector2Int cell, LevelData level)
    {
        return cell.x == 0 || cell.y == 0 ||
               cell.x == level.gridColumns - 1 || cell.y == level.gridRows - 1;
    }

    private static void ResolveOverlaps(LevelData level)
    {
        // Final pass: auto face all outward
        foreach (var s in level.carSpawns)
        {
            s.rotationY = CarFootprintHelper.GetOutwardRotation(
                s.gridPosition, level.gridColumns, level.gridRows);
        }
    }
}

public static class LevelValidator
{
    public static List<string> Validate(LevelData level)
    {
        var issues = new List<string>();
        bool[,] occupied = new bool[level.gridColumns, level.gridRows];

        foreach (var spawn in level.carSpawns)
        {
            if (spawn.carSize == CarSizeType.Big || spawn.carSize == CarSizeType.Bus)
            {
                if (!CarFootprintHelper.IsCardinal(spawn.rotationY))
                    issues.Add($"{spawn.carSize} at {spawn.gridPosition} should use 0/90/180/270 rotation.");
            }

            foreach (var cell in CarFootprintHelper.GetFootprint(spawn.gridPosition, spawn.rotationY, spawn.carSize))
            {
                if (!CarFootprintHelper.IsInsideGrid(cell, level.gridColumns, level.gridRows))
                {
                    issues.Add($"{spawn.carSize} at {spawn.gridPosition} extends outside grid.");
                    continue;
                }

                if (occupied[cell.x, cell.y])
                    issues.Add($"Overlap at ({cell.x},{cell.y}) — two cars share a cell.");

                occupied[cell.x, cell.y] = true;
            }
        }

        return issues;
    }
}
#endif
```

---

## 14. Unity Inspector Setup

### SpawnCarsManager (in SampleScene)

1. Expand **Cars By Size** → set size 4:
   - `Medium` → RedCar prefab
   - `Small` → GreenCar prefab
   - `Big` → BigCar prefab *(create)*
   - `Bus` → BusCar prefab *(create)*

2. Keep existing **Car Prefabs** list as fallback.

3. Confirm `centerArea` and `maxColumns/maxRows` match your scene grid (currently 7×7 in scene).

### LevelManager

No changes — still assigns `LevelData` from `allLevels` list.

### Open editor

**Tools → Grid Level Editor**

---

## 15. Workflow: Create a Level From a Pattern

1. Create or select a `LevelData` asset in `Assets/Script/Level/CustomeLevels/`.
2. Set grid size (e.g. 7×7).
3. Mode = **Pattern Paint** → brush **Paint On**.
4. Click cells to draw your shape (cross, square, arrow, etc.).
5. Click **Generate Cars From Pattern**.
6. Review grid colors:
   - **BU** = Bus, **B** = Big, **S** = Small, **M** = Medium
7. Click **Auto Face All Outward** if you moved anything manually.
8. Check validation warnings (overlaps / out of bounds).
9. **Save Level Asset**.
10. Add asset to `LevelManager.allLevels` list in scene.
11. Press Play — cars spawn with correct size prefabs; clicking uses existing `CarController` → `Carout.CheckForBlockage()` flow.

### Example pattern (7×7 plus shape)

Paint cells:
```
. . . . . . .
. . . P . . .
. . P P P . .
. . . P . . .
. . . . . . .
```

Generator creates mostly **Medium** on border arms and **Small** in center — adjust manually if desired.

---

## 16. Outward Facing & Face-to-Face Blocking

### Auto outward

`CarFootprintHelper.GetOutwardRotation()` picks the nearest border:
- Left column → face **270°**
- Right column → **90°**
- Bottom row → **180°**
- Top row → **0°**

Interior cars face the closest edge even if another car blocks the path — that is intentional for jam puzzles.

### Face-to-face = blocked (already your game rules)

```
Car A (head) → → ← ← Car B (head)
```

When player taps Car A:
1. `CarController.HandleCarSelection()`
2. `Carout.CheckForBlockage()` → `SpawnCars.CheckBlockageForCar()` sees occupied cell → `isBlocked = true`
3. `DoTackle()` animation plays — car does **not** move

No changes needed in `CarController` or `ParkingGameManager`.

---

## 17. Backward Compatibility Checklist

| Scenario | Result |
|----------|--------|
| Old `LevelData` without `carSize` | Defaults to `Medium` (enum value 0) |
| Old `LevelData` without `patternCells` | Empty list — pattern tools optional |
| Levels with only `carsToSpawn` | Fallback branch in `SpawnGridAndCars()` unchanged |
| Existing `CustomeLevels/*.asset` files | Load fine; all cars treated as Medium until re-saved |
| `CarMover` / `Carout` on prefabs | Still required; only new `carSize` field added |

---

## 18. Testing Checklist

- [ ] Open **Tools → Grid Level Editor** — no compile errors
- [ ] Paint pattern on new level → Generate → spawns appear with sizes
- [ ] Save asset → reload editor — data persists
- [ ] Play Mode: level loads, cars visible at correct scales
- [ ] Tap unblocked border car → moves to parking (existing flow)
- [ ] Tap car blocked by another → tackle animation, no move
- [ ] Two cars face each other → both blocked when tapped toward each other
- [ ] Bus occupies 3 cells — another car cannot spawn overlapping
- [ ] Level 1–8 legacy (`carsToSpawn` only) still generates random grid
- [ ] Restart level / complete level — pool recycle works
- [ ] Passengers spawn matching car colors/capacities

---

## 19. Troubleshooting

| Problem | Fix |
|---------|-----|
| All cars same size at runtime | Fill **Cars By Size** on SpawnCars; check `carSize` on spawn info in asset |
| Bus clips through grid | Ensure Bus uses **cardinal** rotation; run validator |
| Overlap warnings in editor | Regenerate pattern or delete conflicting spawns manually |
| Car moves through another car | Confirm all footprint cells marked occupied in `OccupyAllCells()` |
| Random prefab still used | `carsBySize` entry missing for that size — falls back to `carPrefabs` |
| Compile error `CarSizeType not found` | Add `CarSizeType.cs` first |
| Old levels behave differently | Expected if you re-save with generator; backup assets first |

---

## Quick Reference — Files Touched

| File | Action |
|------|--------|
| `Assets/Script/Level/CarSizeType.cs` | **NEW** |
| `Assets/Script/Level/CarFootprintHelper.cs` | **NEW** |
| `Assets/Script/Level/LevelData.cs` | **UPDATE** |
| `Assets/Script/CarMover.cs` | **ADD** `carSize` field |
| `Assets/Script/Carout.cs` | **UPDATE** blockage + occupy |
| `Assets/Script/SpawnCars.cs` | **UPDATE** spawn + blockage |
| `Assets/Script/Level/Editor/LevelEditorWindow.cs` | **REPLACE** |
| Car prefabs | **SET** `carSize` + create Big/Bus |
| `SpawnCarsManager` in scene | **SET** carsBySize list |

---

*End of guide. Apply sections in order, then test in Play Mode. Your existing `CarMover` movement, parking, passengers, tutorials, and power-ups remain untouched.*
